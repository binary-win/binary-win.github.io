# Analyzing Avast AV: Kernel Hooking and Driver Reverse Engineering

##### **Summary**
We walk through undocumented internals such as CKCL, PerfInfoLogSysCallEntry, HalPrivateDispatch / HalpPerformanceCounter, and explain how ETW can be abused to gain early control over syscall dispatch.
also a little reversing avast driver 

Real-World Example that we check : Avast Kernel Hooks
Avast driver (`aswVmm.sys`) hook syscalls using this technique Intercepts sensitive operations like etc:
`NtTerminateProcess`
`NtOpenProcess`

<img width="866" height="277" alt="image" src="https://github.com/user-attachments/assets/fc917027-b060-4098-ad3c-9e9b06cd89f3" />



The following commands are useful for inspecting the relevant structures and execution flow, and you can try them yourself:

`bcdedit /debug on`

`bcdedit /dbgsettings NET HOSTIP:192.168.113.121 PORT:50000`
>Enables kernel debugging and configures remote debugging via WinDbg.

`logman query providers`
>Lists ETW providers, including CKCL.

`ba w 8 ffff8283b9d81ab0`
>Sets a write breakpoint; the 0x40 offset is significant and corresponds to @rsp + 0x40.

`dt _WMI_LOGGER_CONTEXT poi(poi(nt!EtwpDebuggerData + 0x10) + 0x10)`
>Displays the _WMI_LOGGER_CONTEXT structure to inspect the GetCpuLock field.

`dq poi(nt!HalpPerformanceCounter) + 0x70`
>Dumps the pointer at offset 0x70 to identify the address that has been replaced.

---

### Brief History on Antivirus and EDR kernel hooking
Kernel hooking has been a cornerstone technique in endpoint security since the early days of antivirus software, allowing products to intercept system calls, monitor processes, and detect malicious behavior at a low level. However, it has also been controversial due to stability issues, conflicts with OS protections, and similarities to rootkit techniques used by malware.

Traditional antivirus software primarily relied on user-mode scanning and signature-based detection. As threats evolved (e.g., rootkits, stealthy malware), vendors needed deeper visibility. They began loading kernel-mode drivers to hook critical structures like the System Service Descriptor Table (SSDT) or SYSENTER instructions. This allowed interception of system calls (e.g., file access, process creation) to scan for malware in real time.

Common techniques:
SSDT hooking: Replacing pointers in the system call table to redirect calls to AV code.
Inline hooking or patching kernel functions.

### Microsoft Introduces Kernel Patch Protection (PatchGuard)
Microsoft addressed stability and security issues by introducing Kernel Patch Protection (KPP), also known as PatchGuard, starting with 64-bit editions of Windows XP and Server 2003 SP1 (fully enforced in Windows Vista x64).

What it does: Periodically checks critical kernel structures (e.g., SSDT, IDT, kernel code) for unauthorized modifications. If tampering is detected, it triggers a BSOD with CRITICAL_STRUCTURE_CORRUPTION.
Impact on AV: Banned direct kernel patching/hooking on x64 systems. AV vendors could no longer use SSDT hooking without crashing systems.


### Shift to Callbacks and User-Mode Hooking
To comply with PatchGuard, security vendors adopted approved kernel callbacks (e.g., PsSetCreateProcessNotifyRoutine, ObRegisterCallbacks) for notifications without patching the kernel.

Hybrid approach:
Kernel component: Registers callbacks to get notifications (e.g., process creation).
User-mode injection: Injects a DLL into processes and hooks NTDLL.dll functions (e.g., NtCreateFile, NtAllocateVirtualMemory) to monitor API calls.
This became the foundation for modern EDR (Endpoint Detection and Response), which evolved from traditional AV.



--- 

### **Circular Kernel Context Logger (CKCL)**
Avast employs a sophisticated, PatchGuard-compliant technique to intercept system calls at the kernel level by leveraging the Circular Kernel Context Logger (CKCL) — a built-in, always-active ETW session with GUID {54DEA73A-ED1F-42A4-AF71-3E63D056F174}.

