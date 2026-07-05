## PyVault
This tool mitigates the two most critical vulnerabilities of deploying compiled Python applications (e.g., via PyInstaller or Nuitka): 
- Source code extraction
- Memory dumping (RAM analysis)

By decoupling the secrets from the application logic and delegating the network layer to a hardened Rust binary, the token is never exposed to the Python runtime.


<img width="907" height="585" alt="Screenshot_1" src="https://github.com/user-attachments/assets/fe4a28cb-b466-4f13-b713-48a23d1431d8" />
<img width="547" height="556" alt="Screenshot_2" src="https://github.com/user-attachments/assets/2e7e6c8c-f466-408f-a94c-2cb08c6cf923" />
<img width="1014" height="248" alt="Screenshot_3" src="https://github.com/user-attachments/assets/3a60717a-6d16-4607-ac74-8eb03fcc8fb9" />

- Environment Verification - 'vault_native.pyd' performs silent heuristics to detect active debuggers or unauthorized tampering attempts.
- Integrity Validation - The module verifies the cryptographic footprint of the host executable to authorize vault access.
- Vault Access - The encrypted payload is securely extracted from 'vault.dat'.
- In-Memory Decryption - The secret token is decrypted strictly within isolated, volatile memory.
- Dynamic Payload Injection - The placeholder within the outbound network stream is seamlessly swapped with the real token before transmission.
- Memory Sanitization - The unencrypted token is instantly zeroized (overwritten with null bytes) to thwart RAM dumping and memory analysis.

<img width="580" height="669" alt="Screenshot_4" src="https://github.com/user-attachments/assets/c5cefb5c-f865-468d-aea6-25d214aa7447" />
