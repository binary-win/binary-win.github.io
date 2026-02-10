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


In addition, you can explore other interfaces that are not exposed in the current channel (e.g., Stable), but are registered and available in specific Chrome channels like Beta, Dev, Canary, or Chromium builds.

<img width="1066" height="329" alt="image" src="https://github.com/user-attachments/assets/564e0614-d436-4b68-ad8f-a44a0f0870d6" />



3.**Storage in `Local State`:**
- The ciphertext BSTR received from `IElevator::EncryptData` is Base64-encoded.
- The prefix APPB (ASCII: `0x41 0x50 0x50 0x42`) is prepended.
- This final string is stored in Local State as `os_crypt.app_bound_encrypted_key`.

4.**The IElevator COM Service (The Gatekeeper for Decryption):**
When Chrome (or this project's injected DLL) needs the plaintext `app_bound_key`.
It instantiates the IElevator COM object using browser-specific CLSIDs/IIDs:
- Google Chrome: CLSID: `{708860E0-F641-4611-8895-7D867DD3675B}`, IID: `{463ABECF-410D-407F-8AF5-0DF35A005CC8}`
- Microsoft MsEdge: CLSID: `{1FCBE96C-1697-43AF-9140-2897C7C69767}` , IID: `{C9C2B807-7731-4F34-81B7-44FF7779522B}`

The APPB-prefixed, Base64-encoded string from Local State is decoded and the APPB prefix stripped. This resulting blob (the doubly DPAPI-wrapped key) is passed to IElevator::DecryptData.

5.**Unwrapping and Path Validation by `IElevator::DecryptData`:**
- System-Context DPAPI Decryption: The input blob is first decrypted using CryptUnprotectData under the SYSTEM DPAPI context. This removes the outer DPAPI layer.

- User-Context DPAPI Decryption: The intermediate result is then decrypted using CryptUnprotectData under the calling user's DPAPI context (via ScopedClientImpersonation). This removes the inner DPAPI layer, yielding a plaintext blob.

- Extraction of Validation Data and Plaintext Key: This plaintext blob is structured as `[validation_data_length] [validation_data][app_bound_key_length][app_bound_key]`. The service uses `PopFromStringFront` to extract the original validation_data and then the `app_bound_key`.


6.Data Encryption/Decryption using the app_bound_key:
Chrome's OSCrypt (or this project's DLL) then uses this recovered 32-byte AES key with AES-256-GCM to encrypt/decrypt actual user data (cookies, passwords), which are typically prefixed (e.g., v20).



#### 3. Dissecting Encrypted Data Structures

4.1. Local State and the app_bound_encrypted_key
Typical Location: `%LOCALAPPDATA%\<BrowserVendor>\<BrowserName>\User Data\Local State ` (e.g., Google\Chrome\User Data\Local State).
Relevant JSON Key: `os_crypt.app_bound_encrypted_key`.
Format: A string value: ` "APPB<Base64EncodedSystemDPAPIWrappedUserDPAPIWrappedValidationDataAndKey>"`.



Data items encrypted with the app_bound_key generally adhere to a consistent format:

Prefix: A version or type prefix string. For cookies, passwords, and payment data observed thus far, this is typically v20 (ASCII: 0x76 0x32 0x30). Older data encrypted solely with DPAPI might use prefixes like v10 or v11.
Nonce (IV): A 12-byte Initialization Vector, essential for the security of AES-GCM mode.
Ciphertext: The actual encrypted data, variable in length.
Authentication Tag: A 16-byte GCM authentication tag, which ensures both the integrity and authenticity of the decrypted ciphertext.
Overall Blob Structure:
` [Prefix (e.g., 3 bytes for "v20")][IV (12 bytes)][Ciphertext (variable length)][Tag (16 bytes)]
`

4.3. Cookie Value Specifics (from encrypted_value in Cookies DB)
A notable observation during the development of this tool is that after successfully decrypting a v20-prefixed cookie blob using AES-GCM with the app_bound_key, the first 32 bytes of the resulting plaintext appear to be some form of metadata or padding. The actual cookie value string begins after this DECRYPTED_COOKIE_VALUE_OFFSET of 32 bytes.
4.4. Passwords (from password_value in Login Data DB) & Payment Information
These data types also use v20-prefixed blobs.
Unlike cookies, the entire decrypted plaintext (after accounting for the v20 prefix, IV, and tag during the AES-GCM decryption process) is generally considered to be the sensitive value itself (e.g., the password string, credit card number, or CVC).



### Alternative Decryption Vectors & Chrome's Evolving Defenses
#### 5.1. Administrator-Level Decryption (e.g., runassu/chrome_v20_decryption PoC)
The proof-of-concept by runassu illustrates that if an attacker possesses Administrator privileges, the app_bound_key can potentially be decrypted. This aligns with ABE's stated non-goal of protecting against higher-privilege attackers.