This method is commonly known as InfinityHook — originally an offensive technique discovered around 2018–2019 (publicly released by Nick Peterson / everdox on GitHub), later repurposed by some EDR/AV vendors (including Avast) for defensive syscall monitoring.
This method is commonly known as InfinityHook ([originally an offensive technique discovered around 2018–2019, later repurposed by some EDR/AV vendors for defensive syscall monitoring]([url](https://github.com/everdox/InfinityHook))).



### Syscall Flow Analysis: Clean Windows vs. Avast-Installed Environment

**Standard Execution Path (Clean Windows – No Avast)**
On a vanilla Windows system without Avast (or any CKCL-hijacking EDR), the syscall invocation for NtTerminateProcess follows the standard kernel dispatch and return flow:
```
nt!NtTerminateProcess
nt!KiSystemServiceCopyEnd+0x25
ntdll!NtTerminateProcess+0x14
KERNELBASE!TerminateProcess+0x30
```
(Stack trace captured from a clean Windows 11 system)

This is the expected, unmodified behavior:
<img width="655" height="146" alt="image" src="https://github.com/user-attachments/assets/48f3d9a2-2789-44fe-a255-1fbd488303d1" />


**Avast-Installed System: Altered Return Path**
When Avast’s kernel driver (typically `aswbids.sys` or `aswVmm.sys`) has applied its hook on the CKCL session, the syscall exit path changes dramatically:

```
nt!NtTerminateProcess
nt!KiSystemServiceExitPico+0x3a8   ← HERE 
ntdll!NtTerminateProcess+0x14
KERNELBASE!TerminateProcess+0x30
```

> **Why the jump to `KiSystemServiceExitPico`?**
> This is not a random detour — it is a direct side effect of Avast’s ETW hijack activating the kernel’s performance tracing infrastructure.
> **Root Cause: Activation of Performance Tracing via PerfGlobalGroupMask**
>The kernel uses a global bitmap called PerfGlobalGroupMask (located in the PCR or KPRCB structure) to control which >performance-related ETW events are enabled system-wide.


Prior to invoking PerfInfoLogSyscallEntry, the system call arguments and the address of the system call handler are pushed onto the stack. If a breakpoint is set on the syscall handler address obtained from the stack and execution is resumed (g) in WinDbg 
<img width="620" height="218" alt="image" src="https://github.com/user-attachments/assets/973badad-4cb2-43c0-aa80-309105f6a6aa" />

it can be observed that the handler replaces or redirects execution within the aswVMmm module.
<img width="654" height="181" alt="image" src="https://github.com/user-attachments/assets/7aa493dd-c387-41a8-aeee-c26693c97a4b" />

**The call stack reflects the following execution sequence:**
<img width="920" height="468" alt="image" src="https://github.com/user-attachments/assets/b39bc3c7-52ec-4935-82d7-81f34f7957ac" />



---

### **What did Avast overwrite?**
<img width="766" height="308" alt="image" src="https://github.com/user-attachments/assets/4db996fc-4f7d-4e14-8f31-eba272286c74" />

The Avast driver pointer is overwritten with the original GetCpuClock function pointer, which is retrieved from offset 0x70 of the HalpPerformanceCounter structure (mov rax, qword ptr [rdi+70h]).

Further analysis reveals the following implementations. The first image demonstrates a pattern-based search used to resolve the address of HalpPerformanceCounter. The second image shows the primary logic, where GetCpuLock is set to 1 and the call to
`HalpPerformanceCounter.FunctionTable.QueryCounter(internal_data)` is overridden.
<img width="1000" height="136" alt="image" src="https://github.com/user-attachments/assets/bcc37db8-a940-40e9-bdee-cbeb046e57b7" />

<img width="1041" height="360" alt="image" src="https://github.com/user-attachments/assets/4abf1b0c-cb29-4f64-b26c-7cf6e2cc6698" />

---
### a bit Reversing Drivers
also i reversed some functions of the AswVmm and the Aswsp 

<img width="350" height="350" alt="image" src="https://github.com/user-attachments/assets/bbe96a35-1273-41df-9b16-717f08db7523" />


there is a function that loads ci.dll (Code Integrity module) which is used to check file signatures and verify integrity, so i went forward and kept digging deeper. then i found another function that's called inside a loop during driver initialization, and after reverse engineering it closely i realized what it's actually doing:
it's looking up the EPROCESS structure for specific processes (via PsLookupProcessByProcessId), then it creates a new protection record with the PID, the EPROCESS pointer, some flags, and appends it to a hidden hash table (basically a global array of 256 linked lists indexed by PID >> 3).
<img width="612" height="322" alt="image" src="https://github.com/user-attachments/assets/b30cb1f5-b42a-4845-b696-e2b48c872397" />


there is this loop right after the driver queries the full process list with ZwQuerySystemInformation(SystemProcessInformation), and it's walking through every single running process on the system:
```
 SystemInformation = ZwQuerySystemInformation(SystemProcessInformation, SystemProcessInformation, i, 0);
    if ( SystemInformation != -1073741820 )
      break;
    ExFreePoolWithTag(SystemProcessInformation, 0x41735370u);
  }
  if ( SystemInformation >= 0 )
  {
    SystemProcessInformation_1 = SystemProcessInformation;
    for ( j = 0; j < 0xFFFF; ++j )
    {
      pick_lock(1);
      if ( !sub_14000F4F8(SystemProcessInformation_1 + 10)) )  // check if PID is already in protection hash table
        ASW_proc_hash_createor(v33, SystemProcessInformation_1 + 11), v33, 0); // if not, create a new protection record
      sub_14000EFAC();      // release the lock
      ASW_PROC_PROTECTOR(SystemProcessInformation_1 + 10, 0); // classify the process & apply protection rules
      v34 = *SystemProcessInformation_1;
      if ( !(_DWORD)v34 )
        break;
      SystemProcessInformation_1 = (unsigned int *)((char *)SystemProcessInformation_1 + v34); // next entry in SYSTEM_PROCESS_INFORMATION list
    }
  }
```
<img width="516" height="184" alt="image" src="https://github.com/user-attachments/assets/b2b940fd-c56a-4c07-829b-a4e28ad82995" />

now lets take look at `ASW_PROC_PROTECTOR`:
this function classifies every process by its filename and assigns custom kernel-level protection rules based on what it is – Avast's own stuff gets maximum lockdown, lsass/csrss get anti-tamper rules, Defender gets special care, etc.
it takes a PID, first makes sure the process is already in the protection hash table (or creates a new entry if it isn't), then fetches the full EPROCESS and uses SeLocateProcessImageName to get the executable's full path. it extracts just the filename by walking backwards from the last backslash, and then runs a huge list of comparisons against known process names.
it checks if the name matches Avast's own binaries like avgsvc.exe, avgui.exe, afwserv.exe, avdump.exe and so on, or critical Windows processes like csrss.exe, lsass.exe, services.exe, winlogon.exe, svchost.exe, explorer.exe, or even other AVs like msmpeng.exe (Defender) and mbamservice.exe (Malwarebytes), plus various system tools such as msiexec.exe, regedit.exe, werfault.exe and many more.
depending on which name matches, it assigns a category number and sets specific protection flags in the hash table record.
Avast's own processes get the **strictest** rules, core system processes like lsass or csrss get strong **anti-tamper** protection, Defender gets special handling, system utilities get medium lockdown and so on. all these flags are then written back into the hash table entry so the driver knows exactly how sensitive the process is and how aggressively it should defend it later.
in short, this function classifies every process by its filename and assigns custom kernel-level protection rules based on what it is – Avast's own stuff gets maximum lockdown, lsass/csrss get anti-tamper rules, Defender gets special care, and so on. it's a very deliberate and thorough self-protection mechanism.

<img width="799" height="463" alt="image" src="https://github.com/user-attachments/assets/fd7f9acd-ecc0-43e2-bed6-ace14466c8e7" />

Additionally, I extracted Unicode strings from the aswVmm driver using

`s -su fffff8045c840000 fffff8045c8a5000`

Interestingly, this revealed which functions were being hooked.
<img width="704" height="487" alt="image" src="https://github.com/user-attachments/assets/f1449ac6-3d39-489c-b5b4-c6fdb05648f3" />

<img width="294" height="525" alt="image" src="https://github.com/user-attachments/assets/a28f10ed-2e00-49b9-b994-2d9c7ca4ed67" />


**I also reversed other drivers during this process, but I believe this is enough for now. Overall, this work was a valuable learning experience, providing hands-on insight into kernel hooking, ETW internals, low-level Windows debugging, and driver reverse engineering.
Thank you all for reading.**

---
Many thanks to [archie](https://archie-osu.github.io/etw/hooking/2025/04/09/hooking-context-swaps-with-etw.html) and [Denis Skvortcov](https://the-deniss.github.io/) for their excellent research and write-ups.





















