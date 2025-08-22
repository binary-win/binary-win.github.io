#Elastic Defend Bypass: UAC Bypass Chain Leads to Silent Local Privilege Escalation (LPE)

![Preview](https://raw.githubusercontent.com/binary-win/binary-win.github.io/main/image/videos/uac_chain_bypass.mkv)

<video width="700" height="500" controls>
  <source src="https://raw.githubusercontent.com/binary-win/binary-win.github.io/main/image/videos/uac_chain_bypass.mkv" type="video/mp4">


  
**Summary**
A chained technique has been identified that allows a local, unprivileged
attacker to achieve **silent privilege escalation to administrator** by
bypassing protections enforced by **Elastic Defend v9.0.4**. The method
leverages a **trusted auto-elevated Windows binary (fodhelper.exe)** in
conjunction with a **registry hijack** and **COM object execution**,  resulting in
arbitrary code execution at elevated privileges — **without triggering a UAC
prompt or EDR detection**.


This technique bypasses both behavioral and policy-based protections
implemented by Elastic Defend, including detection logic that typically
monitors UAC bypass patterns and LOLBin misuse.


---

### **Product Information**
- Product: Elastic Endpoint Security (Elastic Agent)
- Version: v9.0.4
- Platform: Windows 10 / Windows 11 (x64)

---

### **Vulnerability Classification**
- Local Privilege Escalation (LPE)
- User Account Control (UAC) Bypass
- Elastic EDR Detection Evasion
- Living-off-the-Land Binary (LOLBin) Abuse

----

### **Chained Technique Details:**
1. **Initial vector**: User-level process executes a signed, auto-elevated
binary (**fodhelper.exe**).

2. **Hijack or manipulation**: Registry key (*HKCU\Software\Classes\ms-
settings\Shell\Open\command*) is modified to point to an attacker-
controlled executable.

3.**COM Object Abuse : attacker-controlled executable try to Execute Cmd
via COM Object.**

4. **Privilege Escalation**: fodhelper.exe runs the payload as administrator
**without any UAC prompt**. The COM object also spawns its child process
(e.g., cmd.exe) **elevated**, despite Elastic EDR typically reducing
fodhelper-launched processes to medium integrity if suspicious.



{F4655955}


---
### **Why it Works?**

#### Part 1 – Fodhelper Behavior
fodhelper.exe is a trusted, auto-elevated binary signed by Microsoft. When executed, it queries specific registry keys (previously discussed) to determine which executable to launch.

Attackers often exploit this by redirecting those registry values to binaries like cmd.exe or powershell.exe. However, such high-risk executables are typically monitored and blocked by modern security solutions (e.g., Elastic EDR), causing the UAC bypass to fail.

Interestingly, if these registry values point to low-profile binaries such as calc.exe or a custom executable (e.g., uac.exe), execution proceeds without detection.

The issue:

> The custom binary is launched without elevation — it inherits the standard user token.

####  Part 2 – COM Elevation via cmstplua

To silently escalate privileges, we leverage a COM-based UAC bypass using the following COM object:

CLSID: {3E5FC7F9-9A51-4367-9063-A120244FBEC7}

ProgID: cmstplua


This is an undocumented COM object that exposes the method ShellExec().

Detection mechanisms typically look for:
A parent process of dllhost.exe
The CLSID being passed in the command-line arguments


Here’s the key insight:
> cmstplua is an InprocServer32 COM object, which allows instantiation directly within the current process using CoCreateInstance.

This eliminates the need to spawn dllhost.exe. Instead, we load the COM object in-process and call ShellExec() directly.

However:

> Since the COM object runs elevated, invoking ShellExec() from a non-elevated process triggers a UAC prompt, making the technique noisy and detectable.


### Part 3 – Chaining Fodhelper + cmstplua
To bypass the UAC prompt and achieve silent elevation:

1. fodhelper.exe is launched — it's auto-elevated by default.


2. Registry hijacking causes it to execute a custom binary.


3. Inside the custom binary (now running in an elevated context), CoCreateInstance is used to instantiate the cmstplua COM object.


4. The ShellExec() method is then called from an elevated context.

> Since the custom binary inherits elevation from fodhelper.exe, calling CoCreateInstance and ShellExec() does not trigger a UAC prompt. The entire chain executes silently with elevated privileges.




---
### **Impact**
An unprivileged attacker can elevate to administrator silently, gaining full control of the system while completely bypassing EDR detection and logging. This significantly undermines endpoint security and could be used to:
• Install persistence mechanisms
• Exfiltrate credentials
• Deploy malware or ransomware
• Conduct lateral movement


----


PoC & Artifacts:
• Attached: UAC_Bypass_Elastic_PoC.zip
• Executable - The PoC source code is available upon request. I am
happy to provide it privately once the report has been reviewed
and confirmed as a valid issue.
• Video recording of exploit
• png and svg of the Flow
• README.txt


----


{F4655992}

*sha256:
c6291e24c3f53247e00cbf32cd5aea79d765736f*


---

**References:**
- MITRE ATT&CK T1548.002: Abuse Elevation Control Mechanism: Bypass User Account Control
- MITRE ATT&CK T1559.001 :Inter-Process Communication: Component Object Model


---


**Disclosure Policy:
This report is submitted responsibly through HackerOne. I have not publicly
disclosed the technique and await your analysis and feedback before further
action.**



{F4656222}
{F4656225}