The PoC's description of needing to decrypt the app_bound_encrypted_key from Local State first with SYSTEM DPAPI, then user DPAPI, directly matches the initial steps within the legitimate IElevator::DecryptData function as seen in elevator.cc. An administrator can perform these steps outside of the IElevator service.
After these two DPAPI unwrap steps, the result would be the [validation_data_length][validation_data][app_bound_key_length][app_bound_key] plaintext. An admin tool could then simply parse this structure to extract the app_bound_key directly, without needing to perform path validation.
The runassu PoC's claim that this result is "not the final app_bound_key" and requires a further AES-GCM decryption with a key hardcoded in elevation_service.exe is intriguing.
This additional layer is not part of the standard IElevator::DecryptData flow for returning the app_bound_key to OSCrypt, as evidenced by elevator.cc. The plaintext_str returned by IElevator::DecryptData is the application-level key.
The PoC's extra step might be attempting to decrypt data that has undergone an additional, internal transformation within Chrome, possibly related to the PreProcessData/PostProcessData functions seen in elevator.cc (conditionally compiled with BUILDFLAG(GOOGLE_CHROME_BRANDING)). These functions might apply another layer of encryption using a service-internal key for specific branded builds or key versions.
Alternatively, the PoC might be targeting a different internal key or an older/variant ABE scheme.
Hardcoded Keys in elevation_service.exe: The presence of hardcoded keys in elevation_service.exe (as mentioned by the PoC for ChaCha20_Poly1305 or AES-256-GCM) would most likely be for such internal service operations or specific recovery mechanisms, rather than the primary ABE flow that returns the key to OSCrypt.
Stability Concerns: Relying on such internal administrator-level method, undocumented layers and hardcoded keys is highly unstable and prone to break with Chrome updates. The method employed by this project (injecting and calling the official IElevator::DecryptData COM interface) is more aligned with the intended client interaction path and thus inherently more stable, despite the injection vector.


#### 5.2. Remote Debugging Port (--remote-debugging-port) and Its Mitigation
Attackers had also turned to Chrome's remote debugging capabilities as a vector to exfiltrate cookies, effectively sidestepping ABE's file-based protections.

Chrome's Countermeasure (Chrome 136+): As detailed in a Chrome Developers blog post, Google addressed this by changing the behavior of the --remote-debugging-port and --remote-debugging-pipe command-line switches. Starting with Chrome 136, these switches will no longer function when Chrome is launched with its default user data directory. To enable remote debugging, users must now also specify the --user-data-dir switch, pointing Chrome to a non-standard, separate data directory. This ensures that any debugging session operates on an isolated profile, using a different encryption key, thereby safeguarding the user's primary profile data.
Bypass Simplicity: While this change adds a hurdle, it's worth noting that an attacker can control Chrome's launch parameters (e.g., by modifying shortcuts or through malware that relaunches Chrome), they could potentially still launch Chrome with both --remote-debugging-port and a temporary --user-data-dir, then attempt to import or access data if Chrome allows such operations into a fresh, debuggable profile. The effectiveness of the debug port mitigation hinges on preventing unauthorized modification of launch parameters and on Chrome's policies regarding data access in such scenarios.
#### 5.3. Device Bound Session Credentials (DBSC)
As an overlapping and complementary security effort, Google has been developing Device Bound Session Credentials (DBSC), available for Origin Trial in Chrome 135. DBSC aims to combat cookie theft by cryptographically binding session cookies to the device.

Mechanism: When a DBSC session is initiated, the browser generates a public-private key pair, storing the private key securely (ideally using hardware like a TPM). The server associates the session with the public key. Periodically, the browser proves possession of the private key to refresh the (typically short-lived) session cookie.
Relevance to ABE: While ABE protects data at rest on the user's device, DBSC focuses on making stolen session cookies useless if exfiltrated and used on another device. They are two distinct but synergistic layers of defense against session hijacking. An attacker bypassing ABE to get cookies might still find those cookies unusable elsewhere if they are DBSC-protected.




#### 6. Key Insights from Google's ABE Design Document & Chromium Source Code



Insights from Google's design documents and the Chromium source code (elevator.h, elevator.cc, caller_validation.h, caller_validation.cc) provide a comprehensive understanding:

