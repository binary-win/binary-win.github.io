# **Browser Dumping: The Core Tactic Behind Most Infostealers**

> This blog contains my own research collected from the internet, along with ideas from other blogs and studies. While many parts are written in my own words, the Most sections were copied directly from [external sources](https://github.com/xaitax/Chrome-App-Bound-Encryption-Decryption/blob/main/docs/RESEARCH.md) because they were already very well written and clearly expressed. This blog is mainly for sharing my personal notes and learning journey.


In today’s threat landscape, **credential stuffing**, **ransomware affiliate attacks**, and **initial access** sales are fueled almost entirely by **stolen credentials**. At the heart of nearly every major infostealer family lies one dominant technique: **Browser Dumping**.
This method is responsible for the vast majority of compromised accounts traded on underground markets. billions of **credentials**, **cookies**, and **tokens** harvested year after year. 


<img width="860" height="520" alt="image" src="https://github.com/user-attachments/assets/27e154b7-c5d0-435a-802f-d96fd61a2539" />
> _Advertisement for TitanStealer, first offered for sale in November 2022 via the Russian-language BHF and Dark2Web forums (Source: Kela)_




#### 1. **Introduction: The Evolution of Local Data Protection in Chrome**
For years, **Chromium-based** browsers on Windows relied on the Data Protection API `(DPAPI)` to secure sensitive user data stored locally such as `cookies`, `passwords`, `payment information`, and the like. `DPAPI` binds data to the **logged-in user's credentials**, offering a solid baseline against offline attacks (e.g., a stolen hard drive) and unauthorized access by other users on the **same machine**. However, DPAPI's Achilles' heel has always been its permissiveness within the user's own session: any application running as the same user, with the same privilege level as Chrome, can invoke `CryptUnprotectData` and decrypt this data. This vulnerability has been a perennial favorite for **infostealer malware**.

### Decrypt the V10 Key of Chrome ( DPAPI )
Found **Local State** `::os_crypt:encrypted_key `( DPAPI decryption)

```c
Start
  ↓
Read Local State (JSON)
  ↓
Extract "encrypted_key" (Base64)
  ↓
Decode Base64
  ↓
Remove "DPAPI" prefix
  ↓
CryptUnprotectData() → Master Key
  ↓
Open Login Data (SQLite)
  ↓
For each password:
  ├─ Check prefix (v10)
  ├─ v10 → DPAPI decrypt  [v] [1] [0] [DPAPI encrypted data...]
  ↓
Display results
  ↓
End


┌──────────────────────────────────────────────────────────────────┐
│  v10 Encrypted BLOB Format (Chromium / Chrome / Edge / etc.)     
└──────────────────────────────────────────────────────────────────┘
   ↓
┌───┬───┬───┬────────────────┬──────────────────────────┬──────────────────┐
│ v │ 2 │ 0 │   Nonce (12)   │   Ciphertext        │   Tag (16)       │
└───┴───┴───┴────────────────┴──────────────────────────┴──────────────────┘
  0   1   2 |----- 

```




To counter this, Google introduced `App-Bound Encryption (ABE)` in Chrome (_publicly announced around version 127, July 2024_). `ABE` is a significant architectural shift designed to dramatically raise the bar for attackers. Its core principle is to ensure that the primary decryption keys for sensitive Chrome data are only accessible to legitimate Chrome processes, thereby mitigating trivial data theft by same-user, same-privilege malware.

#### **1.1. Foundational Concepts of ABE**
**Primary Goal**: Prevent an attacker operating with the same privilege level as Chrome from trivially calling DPAPI to decrypt sensitive data.

**Acknowledged Limitations (Non-Goals)**: ABE does not aim to prevent attackers with higher privileges (Administrator, SYSTEM, kernel drivers) or those who can successfully inject code into Chrome. The official Google design documents explicitly recognize code injection as a potent bypass vector, a technique this project leverages for legitimate research and data recovery demonstrations.

**Underlying Mechanism**: ABE introduces an intermediary COM service (part of Chrome's Elevation Service) that acts as a gatekeeper for the DPAPI-unwrapping of a critical session key. This service verifies the "app identity" of the caller.

**Initial Identity Verification Method**: The first iteration relies on path validation of the calling executable. While digital signature validation was considered, path validation was chosen for the initial rollout to "descope the complexity" (as noted in a 2024 update to Google's design document), deemed sufficient against the immediate threat model.

Google's conceptual diagram provides a clear overview:
<img width="957" height="611" alt="image" src="https://github.com/user-attachments/assets/e0ef5d36-4fcf-4b0e-9d5d-9cfefd8e1520" />

---

#### 2. **The ABE Mechanism: A Step-by-Step Breakdown**
ABE employs a multi-layered strategy for key management and data encryption:

**The app_bound_key (Session Key):**
A unique **32-byte AES-256** key is the target plaintext that applications like Chrome's OSCrypt use.
This key is what that use to recover for subsequent data decryption.

**Generation of `validation_data`  and `app_bound_key` Wrapping (During Encryption by Chrome):**
- When Chrome (via OSCrypt) needs to protect the app_bound_key using ABE, it calls the IElevator::EncryptData COM method.
- **Caller Validation Data Generation**: Inside `IElevator::EncryptData`, the service first generates `validation_data`. If `ProtectionLevel::PROTECTION_PATH_VALIDATION` is specified, this involves:
  -  Obtaining the calling process's executable path (`GetProcessExecutablePath`).
  - Normalizing this path using a specific routine (`MaybeTrimProcessPath`), which removes the .exe name, common temporary/application subfolders (like "Application", "Temp", version strings), and standardizes "Program Files (x86)" to "Program Files". This results in a canonical base installation path.

- This normalized path string (UTF-8 encoded) becomes the core of the validation_data. The ProtectionLevel itself is also prepended to this data.


**User-Context DPAPI Encryption**: This `data_to_encrypt` blob is then encrypted using `CryptProtectData` under the **calling user's DPAPI context** (achieved via `ScopedClientImpersonation`).

**System-Context DPAPI Encryption (Outer Layer):** The result from the **user-context DPAPI encryption** is then encrypted again using `CryptProtectData`, this time under the **SYSTEM DPAPI** context (or the service's own context if not explicitly SYSTEM). This creates a "**DPAPI-ception**" or layered DPAPI protection.

**This doubly DPAPI-wrapped blob is what IElevator::EncryptData returns as the ciphertext BSTR.**


As you can see it expose only the `Interface {1BF5208B-295F-4992-B5F4-3A9BB6494838}` : **IElevator2Chrome**

<img width="591" height="109" alt="image" src="https://github.com/user-attachments/assets/dbd7e8d6-023c-4017-9f27-8501e18b43cc" />



<img width="817" height="488" alt="image" src="https://github.com/user-attachments/assets/ded7ef0e-c4a9-47ac-8fa3-d39a6520eed8" />


In addition, you can explore other interfaces that are not exposed in the current channel (e.g., Stable), but are registered and available in specific Chrome channels like Beta, Dev, Canary, or Chromium builds.

<img width="1066" height="329" alt="image" src="https://github.com/user-attachments/assets/564e0614-d436-4b68-ad8f-a44a0f0870d6" />



**Storage in `Local State`:**
- The ciphertext BSTR received from `IElevator::EncryptData` is Base64-encoded.
- The prefix APPB (ASCII: `0x41 0x50 0x50 0x42`) is prepended.
- This final string is stored in Local State as `os_crypt.app_bound_encrypted_key`.

**The IElevator COM Service (The Gatekeeper for Decryption):**
When Chrome (or this project's injected DLL) needs the plaintext `app_bound_key`.
It instantiates the IElevator COM object using browser-specific CLSIDs/IIDs:
- Google Chrome: CLSID: `{708860E0-F641-4611-8895-7D867DD3675B}`, IID: `{463ABECF-410D-407F-8AF5-0DF35A005CC8}`
- Microsoft MsEdge: CLSID: `{1FCBE96C-1697-43AF-9140-2897C7C69767}` , IID: `{C9C2B807-7731-4F34-81B7-44FF7779522B}`

The APPB-prefixed, Base64-encoded string from **Local State** is decoded and the `APPB` prefix stripped. This resulting blob (the doubly DPAPI-wrapped key) is passed to `IElevator::DecryptData`.

**Unwrapping and Path Validation by `IElevator::DecryptData`:**
- **System-Context DPAPI Decryption:** The input blob is first decrypted using `CryptUnprotectData` under the **SYSTEM DPAPI context**. This removes the outer DPAPI layer.

- **User-Context DPAPI Decryption:** The intermediate result is then decrypted using `CryptUnprotectData` under the calling **user's DPAPI** context (via ScopedClientImpersonation). This removes the inner DPAPI layer, yielding a plaintext blob.

- Extraction of Validation Data and Plaintext Key: This plaintext blob is structured as `[validation_data_length] [validation_data][app_bound_key_length][app_bound_key]`. The service uses `PopFromStringFront` to extract the original validation_data and then the `app_bound_key`.


**Data Encryption/Decryption using the app_bound_key:**
Chrome's OSCrypt then uses this recovered **32-byte AES** key with **AES-256-GCM** to **encrypt/decrypt** actual user data (`cookies`, `passwords`), **which are typically prefixed (e.g., v20).**

----

#### **3. Dissecting Encrypted Data Structures**

**Local State and the app_bound_encrypted_key**
_Typical Location_: `%LOCALAPPDATA%\<BrowserVendor>\<BrowserName>\User Data\Local State`
_Relevant JSON Key_: `os_crypt.app_bound_encrypted_key`.
_Format A string value_: `APPB<Base64EncodedSystemDPAPIWrappedUserDPAPIWrappedValidationDataAndKey>`.

**Data items encrypted with the app_bound_key generally adhere to a consistent format:**
**Prefix**: A version or type prefix string. For `cookies`, `passwords`, and `payment data` observed thus far, this is typically **v20** `(ASCII: 0x76 0x32 0x30)` . Older data encrypted solely with DPAPI might use **prefixes** like `v10` or `v11`.
**Nonce (IV):** A `12-byte` Initialization Vector, essential for the security of **AES-GCM** mode.
**Ciphertext:** The actual encrypted data, variable in length.
**Authentication Tag:** A `16-byte` GCM authentication tag, which ensures both the integrity and authenticity of the decrypted ciphertext.

**Overall Blob Structure:**
`[Prefix (e.g., 3 bytes for "v20")][IV (12 bytes)][Ciphertext (variable length)][Tag (16 bytes)]`


**Cookie Value Specifics (from encrypted_value in Cookies DB)**
A notable observation during the development of this tool is that after successfully decrypting a v20-prefixed cookie blob using AES-GCM with the app_bound_key, the first 32 bytes of the resulting plaintext appear to be some form of metadata or padding. The actual cookie value string begins after this DECRYPTED_COOKIE_VALUE_OFFSET of 32 bytes.

**The value of the cookie starts after the first 32 bytes (Thanks to luci4_vx::https://luci4.net) ** -> _after Decryption_ go forward in 32 bytes.


**Passwords (from password_value in Login Data DB) & Payment Information**
These data types also use `v20-prefixed` blobs.
Unlike cookies, the entire decrypted plaintext (after accounting for the v20 prefix, IV, and tag during the AES-GCM decryption process) is generally considered to be the sensitive value itself (e.g., the password string, credit card number, or CVC).

---

#### **4. Alternative Decryption Vectors & Chrome's Evolving Defenses**

#### Administrator-Level Decryption (e.g., `runassu/chrome_v20_decryption` PoC)
The proof-of-concept by runassu illustrates that if an attacker possesses Administrator privileges, the app_bound_key can potentially be decrypted. This aligns with ABE's stated non-goal of protecting against higher-privilege attackers.
[REPO](https://github.com/runassu/chrome_v20_decryption?tab=readme-ov-file)

The PoC's description of needing to decrypt the **app_bound_encrypted_key** from Local State **first with SYSTEM DPAPI**, then **user DPAPI**, directly matches the initial steps within the legitimate `IElevator::DecryptData` function as seen in `elevator.cc`. An administrator can perform these steps outside of the IElevator service.

After these two DPAPI unwrap steps, the result would be the
`[validation_data_length][validation_data][app_bound_key_length][app_bound_key]` 
plaintext. An admin tool could then simply parse this structure to extract the app_bound_key directly, without needing to perform **path validation**.
The runassu PoC's claim that this result is "**not the final app_bound_key**" and requires a further **AES-GCM decryption with a key hardcoded in elevation_service.exe is intriguing**.
This additional layer is not part of the standard `IElevator::DecryptData` flow for returning the app_bound_key to OSCrypt, as evidenced by elevator.cc. The plaintext_str returned by `IElevator::DecryptData` is the application-level key.

<img width="874" height="1043" alt="image" src="https://github.com/user-attachments/assets/1c83b854-b16b-4c5c-98b1-c018c893990d" />



The PoC's extra step might be attempting to decrypt data that has undergone an additional, internal transformation within Chrome, possibly related to the `PreProcessData/PostProcessData` functions seen in `elevator.cc` (conditionally compiled with `BUILDFLAG(GOOGLE_CHROME_BRANDING))`. These functions might apply another layer of encryption using a service-internal key for specific branded builds or key versions.
Alternatively, the PoC might be targeting a different internal key or an older/variant ABE scheme.

**Hardcoded Keys in elevation_service.exe:** The presence of hardcoded keys in elevation_service.exe (as mentioned by the PoC for `ChaCha20_Poly1305` or `AES-256-GCM`) would most likely be for such internal service operations or specific recovery mechanisms, rather than the primary ABE flow that returns the key to OSCrypt.

**Stability Concerns**: Relying on such internal administrator-level method, undocumented layers and hardcoded keys is highly unstable and prone to **break with Chrome updates**.

<img width="817" height="250" alt="image" src="https://github.com/user-attachments/assets/798e3a77-b8c0-407c-9465-cafa44c52b3a" />



[HardCoded KEYs:](https://github.com/bitwarden/clients/blob/main/apps/desktop/desktop_native/bitwarden_chromium_import_helper/src/windows/crypto.rs)

<img width="971" height="748" alt="image" src="https://github.com/user-attachments/assets/06a86a96-ece1-4ae9-8ca7-f572cf00d405" />

<img width="803" height="363" alt="image" src="https://github.com/user-attachments/assets/b4ef21ca-7c7d-4a7b-b137-8610eb37f3de" />



----



#### Device Bound Session Credentials (DBSC) 
As an overlapping and complementary security effort, Google has been developing Device Bound Session Credentials (DBSC), available for Origin Trial in Chrome 135. DBSC aims to combat cookie theft by cryptographically binding session cookies to the device.

Mechanism: When a DBSC session is initiated, the browser generates a public-private key pair, storing the private key securely (ideally using hardware like a TPM). The server associates the session with the public key. Periodically, the browser proves possession of the private key to refresh the (typically short-lived) session cookie.
Relevance to ABE: While ABE protects data at rest on the user's device, DBSC focuses on making stolen session cookies useless if exfiltrated and used on another device. They are two distinct but synergistic layers of defense against session hijacking. An attacker bypassing ABE to get cookies might still find those cookies unusable elsewhere if they are DBSC-protected.

[**DBSC is exclusively available in Google Chrome (starting with version 145), and not yet supported in Firefox, Safari, Edge, or other major browsers.**](https://chromestatus.com/feature/5140168270413824)



---

#### 6. Key Insights from Google's ABE Design Document & Chromium Source Code

Insights from Google's design documents and the Chromium source code (`elevator.h` , `elevator.cc`, `caller_validation.h` , `caller_validation.cc` ) provide a comprehensive understanding:

- **Security Approach Change:** Initial proposal required digital signature validation, but was descoped to path validation for simplicity, assessed as providing "equivalent protection against non-admin attackers"

- **OSCrypt Module Modifications:** Modified to use new IPC mechanisms to communicate with Elevation Service instead of making direct DPAPI calls

- **Key Encryption Delegates System:** OSCrypt iterates through a list of delegates - one for legacy DPAPI keys, another for ABE-protected keys via IPC

- **Stateless Service Design:** IElevator service functions as a largely stateless encrypt/decrypt primitive with no persistent storage required for ABE operations

- **Acknowledged Injection Vulnerability:** Design document explicitly admits "an attacker could inject code into Chrome browser and call the IPC interface" - defeating a determined attacker using this technique would be hard



---

#### 9. References and Further Reading
- **the main one** [xaitax](https://github.com/xaitax/Chrome-App-Bound-Encryption-Decryption/blob/main/docs/RESEARCH.md )

- [Maldev-Academy ](https://github.com/Maldev-Academy/DumpBrowserSecrets)


- [Google Security Blog: Improving the security of Chrome cookies on Windows (July 30, 2024)
](https://security.googleblog.com/2024/07/improving-security-of-chrome-cookies-on.html)

- [Google Design Document: Chrome app-bound encryption Service (formerly: Chrome Elevated Data Service) (Original: Jan 25, 2021, with later updates)](https://drive.google.com/file/d/1xMXmA0UJifXoTHjHWtVir2rb94OsxXAI/view)

- [Chrome Developers Blog (DBSC): Origin trial: Device Bound Session Credentials in Chrome](https://developer.chrome.com/blog/dbsc-origin-trial)


- [runassu's PoC (Admin-level decryption): chrome_v20_decryption](https://github.com/runassu/chrome_v20_decryption)

- [SilentDev33's ChromeAppBound-key-injection ](https://github.com/SilentDev33/ChromeAppBound-key-injection)






 














