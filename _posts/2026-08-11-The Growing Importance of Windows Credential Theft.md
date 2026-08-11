# The Growing Importance of Windows Credential Theft

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/646eeef0-1423-4677-baee-3ce4c10439c5" />

**Tags:** Memory Forensics · Volatility 3 · WinPmem · ntlm hash

---

## Overview

We acquire a full physical memory image from a live Windows 11 system using [WinPmem](https://github.com/Velocidex/WinPmem) — a legitimate, open-source forensic memory acquisition tool developed by Velocidex and then analyze the image with [Volatility 3](https://github.com/volatilityfoundation/volatility3).

The initial objective is straightforward: determine whether NTLM credential material can be recovered from a complete physical memory image.

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

---

## 2. Acquiring Memory with WinPmem

[WinPmem](https://github.com/Velocidex/WinPmem) is a multi-platform physical memory acquisition tool developed by Velocidex and widely used in digital-forensics and incident-response investigations. On Windows, the standalone executable is self-contained and automatically loads the appropriate signed kernel driver required to access physical memory. Once the acquisition is complete, WinPmem automatically unloads the driver, leaving no persistent driver service running from the acquisition process.

For this investigation, a complete physical memory image is acquired from a live Windows 11 system.

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
  NtBuildNumber: ....
  KernelBase: ....

  KPCR:
  - 0xfffff8063f6f3000
  - 0xffffac80b53e5000

...

Completed imaging in 20.0621657s
Stopped service winpmem
```

The resulting file is a raw physical-memory image.

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
└── ControlSet
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

This is essentially the same underlying process used by tools such as secretsdump and other SAM-dumping utilities: obtain the required key material from the SYSTEM hive, use it to access the protected SAM data, and then process the account records to recover the stored NTLM hash material.

The main difference is the source of the data. Tools such as secretsdump can operate directly on registry hives, while Volatility performs the same type of analysis against registry structures reconstructed from a physical memory image.

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



With the required registry data available, the NTLM hash can be successfully recovered, as demonstrated in the following image:
<img width="775" height="179" alt="image" src="https://github.com/user-attachments/assets/1a5bd157-734c-4e51-a3b0-c9b96e3ec60b" />



## 14. Conclusion

From a Red Team perspective, this experiment demonstrated that memory acquisition can be an effective technique for credential access after compromising a Windows system. The successful analysis of the SYSTEM and SAM hives with WinPmem and Volatility 3 showed that sensitive authentication artifacts can be recovered from a live system's memory. This highlights the importance of memory analysis as part of post-exploitation activities and demonstrates how it can be used to evaluate the real impact of credential exposure on a compromised endpoint.


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

6. **GBHackers — Interlock Credential Theft**
   https://gbhackers.com/interlock-credential-theft/