Original Intent vs. Implemented Reality (Path vs. Signature Validation): The initial proposal (Page 4 of the design doc) contemplated validating the digital signature of both the calling process and the IElevator service executable. However, an "Update (2024)" note clarifies that the project was descoped to use path validation for the initial implementation, primarily for simplicity, with the assessment that it offered "equivalent protection against a non-admin attacker" for the prevailing threat models at the time.
OSCrypt Module Modifications: The core components/os_crypt module within Chromium was slated to be augmented. Instead of making direct DPAPI calls, it would use new IPC mechanisms to communicate with the Elevation Service (Pages 2, 5). The design proposed that OSCrypt would iterate through a list of "key encryption delegates" - one for legacy DPAPI keys, another for ABE-protected keys via IPC - to find a delegate capable of decrypting a given key (Page 6).
Stateless Nature of the Service: The IElevator service, in its role for ABE, is designed as a largely stateless encrypt/decrypt primitive. It doesn't require its own persistent storage for ABE operations (Page 4).
Explicit Acknowledgment of Injection as a Bypass: Page 7 ("Weaknesses") of the design document candidly states: "An attacker could inject code into Chrome browser and call the IPC interface. It would be hard to defeat a determined attacker using this technique..." This project serves as a practical validation of this assessment.
Understanding the IElevator COM Interface and its Definition:
The IElevator interface is a standard Windows COM (Component Object Model) interface. Such interfaces define a contract between a service provider (like Chrome's Elevation Service) and a client (like Chrome's OSCrypt module, or in this project's case, the injected chrome_decrypt.dll).
This contract is formally specified using MIDL (Microsoft Interface Definition Language). An .idl file written in MIDL describes the methods, parameters, and data types. The MIDL compiler processes this .idl file to generate C/C++ header files (defining the interface structure for compilers) and a type library (.tlb) that describes the interface's binary layout. It also generates proxy/stub code that enables COM to transparently manage communication between the client and server, even if they are in different processes.
While this project's chrome_decrypt.dll contains a C++ stub for IElevator (using the MIDL_INTERFACE macro), this serves as a compile-time declaration of the interface's shape. The crucial elements for runtime interaction are the correct CLSID (to identify the COM component) and IID (to request the specific IElevator interface pointer) passed to CoCreateInstance.
The IElevator interface, as potentially defined by Chrome, would include methods like EncryptData and DecryptData. An illustrative C++ stub, similar to what's in chrome_decrypt.cpp, is:
```c
// Illustrative C++ MIDL_INTERFACE definition stub from chrome_decrypt.cpp
MIDL_INTERFACE("A949CB4E-C4F9-44C4-B213-6BF8AA9AC69C") 
IElevator : public IUnknown
{
public:
    // Method for Chrome's recovery mechanisms, not directly used for decryption by this tool.
    virtual HRESULT STDMETHODCALLTYPE RunRecoveryCRXElevated(
        const WCHAR *crx_path, const WCHAR *browser_appid, /* ...other params... */) = 0; 
    
    // Method used by Chrome to initially encrypt the app_bound_key.
    virtual HRESULT STDMETHODCALLTYPE EncryptData(
        ProtectionLevel protection_level, // Specifies the type of protection to apply
        const BSTR plaintext,
        BSTR *ciphertext,
        DWORD *last_error) = 0;
    
    // The key method utilized by this tool to decrypt the app_bound_key.
    virtual HRESULT STDMETHODCALLTYPE DecryptData(
        const BSTR ciphertext, // DPAPI-wrapped app_bound_key blob from Local State
        BSTR *plaintext,      // Output: raw 32-byte app_bound_key
        DWORD *last_error) = 0; // Propagates underlying errors (e.g., from DPAPI)
};
```

The EncryptData method, though not called by this decryption tool, would likely use an enum like ProtectionLevel to dictate the security measures applied during the encryption of the app_bound_key. This project includes such an enum in chrome_decrypt.cpp:
```c
// From elevation_service_idl.h (implicitly, via project's chrome_decrypt.cpp stub)
enum class ProtectionLevel // As used by IElevator
{
    PROTECTION_NONE = 0,
    PROTECTION_PATH_VALIDATION_OLD = 1, // An older path validation scheme
    PROTECTION_PATH_VALIDATION = 2,    // The ABE path validation relevant to this research
    PROTECTION_MAX = 3                 // Boundary for valid levels
};
```
By specifying ProtectionLevel::PROTECTION_PATH_VALIDATION during the EncryptData call, Chrome instructs the IElevator service to enforce the path validation check when creating the app_bound_encrypted_key. The DecryptData method, subsequently used by this tool, implicitly respects the protection level that was originally applied during encryption.
The IElevator::EncryptData method, when called by Chrome with ProtectionLevel::PROTECTION_PATH_VALIDATION, generates caller-specific validation_data (based on the normalized path of Chrome itself), prepends this to the actual app_bound_key, and then encrypts this combined payload twice with DPAPI (first user-context, then system-context).
The IElevator::DecryptData method reverses this: decrypts twice with DPAPI (first system-context, then user-context), extracts the validation_data and the app_bound_key, performs path validation using the extracted validation_data against the current caller, and returns the app_bound_key if valid. This project's tool correctly utilizes this returned key.
Path Normalization (MaybeTrimProcessPath in caller_validation.cc): A critical detail for ProtectionLevel::PROTECTION_PATH_VALIDATION is that the validation does not use the raw executable path. Instead, MaybeTrimProcessPath normalizes it by:
Removing the executable filename (e.g., chrome.exe).
Conditionally removing trailing directory components if they match "Temp", "Application", or a version string (e.g., 127.0.0.0).
Standardizing Program Files (x86) to Program Files. This ensures that different Chrome versions or temporary unpack locations within the same sanctioned base installation directory can still validate successfully.






































