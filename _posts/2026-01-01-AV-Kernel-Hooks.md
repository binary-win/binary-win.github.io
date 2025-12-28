#  Avast Hooking the Kernel (AV hooks)


  
##### **Summary**
We walk through undocumented internals such as CKCL, PerfInfoLogSysCallEntry, HalPrivateDispatch / HalpPerformanceCounter, and explain how ETW can be abused to gain early control over syscall dispatch.
also a little reversing avast driver 

Real-World Example that we check : Avast Kernel Hooks
Avast driver (`aswVmm.sys`) hook syscalls using this technique Intercepts sensitive operations like etc:
`NtTerminateProcess`
`NtOpenProcess`




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




### **Circular Kernel Context Logger (CKCL)**
The Circular Kernel Context Logger (CKCL) is a built-in, always-active Event Tracing for Windows (ETW) session in Windows, specifically designed for lightweight, high-frequency kernel event capture. It has been present since Windows Vista and is enabled by default in all modern Windows versions.



##### Syscall Interception via CKCL











































