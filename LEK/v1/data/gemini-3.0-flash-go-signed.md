This security review targets the provided Go source code for the `core` package.

### Executive Summary
The codebase acts as a bridge between a Forgejo instance and remote "agent" machines. The most critical vulnerabilities are found in the **job dispatch mechanism**, which uses unsanitized input to construct shell commands and distributes long-lived administrative secrets to remote targets.

---

### 1. Remote Command Injection (Shell Injection)
- **Severity**: Critical (CVSS 9.8)
- **Location**: `pkg/jobrunner/handlers/dispatch.go` -> `scpTicket()` and `ticketExists()`
- **Issue**: The code constructs shell commands by interpolating `remotePath` and `ticketName` directly into a string passed to `ssh`. These variables are derived from `RepoOwner` and `RepoName`, which are fetched from the Forgejo API.
- **Code Snippet**:
  ```go
  // In scpTicket
  fmt.Sprintf("cat > %s", remotePath) 
  // In ticketExists
  fmt.Sprintf("test -f %s/%s || ...", agent.QueueDir, ticketName)
  ```
- **Impact**: If an attacker creates a repository named `;rm -rf /;`, the dispatcher will execute that command on the remote agent machine. This allows for full Remote Code Execution (RCE) on all agent nodes.
- **Recommendation**: 
  1. Avoid shell interpolation. Use `scp` directly for file transfers instead of `ssh cat > file`.
  2. For the `test -f` check, pass arguments separately or use a strictly sanitized whitelist for repository names.
  3. Use `shellescape` logic if you must use a shell, but preferably use the `crypto/ssh` package instead of `os/exec` to manage remote files.

### 2. Secret Exposure (Token Proliferation)
- **Severity**: High (CVSS 8.1)
- **Location**: `pkg/jobrunner/handlers/dispatch.go` -> `DispatchTicket` struct
- **Issue**: The `ForgeToken` (an administrative API token) is included in the `DispatchTicket` JSON and written to disk on every remote agent machine.
- **Impact**: The "Blast Radius" of a single compromised agent machine is the entire Forgejo instance. If one agent is breached, the attacker gains the administrative token used by the central coordinator, allowing them to delete repos, steal code, or modify any project.
- **Recommendation**: 
  1. **Token Scoping**: Use Forgejo "Scoped Tokens" or "App Passwords" limited only to the specific repository the agent is working on.
  2. **Short-lived Credentials**: Implement a mechanism to provide agents with temporary OIDC tokens or short-lived SSH keys rather than the master API token.

### 3. Man-in-the-Middle (MitM) via Insecure SSH Options
- **Severity**: High (CVSS 7.5)
- **Location**: `pkg/jobrunner/handlers/dispatch.go` -> `scpTicket()` and `ticketExists()`
- **Issue**: The SSH commands use `-o StrictHostKeyChecking=accept-new`.
- **Impact**: This flag automatically trusts the host key of any machine on the first connection. An attacker on the network can intercept the connection (ARP spoofing, DNS hijacking), present a malicious key, and the dispatcher will proceed to send the `ForgeToken` (see Finding #2) to the attacker.
- **Recommendation**: Pre-populate the `known_hosts` file on the coordinator machine with the public keys of all authorized agent machines. Remove `accept-new` in production environments.

### 4. Path Traversal in Audit Logging
- **Severity**: Medium (CVSS 6.3)
- **Location**: `pkg/jobrunner/journal.go` -> `Append()`
- **Issue**: The journaler creates directories using `signal.RepoOwner` and `signal.RepoName` directly from the API.
- **Code Snippet**:
  ```go
  dir := filepath.Join(j.baseDir, signal.RepoOwner, signal.RepoName)
  if err := os.MkdirAll(dir, 0o755); err != nil ...
  ```
- **Impact**: An attacker can create a repository named `../../etc/cron.d/` (if Forgejo allows such names) or similar, potentially allowing the coordinator to write or overwrite files outside the intended `baseDir`.
- **Recommendation**: Sanitize `RepoOwner` and `RepoName` before using them in file paths. Use a helper function to strip path separators or validate against a regex (e.g., `^[a-zA-Z0-9-_.]+$`).

### 5. Sensitive Data in URL Query Parameters
- **Severity**: Medium (CVSS 5.3)
- **Location**: `pkg/ratelimit/ratelimit.go` -> `CountTokens()`
- **Issue**: The Google API Key is passed as a URL query parameter.
- **Code Snippet**:
  ```go
  url := fmt.Sprintf(".../models/%s:countTokens?key=%s", model, apiKey)
  ```
- **Impact**: Query parameters are frequently logged by corporate proxies, web servers, and internal logging frameworks. This increases the risk of the API key leaking into log management systems (Splunk, ELK, etc.).
- **Recommendation**: Pass the API key in a request header (e.g., `X-Goog-Api-Key`) as supported by the Google Generative Language API.

### 6. Insecure Default Configuration
- **Severity**: Low (CVSS 3.5)
- **Location**: `pkg/forge/config.go` -> `DefaultURL`
- **Issue**: The default URL is `http://localhost:4000`.
- **Impact**: Using `http` by default encourages unencrypted communication of API tokens over the network if the user changes `localhost` to a remote IP without switching to `https`.
- **Recommendation**: Change the default to `https://` or add a warning log if an `http://` URL is used for a remote host.

### 7. Information Leakage in Error Logs
- **Severity**: Low (CVSS 3.1)
- **Location**: `pkg/forge/client.go` and `pkg/jobrunner/poller.go`
- **Issue**: Errors from the Forgejo SDK and SSH commands are logged directly.
- **Impact**: If the SDK returns verbose errors containing database snippets or internal paths, these are written to the application logs. While useful for debugging, they can assist an attacker in footprinting the environment.
- **Recommendation**: Use a structured logger to separate "Internal Error" (detailed) from "User Error" (generic).

---

### Summary of Ratings

| Finding | Severity | OWASP Category |
| :--- | :--- | :--- |
| **1. Remote Command Injection** | **Critical** | A03:2021-Injection |
| **2. Secret Exposure** | **High** | A02:2021-Cryptographic Failures |
| **3. SSH MitM** | **High** | A07:2021-Identification and Auth Failures |
| **4. Path Traversal** | **Medium** | A01:2021-Broken Access Control |
| **5. API Key in URL** | **Medium** | A09:2021-Security Logging and Monitoring Failures |
| **6. Insecure Default** | **Low** | A05:2021-Security Misconfiguration |