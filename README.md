## PyVault
This tool mitigates the two most critical vulnerabilities of deploying compiled Python applications (e.g., via PyInstaller or Nuitka): 
- Source code extraction
- Memory dumping (RAM analysis)

By decoupling the secrets from the application logic and delegating the network layer to a hardened Rust binary, the token is never exposed to the Python runtime.

<img width="907" height="585" alt="Screenshot_1" src="https://github.com/user-attachments/assets/fe4a28cb-b466-4f13-b713-48a23d1431d8" />

**How it works:** 
The Provisioning Workflow completely decouples your sensitive API tokens from the application's source code. Instead of hardcoding secrets, the `builder.exe` utility analyzes your compiled Python application and generates a unique cryptographic fingerprint. This footprint is used to derive a secure, hardware-agnostic key that encrypts your token into an external `vault.dat` file. Because the vault is cryptographically bound to the exact hash of your executable, any tampering with the application immediately invalidates the decryption process.

<img width="547" height="556" alt="Screenshot_2" src="https://github.com/user-attachments/assets/2e7e6c8c-f466-408f-a94c-2cb08c6cf923" />

**How it works:** 
At runtime, your Python application never touches or processes the real API token, eliminating the risk of accidental exposure in logs or memory dumps. The app simply dispatches standard HTTP requests containing a dummy placeholder. Meanwhile, the `vault_native.pyd` module establishes a secure, low-level native socket hook. It silently monitors outbound traffic and intercepts the placeholder request seamlessly in the background, ensuring the Python runtime remains entirely isolated from the sensitive data.

<img width="1014" height="248" alt="Screenshot_3" src="https://github.com/user-attachments/assets/3a60717a-6d16-4607-ac74-8eb03fcc8fb9" />

- Environment Verification - 'vault_native.pyd' performs silent heuristics to detect active debuggers or unauthorized tampering attempts.
- Integrity Validation - The module verifies the cryptographic footprint of the host executable to authorize vault access.
- Vault Access - The encrypted payload is securely extracted from 'vault.dat'.
- In-Memory Decryption - The secret token is decrypted strictly within isolated, volatile memory.
- Dynamic Payload Injection - The placeholder within the outbound network stream is seamlessly swapped with the real token before transmission.
- Memory Sanitization - The unencrypted token is instantly zeroized (overwritten with null bytes) to thwart RAM dumping and memory analysis.

<img width="580" height="669" alt="Screenshot_4" src="https://github.com/user-attachments/assets/c5cefb5c-f865-468d-aea6-25d214aa7447" />

**How it works:** 
After the native module successfully validates the environment and dynamically injects the decrypted token into the outbound byte stream, the payload is transmitted via a secure TLS/HTTPS connection. Simultaneously, the decrypted token is instantly zeroized (wiped) from the system's RAM to thwart memory analysis. The target API receives a perfectly formatted, authenticated request and returns the standard response back to the Python app. The entire process is completely transparent to the host application, requiring zero changes to your existing networking logic.
