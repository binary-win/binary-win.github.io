
##### **Summary**
We walk through undocumented internals such as CKCL, PerfInfoLogSysCallEntry, HalPrivateDispatch / HalpPerformanceCounter, and explain how ETW can be abused to gain early control over syscall dispatch.
also a little reversing avast driver 

Real-World Example that we check : Avast Kernel Hooks
Avast driver (`aswVmm.sys`) hook syscalls using this technique Intercepts sensitive operations like etc:
`NtTerminateProcess`
`NtOpenProcess`

<img width="866" height="277" alt="image" src="https://github.com/user-attachments/assets/fc917027-b060-4098-ad3c-9e9b06cd89f3" />


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




<img width="1000" height="136" alt="image" src="https://github.com/user-attachments/assets/bcc37db8-a940-40e9-bdee-cbeb046e57b7" />

<img width="1041" height="360" alt="image" src="https://github.com/user-attachments/assets/4abf1b0c-cb29-4f64-b26c-7cf6e2cc6698" />






with great thanks to [archie](https://archie-osu.github.io/etw/hooking/2025/04/09/hooking-context-swaps-with-etw.html) and [Denis Skvortcov](https://the-deniss.github.io/)
























