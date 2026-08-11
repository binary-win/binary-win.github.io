# The Growing Importance of Windows Credential Theft

**Tags:** Memory Forensics · Volatility 3 · WinPmem · Windows Internals

---

## Overview

We acquire a full physical memory image from a live Windows 11 system using [WinPmem](https://github.com/Velocidex/WinPmem) — a legitimate, open-source forensic memory acquisition tool developed by Velocidex — and then analyze the image with [Volatility 3](https://github.com/volatilityfoundation/volatility3).

The initial objective is straightforward: determine whether NTLM credential material can be recovered from a complete physical memory image.

However, this is not simply a step-by-step extraction guide. It documents a real failure, investigates the underlying cause, and demonstrates how the limitation can be addressed by combining offline memory forensics with information obtained from the live system.

The investigation is particularly interesting because the memory image contains the expected Windows registry hives, yet Volatility is unable to reconstruct the required credential material.

---

## 1. Why Windows Credential Material Matters

Credential theft remains an important objective during post-compromise activity on Windows systems. Authentication material can potentially allow an attacker to impersonate users, access additional resources, and move laterally through an environment.

Recent threat reporting has continued to highlight credential theft as an important component of modern ransomware and intrusion operations. The [Interlock credential-theft report](https://gbhackers.com/interlock-credential-theft/) provides a useful example of this broader trend.

From a defensive and forensic perspective, this raises an important question:

> **If authentication material was present on a Windows system during an incident, can it be recovered from a physical memory image acquired from that system?**

Windows authentication involves multiple components and credential stores. Depending on the authentication mechanism and operating-system configuration, credential-related material can exist in process memory, registry-backed structures, security subsystems, and other locations.

Microsoft documents the role of the Local Security Authority and related components in Windows authentication:

* [Microsoft Learn — Credentials Processes in Windows Authentication](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/credentials-processes-in-windows-authentication)

The [MITRE ATT&CK framework](https://attack.mitre.org/techniques/T1555/004/) also documents credential access techniques involving Windows Credential Manager and other sources of authentication material.

This makes physical memory an interesting forensic artifact. A memory image can preserve information that is not necessarily present in ordinary files on disk, including process state, kernel structures, registry data, and remnants of security-sensitive information.

At the same time, a memory image is not a perfect representation of everything that existed in RAM during the lifetime of the operating system. Windows uses virtual memory and demand paging, meaning that pages can be moved between physical memory and the pagefile.

That distinction becomes critical later in this investigation.

---

## 2. Acquiring Memory with WinPmem

[WinPmem](https://github.com/Velocidex/WinPmem) is a multi-platform memory acquisition tool developed by Velocidex and commonly used in legitimate digital-forensics investigations.

For this investigation, a complete physical memory image is acquired from a live Windows 10 system.

Run the acquisition tool from an elevated PowerShell session:

```powershell
.\go-winpmem_amd64_1.0-rc2_signed.exe acquire memdump.raw
```

A typical acquisition looks similar to:

```text
Writing driver to C:\Users\...\AppData\Local\Temp\875924924.sys
Creating service winpmem
Started service winpmem

Memory Info:
  CR3: 0x1ad000
  NtBuildNumber: 0x4a65        ← 19045 = Windows 10 22H2
  KernelBase: 0xfffff80640a03000

  KPCR:
  - 0xfffff8063f6f3000
  - 0xffffac80b53e5000

...

Completed imaging in 2.0621657s
Stopped service winpmem
```

The resulting file is a raw physical-memory image.

WinPmem can represent physical memory as a sparse image. In other words, physical ranges that are not backed by captured memory pages may appear as gaps rather than as a contiguous representation of every physical address.

This distinction becomes important during offline registry analysis.

> **Note:** The exact output, driver path, addresses, and acquisition time will vary between systems.

---

## 3. Setting Up Volatility 3

Clone the [Volatility 3](https://github.com/volatilityfoundation/volatility3) repository:

```powershell
git clone https://github.com/volatilityfoundation/volatility3.git
cd volatility3
```

Install the required cryptographic dependency:

```powershell
pip install pycryptodome
```

> **Important:** Do not install the unrelated `Crypto` package from PyPI. `pycryptodome` provides the `Crypto` namespace used by Volatility components that require cryptographic primitives such as AES and ARC4.

Verify the installation:

```powershell
python -c "from Crypto.Cipher import AES; print('OK')"
```

Expected output:

```text
OK
```

---

## 4. Extracting Hashes — First Attempt

With the memory image available, the first step is to let Volatility attempt the extraction normally:

```powershell
python vol.py -f memdump.raw windows.registry.hashdump
```

The result is:

```text
User    rid     lmhash  nthash
WARNING volatility3.plugins.windows.registry.hashdump: Hbootkey is not valid
```

No hashes are returned.

At first glance, this is surprising. The registry hives appear to exist in the memory image, so why can't Volatility derive the required key material?

To answer that, we need to understand what the `hashdump` plugin actually requires.

---

## 5. What Volatility Needs

The Windows registry stores the information required to derive the keys used to protect SAM password hashes.

At a high level, the extraction process involves several stages:

1. Locate the SYSTEM hive.
2. Recover the Windows boot key from the LSA registry structure.
3. Use the boot key to derive the key material required for the SAM database.
4. Locate the SAM hive.
5. Decrypt the relevant SAM structures.
6. Recover the stored NTLM hash material.

The boot key is assembled from the Class data associated with four LSA subkeys:

```text
SYSTEM
└── ControlSet001
    └── Control
        └── Lsa
            ├── JD
            ├── Skew1
            ├── GBG
            └── Data
```

These four pieces are combined and transformed according to the Windows boot-key derivation process.

The SAM hive contains the corresponding encrypted structures under:

```text
SAM
└── SAM
    └── Domains
        └── Account
```

Therefore, simply having the SYSTEM and SAM hive structures present in memory is not necessarily sufficient.

---

## 6. Confirming the Hives Exist

First, enumerate the registry hives available in the memory image:

```powershell
python vol.py -f memdump.raw windows.registry.hivelist
```

Example:

```text
0xdd0950658000  \REGISTRY\MACHINE\SYSTEM
0xdd0951d3d000  \SystemRoot\System32\Config\SAM
```

This tells us something important:

* The SYSTEM hive exists.
* The SAM hive exists.
* Volatility can locate both hives.

Therefore, the failure is not simply caused by a missing registry hive.

The next question is whether the specific registry data required for boot-key derivation is actually available in the physical memory image.

---

## 7. Root Cause — Missing or Paged-Out Registry Data

The boot-key components are associated with the **Class data** of the four LSA subkeys.

We can inspect one of the keys directly:

```powershell
python vol.py -f memdump.raw windows.registry.printkey `
  --offset 0xdd0950658000 `
  --key "ControlSet001\Control\Lsa\JD"
```

The output may look like:

```text
Key  \REGISTRY\MACHINE\SYSTEM\ControlSet001\Control\Lsa\JD  -  -  -
```

The relevant Class information is unavailable.

The same situation can occur for:

```text
JD
Skew1
GBG
Data
```

This is the critical observation.

### Why can this happen?

A physical memory image captures the pages that are physically resident at the moment of acquisition. It does not guarantee that every virtual-memory page that has existed during the lifetime of the operating system is currently resident in RAM.

Windows uses demand paging and virtual memory management. Registry hive data can be backed by memory-mapped files, and individual pages may not be resident when the acquisition occurs.

Consequently:

```text
Registry hive exists
        ↓
Registry key exists
        ↓
Required Class data may not be resident
        ↓
Physical acquisition does not contain that page
        ↓
Volatility cannot reconstruct the boot key
        ↓
hashdump fails
```

This distinction is fundamental to memory forensics:

> **A memory image containing a registry hive does not necessarily contain every piece of registry data that existed logically in that hive.**

This is not necessarily a WinPmem bug. It is a consequence of acquiring a snapshot of physical memory from a running virtual-memory system.

---

## 8. The Practical Implication

At this point, there are two separate data sources:

### Offline evidence

The memory image provides access to artifacts such as:

* Processes
* Threads
* Kernel structures
* Network connections
* Handles
* Loaded modules
* Registry structures
* Injected or resident code
* Other volatile artifacts

### Live-system state

The running operating system may still have access to registry information that is not recoverable from the physical memory image.

This creates an important forensic distinction:

> **The logical state of the registry and the physical state of RAM are not necessarily identical.**

If the objective is specifically to investigate credential material, a memory-only workflow may therefore be insufficient in some situations.

For a controlled lab or forensic investigation where live-system access is explicitly authorized, the live registry can be examined as a separate evidence source.

---

## 9. Combining Live Registry Information with Offline Analysis

For this lab, the workaround is to obtain the required key material from the live Windows registry and provide that information to the offline Volatility analysis.

Conceptually, the workflow becomes:

```text
             Live Windows System
                    │
          ┌─────────┴─────────┐
          │                   │
      SYSTEM hive          SAM hive
          │                   │
       boot key          required key data
          │                   │
          └─────────┬─────────┘
                    │
                    ▼
             Volatility 3
                    │
                    ▼
          Offline memory image
                    │
                    ▼
           Registry analysis
```

This approach avoids assuming that every registry page required by the plugin must be present in the physical memory image.

In other words, instead of modifying the memory image itself, the missing prerequisite information is supplied to the analysis layer.

---

## 10. Why the Workaround Works

The original failure occurs before Volatility can process the individual SAM records.

The dependency chain is approximately:

```text
SYSTEM hive
    │
    ├── LSA Class data
    │
    ▼
Boot key
    │
    ▼
SAM key material
    │
    ▼
SAM records
    │
    ▼
NTLM hash material
```

If the boot-key material cannot be recovered, the subsequent stages cannot proceed.

The workaround supplies the missing prerequisite from an authorized live-system source while leaving the memory image itself unchanged.

This also demonstrates an important principle of forensic tooling:

> **A plugin failure does not necessarily mean that the evidence is absent. It may mean that one of the prerequisites required by the extraction algorithm is unavailable from the particular evidence source.**

---

## 11. Validation

After supplying the missing key material to the analysis process, the registry structures contained in the memory image can be processed again.

The important distinction is that the memory image itself has not changed.

Instead:

```text
Original memory image
        +
Required key material
        ↓
Volatility analysis
        ↓
Registry records become processable
```

This makes it possible to separate two questions that are often conflated:

1. **Does the memory image contain the required registry structures?**
2. **Does the forensic plugin have everything it needs to interpret and decrypt those structures?**

In this case, the answer to the first question is partially yes, while the second is initially no.

---

## 12. Lessons from the Failure

This investigation highlights several important characteristics of Windows memory forensics.

### 12.1 A memory image is a snapshot

Physical memory acquisition captures the state of RAM at a particular point in time. It should not be treated as a complete historical copy of everything that has existed in the operating system's virtual address space.

### 12.2 Registry hives are not enough

Finding the SYSTEM and SAM hives does not automatically mean that every structure required for credential analysis is available.

### 12.3 Paging matters

Pages that are not resident at acquisition time may not be recoverable from the physical memory image alone.

### 12.4 Tool failures can reveal evidence limitations

An error such as:

```text
Hbootkey is not valid
```

does not necessarily mean that the SAM hive is corrupt or that the credentials never existed.

It can instead indicate that an upstream dependency is unavailable.

### 12.5 Offline and live analysis answer different questions

Offline memory analysis preserves the acquired evidence and is highly valuable for incident response.

Live analysis, however, may expose operating-system state that was not physically resident when the memory image was captured.

---

## 13. Final Results

| Step                                         | Result                            |
| -------------------------------------------- | --------------------------------- |
| Memory acquisition with WinPmem              | ✅ Successful                      |
| SYSTEM hive located                          | ✅ Successful                      |
| SAM hive located                             | ✅ Successful                      |
| Required LSA Class data available in image   | ❌ Not available                   |
| Standard Volatility hashdump                 | ❌ Failed                          |
| Root cause identified                        | ✅ Missing/paged-out registry data |
| Live-system registry analysis                | ✅ Successful in the lab           |
| Offline analysis with supplied prerequisites | ✅ Successful                      |

The key takeaway is that memory forensics is not simply a matter of obtaining a large `.raw` file and running every available plugin against it.

The quality of the final result depends on understanding how Windows manages memory, how registry hives are represented in memory, what information a particular forensic plugin requires, and which portions of that information were actually resident at acquisition time.

---

## 14. Conclusion

The initial assumption was simple:

> **If the SYSTEM and SAM hives are present in a memory image, Volatility should be able to extract the NTLM hashes.**

The investigation demonstrates why that assumption is incomplete.

Volatility successfully located the relevant registry hives, but the information required to derive the boot key was not available through the captured physical pages. The failure therefore occurred not because the hives were absent, but because an important dependency of the extraction process was unavailable in the acquired memory snapshot.

This is a useful example of why Windows internals matter in memory forensics. Understanding the distinction between logical registry state, virtual memory, and resident physical pages makes it possible to interpret forensic-tool failures correctly rather than treating them as simple extraction errors.

For defenders, the same observation has an important implication: **a single memory snapshot should not be considered a complete representation of all credential-related state on a Windows system.**

For forensic investigators, it reinforces the value of combining multiple evidence sources — memory images, registry hives, pagefiles, disk artifacts, and appropriately collected live-system data — when the investigation requires reconstruction of volatile authentication state.

Ultimately, the interesting part of this investigation was not that the first command failed.

It was understanding **why** it failed.

---

## References

1. **Velocidex — WinPmem**
   https://github.com/Velocidex/WinPmem

2. **Volatility Foundation — Volatility 3**
   https://github.com/volatilityfoundation/volatility3

3. **Microsoft Learn — Credentials Processes in Windows Authentication**
   https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/credentials-processes-in-windows-authentication

4. **MITRE ATT&CK — Credentials from Web Browsers / Credential Stores**
   https://attack.mitre.org/techniques/T1555/

5. **Moyix — Syskey and SAM**
   https://moyix.blogspot.com/2008/02/syskey-and-sam.html

6. **Passcape — Bootkey and Syskey**
   https://www.passcape.com/index.php?section=docsys&cmd=details&id=23

7. **GBHackers — Interlock Credential Theft**
   https://gbhackers.com/interlock-credential-theft/
