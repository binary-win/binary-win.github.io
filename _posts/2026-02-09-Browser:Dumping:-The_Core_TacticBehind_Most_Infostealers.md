# Browser Dumping: The Core Tactic Behind Most Infostealers

> **This blog is a collection of my own research from the internet, along with insights from other blogs and studies. While similar information can be found elsewhere, my main goal is to write everything in my own words and style. Please consider this as my personal notes and learning journal.**

#### 1. Introduction: The Evolution of Local Data Protection in Chrome
For years, Chromium-based browsers on Windows relied on the Data Protection API (DPAPI) to secure sensitive user data stored locally such as cookies, passwords, payment information, and the like. DPAPI binds data to the logged-in user's credentials, offering a solid baseline against offline attacks (e.g., a stolen hard drive) and unauthorized access by other users on the same machine. However, DPAPI's Achilles' heel has always been its permissiveness within the user's own session: any application running as the same user, with the same privilege level as Chrome, can invoke CryptUnprotectData and decrypt this data. This vulnerability has been a perennial favorite for infostealer malware.

To counter this, Google introduced App-Bound Encryption (ABE) in Chrome (publicly announced around version 127, July 2024). ABE is a significant architectural shift designed to dramatically raise the bar for attackers. Its core principle is to ensure that the primary decryption keys for sensitive Chrome data are only accessible to legitimate Chrome processes, thereby mitigating trivial data theft by same-user, same-privilege malware.

1.1. Foundational Concepts of ABE
**Primary Goal**: Prevent an attacker operating with the same privilege level as Chrome from trivially calling DPAPI to decrypt sensitive data.
**Acknowledged Limitations (Non-Goals)**: ABE does not aim to prevent attackers with higher privileges (Administrator, SYSTEM, kernel drivers) or those who can successfully inject code into Chrome. The official Google design documents explicitly recognize code injection as a potent bypass vector, a technique this project leverages for legitimate research and data recovery demonstrations.
**Underlying Mechanism**: ABE introduces an intermediary COM service (part of Chrome's Elevation Service) that acts as a gatekeeper for the DPAPI-unwrapping of a critical session key. This service verifies the "app identity" of the caller.
**Initial Identity Verification Method**: The first iteration relies on path validation of the calling executable. While digital signature validation was considered, path validation was chosen for the initial rollout to "descope the complexity" (as noted in a 2024 update to Google's design document), deemed sufficient against the immediate threat model.

Google's conceptual diagram provides a clear overview:
https://camo.githubusercontent.com/820b98efc081b49f087006dd3726eeaf51a7b55597c5e6c1dcc3136f0a44353b/68747470733a2f2f626c6f676765722e676f6f676c6575736572636f6e74656e742e636f6d2f696d672f622f523239765a32786c2f41567658734567706a6b41436c58325676677349684c69327a416d765277564d5045654a7155687169734b48494b786266474177683870382d56374978637435617a7a6e5f6a59664a596f32697a576e4763626b56683363616262434c56515151734a414a616776465043464a7378344d696261754a716e4c56796d51596468644747633533713377534a536554505136767978586f734a2d744a524b7561616f56375f4a5f45324b4239676c535a316d334e53457745426a2d6475657667524f486c4d2f73313431362f53637265656e73686f74253230323032342d30372d3236253230322e31352e3036253230504d2e706e67






































