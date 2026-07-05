## PyVault
This tool mitigates the two most critical vulnerabilities of deploying compiled Python applications (e.g., via PyInstaller or Nuitka): 
- Source code extraction
- Memory dumping (RAM analysis)

By decoupling the secrets from the application logic and delegating the network layer to a hardened Rust binary, the token is never exposed to the Python runtime.


<img width="907" height="585" alt="Screenshot_1" src="https://github.com/user-attachments/assets/fe4a28cb-b466-4f13-b713-48a23d1431d8" />
