# Browser Dumping: The Core Tactic Behind Most Infostealers

> **This blog is a collection of my own research from the internet, along with insights from other blogs and studies. While similar information can be found elsewhere, my main goal is to write everything in my own words and style. Please consider this as my personal notes and learning journal.**

#### 1. Introduction: The Evolution of Local Data Protection in Chrome
For years, Chromium-based browsers on Windows relied on the Data Protection API (DPAPI) to secure sensitive user data stored locally such as cookies, passwords, payment information, and the like. DPAPI binds data to the logged-in user's credentials, offering a solid baseline against offline attacks (e.g., a stolen hard drive) and unauthorized access by other users on the same machine. However, DPAPI's Achilles' heel has always been its permissiveness within the user's own session: any application running as the same user, with the same privilege level as Chrome, can invoke CryptUnprotectData and decrypt this data. This vulnerability has been a perennial favorite for infostealer malware.

To counter this, Google introduced App-Bound Encryption (ABE) in Chrome (publicly announced around version 127, July 2024). ABE is a significant architectural shift designed to dramatically raise the bar for attackers. Its core principle is to ensure that the primary decryption keys for sensitive Chrome data are only accessible to legitimate Chrome processes, thereby mitigating trivial data theft by same-user, same-privilege malware.

#### 1.1. Foundational Concepts of ABE
**Primary Goal**: Prevent an attacker operating with the same privilege level as Chrome from trivially calling DPAPI to decrypt sensitive data.

**Acknowledged Limitations (Non-Goals)**: ABE does not aim to prevent attackers with higher privileges (Administrator, SYSTEM, kernel drivers) or those who can successfully inject code into Chrome. The official Google design documents explicitly recognize code injection as a potent bypass vector, a technique this project leverages for legitimate research and data recovery demonstrations.

**Underlying Mechanism**: ABE introduces an intermediary COM service (part of Chrome's Elevation Service) that acts as a gatekeeper for the DPAPI-unwrapping of a critical session key. This service verifies the "app identity" of the caller.

**Initial Identity Verification Method**: The first iteration relies on path validation of the calling executable. While digital signature validation was considered, path validation was chosen for the initial rollout to "descope the complexity" (as noted in a 2024 update to Google's design document), deemed sufficient against the immediate threat model.

Google's conceptual diagram provides a clear overview:
<img width="968" height="619" alt="image" src="https://github.com/user-attachments/assets/5d463de7-eb8f-4f74-9783-afd98711e3dc" />



##### 2. The ABE Mechanism: A Step-by-Step Breakdown
ABE employs a multi-layered strategy for key management and data encryption:

1.**The app_bound_key (Session Key):**
A unique 32-byte AES-256 key is the target plaintext that applications like Chrome's OSCrypt use.
This key is what that use to recover for subsequent data decryption.

2.**Generation of `validation_data`  and `app_bound_key` Wrapping (During Encryption by Chrome):**
- When Chrome (via OSCrypt) needs to protect the app_bound_key using ABE, it calls the IElevator::EncryptData COM method.
- **Caller Validation Data Generation**: Inside `IElevator::EncryptData`, the service first generates `validation_data`. If `ProtectionLevel::PROTECTION_PATH_VALIDATION` is specified, this involves:
  -  Obtaining the calling process's executable path (GetProcessExecutablePath).
  - Normalizing this path using a specific routine (MaybeTrimProcessPath), which removes the .exe name, common temporary/application subfolders (like "Application", "Temp", version strings), and standardizes "Program Files (x86)" to "Program Files". This results in a canonical base installation path.

This normalized path string (UTF-8 encoded) becomes the core of the validation_data. The ProtectionLevel itself is also prepended to this data.


Payload Construction: The validation_data (with its length) is prepended to the plaintext app_bound_key (also with its length). This forms the data_to_encrypt.


User-Context DPAPI Encryption: This data_to_encrypt blob is then encrypted using CryptProtectData under the calling user's DPAPI context (achieved via ScopedClientImpersonation).

System-Context DPAPI Encryption (Outer Layer): The result from the user-context DPAPI encryption is then encrypted again using CryptProtectData, this time under the SYSTEM DPAPI context (or the service's own context if not explicitly SYSTEM). This creates a "DPAPI-ception" or layered DPAPI protection.

This doubly DPAPI-wrapped blob is what IElevator::EncryptData returns as the ciphertext BSTR.


As you can see it expose only the `Interface {1BF5208B-295F-4992-B5F4-3A9BB6494838}` : **IElevator2Chrome**

<img width="591" height="109" alt="image" src="https://github.com/user-attachments/assets/dbd7e8d6-023c-4017-9f27-8501e18b43cc" />



<img width="817" height="488" alt="image" src="https://github.com/user-attachments/assets/ded7ef0e-c4a9-47ac-8fa3-d39a6520eed8" />





<img width="1066" height="329" alt="image" src="https://github.com/user-attachments/assets/564e0614-d436-4b68-ad8f-a44a0f0870d6" />




3.**Storage in Local State:**
































