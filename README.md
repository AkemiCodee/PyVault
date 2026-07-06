![License](https://img.shields.io/badge/License-Proprietary-red) ![OS](https://img.shields.io/badge/OS-Windows%2064--bit-blue) ![Python](https://img.shields.io/badge/Python-3.7%20--%203.13%2B-blue) ![Status](https://img.shields.io/badge/Status-Active-brightgreen) ![Built with Rust](https://img.shields.io/badge/Built%20with-Rust-orange?logo=rust)

## PyVault
This tool mitigates the two most critical vulnerabilities of deploying compiled Python applications (e.g., via PyInstaller or Nuitka): 
- **Source code extraction** 
- **Memory dumping (RAM analysis)**

By decoupling the secrets from the application logic and delegating the network layer to a hardened Rust binary, the token is never exposed to the Python runtime.

#### Get the latest ready-to-use version of the application here.
[![Download for Windows 64-bit](https://img.shields.io/badge/Download-Windows%2064--bit-success?style=for-the-badge&logo=windows)](https://github.com/AkemiCodee/PyVault/releases/latest)


## Table of Contents

- [Getting Started: Implementing PyVault](#getting-started-implementing-pyvault)
  - [Prerequisites](#prerequisites)
  - [Workflow](#workflow)
  - [Important Guidelines](#important-guidelines)
- [Python Integration](#python-integration)
  - [Example: Discord-Webhook](#example-discord-webhook)
  - [Example: Cloudflare Worker (Custom Auth Header)](#example-cloudflare-worker-custom-auth-header)
- [Important Note: Placeholder Limitations](#important-note-placeholder-limitations)
- [Compatibility](#compatibility)
- [Security & Trust Philosophy](#security--trust-philosophy)
- [Support & Contact](#support--contact)

<img width="907" height="585" alt="Screenshot_1" src="https://github.com/user-attachments/assets/fe4a28cb-b466-4f13-b713-48a23d1431d8" />

**How it works:** 
The Provisioning Workflow completely decouples your sensitive API tokens from the application's source code. Instead of hardcoding secrets, the `builder.exe` utility analyzes your compiled Python application and generates a unique cryptographic fingerprint. This footprint is used to derive a secure, hardware-agnostic key that encrypts your token into an external `vault.dat` file. Because the vault is cryptographically bound to the exact hash of your executable, any tampering with the application immediately invalidates the decryption process.

<img width="547" height="556" alt="Screenshot_2" src="https://github.com/user-attachments/assets/2e7e6c8c-f466-408f-a94c-2cb08c6cf923" />

**How it works:** 
At runtime, your Python application never touches or processes the real API token, eliminating the risk of accidental exposure in logs or memory dumps. The app simply dispatches standard HTTP requests containing a dummy placeholder. Meanwhile, the `vault_native.pyd` module establishes a secure, low-level native socket hook. It silently monitors outbound traffic and intercepts the placeholder request seamlessly in the background, ensuring the Python runtime remains entirely isolated from the sensitive data.

<img width="1014" height="248" alt="Screenshot_3" src="https://github.com/user-attachments/assets/3a60717a-6d16-4607-ac74-8eb03fcc8fb9" />

- **Environment Verification** - `vault_native.pyd` performs silent heuristics to detect active debuggers or unauthorized tampering attempts.
- **Integrity Validation** - The module verifies the cryptographic footprint of the host executable to authorize vault access.
- **Vault Access** - The encrypted payload is securely extracted from `vault.dat`.
- **In-Memory Decryption** - The secret token is decrypted strictly within isolated, volatile memory.
- **Dynamic Payload Injection** - The placeholder within the outbound network stream is seamlessly swapped with the real token before transmission.
- **Memory Sanitization** - The unencrypted token is instantly zeroized (overwritten with null bytes) to thwart RAM dumping and memory analysis.

<img width="580" height="669" alt="Screenshot_4" src="https://github.com/user-attachments/assets/c5cefb5c-f865-468d-aea6-25d214aa7447" />

**How it works:** 
After the native module successfully validates the environment and dynamically injects the decrypted token into the outbound byte stream, the payload is transmitted via a secure TLS/HTTPS connection. Simultaneously, the decrypted token is instantly zeroized (wiped) from the system's RAM to thwart memory analysis. The target API receives a perfectly formatted, authenticated request and returns the standard response back to the Python app. The entire process is completely transparent to the host application, requiring zero changes to your existing networking logic.


## Getting Started: Implementing PyVault
**To secure your application's API tokens, follow this implementation guide.**

### Prerequisites
Before you begin, ensure you have the following files:
- `builder.exe`: The configuration utility.
- `vault_native.pyd`: The native Rust-based interceptor module.
- `Your Compiled App`: Your Python script must be compiled into an .exe (using Nuitka or PyInstaller). **PyVault does not support standard .py script files.**

### Workflow
1. Navigate to the directory containing your compiled application in your Command Prompt (CMD) or PowerShell.
2. Execute the builder.exe utility to generate the cryptographically bound vault.dat file.

CMD
```bash
  builder.exe --exe "C:\Path\To\Your\Application.exe" --token "YOUR_SECRET_TOKEN"
```
PowerShell
```powershell
.\builder.exe --exe "C:\Path\To\Your\Application.exe" --token "YOUR_SECRET_TOKEN"
```

### Important Guidelines
- Full Paths: The `<PATH>` argument must be the **absolute path** to your file (e.g., C:\Users\Name\Desktop\MyApp.exe).
- Immutable Binary: Once `vault.dat` is generated, it is strictly bound to the SHA-256 hash of your .exe. **If you modify your Python application code or recompile it, the vault.dat will become invalid and must be regenerated.**
- The Token: The token is the sensitive, private part of your API request (e.g., your Discord Webhook ID/Secret, API Key, or Bearer Token).

## Python Integration

### Example: Discord-Webhook

Full URL: 

`https://discord.com/api/webhooks/1523306830682046659/tvq0JnDvZM0xv6uZVXJAP2ta3f5fdguHbG5FaoE-IWT9S80yLtbicCBUlHnbJ6Qy_wRf`

YOUR_SECRET_TOKEN:

`1523306830682046659/tvq0JnDvZM0xv6uZVXJAP2ta3f5fdguHbG5FaoE-IWT9S80yLtbicCBUlHnbJ6Qy_wRf`

#### Implementation Example
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

### Example: Cloudflare Worker (Custom Auth Header)
PyVault is highly versatile and works perfectly with custom HTTP headers. In this scenario, we protect a static authorization password required by a Cloudflare Worker.

Full Header:
```python
headers = {
    "X-Custom-Auth": "SECRET_PASSWORD",
    "Content-Type": "application/json"
}
```
Token:
`"SECRET_PASSWORD"`

#### Implementation Example
Instead of exposing the secret password in the request headers, we inject the `VAULT_TOKEN_PLACEHOLDER`. The Rust interceptor dynamically identifies the placeholder within the outgoing header payload and replaces it with the decrypted string before the request reaches the network.

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

# Usage - Cloudflare Worker configuration
url = "https://MyApp.YourName.workers.dev/"

# Use the placeholder inside the HTTP header
headers = {
    "X-Custom-Auth": "VAULT_TOKEN_PLACEHOLDER",
    "Content-Type": "application/json"
}

log_data = {
    "Status": "Online",
    "payload": "Secure log transmission via PyVault."
}

# The request is sent with the placeholder, PyVault handles the rest seamlessly
response = requests.post(url, json=log_data, headers=headers)

if response.status_code == 200:
    print("Log successfully sent to Cloudflare!")
else:
    print(f"Request failed with status {response.status_code}: {response.text}")
```

### Versatility

PyVault is not limited to Discord Webhooks / Cloudflare Worker. You can use the `VAULT_TOKEN_PLACEHOLDER` in any HTTP request, header, or API path. Wherever your application requires a hardcoded secret, simply use the placeholder, and the native interceptor will securely handle the injection and memory sanitization for you.

## Important Note: Placeholder Limitations
While PyVault is highly flexible, there is one technical limitation you must keep in mind regarding where you can safely place the `VAULT_TOKEN_PLACEHOLDER`.

Safe to use in:

- URLs (e.g., https://api.Example.com/VAULT_TOKEN_PLACEHOLDER)
- HTTP Headers (e.g., "Example": "Bearer VAULT_TOKEN_PLACEHOLDER")

Changing URLs or Headers at the socket level is perfectly safe and will not disrupt the HTTP protocol.

Do NOT use inside the HTTP Body (e.g., JSON payloads):

- If you attempt to inject the placeholder directly into the data payload, it will likely cause the target server to reject your request.

```python
# Do NOT put the placeholder inside the body!!
data = {
    "auth_secret": "VAULT_TOKEN_PLACEHOLDER" 
}
response = requests.post(url, json=data)
```

### Why does this happen?
High-level HTTP libraries (like Python's `requests`) automatically calculate the `Content-Length` header based on the exact size of your JSON payload before the data is sent.

Because `vault_native.pyd` intercepts and modifies the traffic at the lowest network (socket) level, it swaps the placeholder with your real token after Python has already calculated this length. If your real token has a different length than the placeholder string (23 characters), the `Content-Length` header will be incorrect. The receiving API will detect this mismatch and immediately drop the connection or return a `400 Bad Request` error.

## Compatibility

Thanks to Python's Stable ABI (abi3), the native interceptor is highly compatible and does not require recompilation for different Python environments.

- **Python Version:** Supports **Python 3.7 and all newer versions** (3.8, 3.9, 3.10, 3.11, 3.12, 3.13+).
- **OS/Architecture:** Windows 64-bit (x86_64) only.

## Security & Trust Philosophy

PyVault is closed-source to protect the integrity of its obfuscation and protection mechanisms. However, we value transparency and user safety.

- Local-Only Execution: PyVault processes everything locally. No data is ever sent to our servers.
- Privacy by Design: We do not have access to your tokens, and we do not collect any telemetry or user data.
- Dependency Transparency: PyVault relies on vetted, industry-standard Rust libraries (e.g., `ring` for encryption, `zeroize` for memory safety) to ensure cryptographic robustness.

#### Virustotal:
builder.exe (8097d77204975898de2671629d98eb8a71b4395ffe90b755195b3dc3ee85fee7):
https://www.virustotal.com/gui/file/8097d77204975898de2671629d98eb8a71b4395ffe90b755195b3dc3ee85fee7/detection
native_vault.pyd (79b3253eb1d2f41624240cdc58c654125ee70f659a9a9f20498fa9221d19d9ca):
https://www.virustotal.com/gui/file/79b3253eb1d2f41624240cdc58c654125ee70f659a9a9f20498fa9221d19d9ca?nocache=1

# Support & Contact
Questions, feedback, or found a bug? Feel free to reach out:
- GitHub: @AkemiCodee (Preferred for Issues & Bugs)
- Discord: @akeemi666_ (For quick questions)
