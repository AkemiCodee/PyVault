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


## Getting Started: Implementing PyVault
To secure your application's API tokens, follow this implementation guide.

### Prerequisites
Before you begin, ensure you have the following files:
- `builder.exe`: The configuration utility.
- `vault_native.pyd`: The native Rust-based interceptor module.
- `Your Compiled App`: Your Python script must be compiled into an .exe (using Nuitka or PyInstaller). PyVault does not support standard .py script files.

### Workflow
1. Navigate to the directory containing your compiled application in your Command Prompt (CMD) or PowerShell.
2. Execute the builder.exe utility to generate the cryptographically bound vault.dat file.

```bash
  builder.exe --exe "C:\Path\To\Your\Application.exe" --token "YOUR_SECRET_TOKEN"
```

### Important Guidelines
- Full Paths: The `<PATH>` argument must be the absolute path to your file (e.g., C:\Users\Name\Desktop\MyApp.exe).
- Immutable Binary: Once `vault.dat` is generated, it is strictly bound to the SHA-256 hash of your .exe. If you modify your Python application code or recompile it, the vault.dat will become invalid and must be regenerated.
- The Token: The token is the sensitive, private part of your API request (e.g., your Discord Webhook ID/Secret, API Key, or Bearer Token).

### Example: Discord Webhook Integration

Full URL: 

`https://discord.com/api/webhooks/1523306830682046659/tvq0JnDvZM0xv6uZVXJAP2ta3f5fdguHbG5FaoE-IWT9S80yLtbicCBUlHnbJ6Qy_wRf`

YOUR_SECRET_TOKEN:

`1523306830682046659/tvq0JnDvZM0xv6uZVXJAP2ta3f5fdguHbG5FaoE-IWT9S80yLtbicCBUlHnbJ6Qy_wRf`

### Implementation Example
In your Python code, replace your sensitive token with the `VAULT_TOKEN_PLACEHOLDER` string. The native module will automatically intercept the request and inject the decrypted secret at the socket level.

```python
import sys
import requests
import vault_native

# Initialize the interceptor
try:
    vault_native.init_interceptor()
    print("Vault-Interceptor loaded successfully.")
except Exception as e:
    print(f"Failed to load Vault-Interceptor: {e}")
    sys.exit(1)

# Usage
data = {"message": "Authenticated via PyVault."}

# Use the placeholder instead of the real token
webhook_url = "https://discord.com/api/webhooks/VAULT_TOKEN_PLACEHOLDER"

response = requests.post(webhook_url, json=data)

if response.status_code in [200, 204]:
    print("Message sent successfully.")
else:
    print(f"Request failed with status {response.status_code}: {response.text}")
```

### Versatility

PyVault is not limited to Discord Webhooks. You can use the `VAULT_TOKEN_PLACEHOLDER` in any HTTP request, header, or API path. Wherever your application requires a hardcoded secret, simply use the placeholder, and the native interceptor will securely handle the injection and memory sanitization for you.

