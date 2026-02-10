This security review identifies several critical and high-severity vulnerabilities in the provided Go codebase, primarily centered around **Command Injection**, **Credential Exposure**, and **Path Traversal**.

---

### 1. Remote Command Injection (SSH/Shell)
- **Severity**: **Critical** (CVSS 9.8)
- **Location**: `pkg/jobrunner/handlers/dispatch.go` (Functions: `scpTicket` and `ticketExists`)
- **Issue**: The application uses `fmt.Sprintf` to construct shell commands that are executed on remote machines via SSH. The variables `signal.RepoOwner`, `signal.RepoName`, and `remotePath` are interpolated directly into the command string.
    ```go
    // In scpTicket
    fmt.Sprintf("cat > %s", remotePath) 
    // In ticketExists
    fmt.Sprintf("test -f %s/%s || ...", agent.QueueDir, ticketName)
    ```
    If a repository is named `; rm -rf / ;`, the remote shell will execute the injected command. Since repository names in Forgejo can contain various characters (and "Owner" names are also strings), this is a direct path to Remote Code Execution (RCE) on the agent machines.
- **Recommendation**: 
    1.  **Avoid Shell Wrapping**: Instead of `cat > %s`, use the `scp` protocol or a library like `github.com/pkg/sftp` to transfer files without invoking a shell.
    2.  **Sanitize Inputs**: If shell commands must be used, use a whitelist of allowed characters for `RepoOwner` and `RepoName`.
    3.  **Use Positional Arguments**: Never interpolate variables into a string intended for a shell.
- **Impact**: Complete compromise of remote agent machines. An attacker could pivot from a malicious repository name to full control of the infrastructure.

---

### 2. Sensitive Secret Exposure (Token Leakage)
- **Severity**: **High** (CVSS 8.1)
- **Location**: `pkg/jobrunner/handlers/dispatch.go` (Struct: `DispatchTicket`)
- **Issue**: The `ForgeToken` (a high-privilege API token) is included in the `DispatchTicket` struct, marshaled to JSON, and sent via SCP to remote agent machines where it is stored in plain text on the filesystem (`ticket-*.json`).
    ```go
    ticket := DispatchTicket{
        // ...
        ForgeToken:   h.token, // Plaintext token sent to remote host
    }
    ```
- **Recommendation**: 
    1.  **Short-lived Tokens**: Use scoped, short-lived tokens (e.g., Forgejo App Tokens with limited repo access) instead of the main user token.
    2.  **Agent-Side Auth**: Have the agent authenticate itself via its own credentials rather than passing the runner's master token.
    3.  **In-Memory Only**: If the token must be passed, ensure it is never written to disk on the remote side (e.g., pipe it directly to the agent process).
- **Impact**: If any agent machine is compromised, the attacker gains the master `FORGE_TOKEN`, allowing them to access all repositories, delete code, or modify organization settings.

---

### 3. Path Traversal in Audit Logging
- **Severity**: **High** (CVSS 7.5)
- **Location**: `pkg/jobrunner/journal.go` (Function: `Append`)
- **Issue**: The journal system creates directories and files using `signal.RepoOwner` and `signal.RepoName` without validation.
    ```go
    dir := filepath.Join(j.baseDir, signal.RepoOwner, signal.RepoName)
    if err := os.MkdirAll(dir, 0o755); err != nil { ... }
    path := filepath.Join(dir, date+".jsonl")
    ```
    An attacker could create a repository named `../../etc/` or `../../home/user/.ssh/`. While the `.jsonl` extension is appended, `MkdirAll` and `OpenFile` could be used to create directories or overwrite files in unauthorized locations.
- **Recommendation**: 
    1.  **Sanitize Paths**: Use `filepath.Base()` on the owner and repo names or validate them against a strict alphanumeric regex.
    2.  **Path Scrubbing**: Ensure the resulting path is a subdirectory of `baseDir` using `strings.HasPrefix`.
- **Impact**: Local file inclusion/overwrite on the runner machine. Could lead to Denial of Service or privilege escalation.

---

### 4. Man-in-the-Middle (MitM) via SSH Trust-on-First-Use
- **Severity**: **Medium** (CVSS 5.3)
- **Location**: `pkg/jobrunner/handlers/dispatch.go` (Function: `scpTicket`)
- **Issue**: The SSH command uses `-o StrictHostKeyChecking=accept-new`.
    ```go
    "-o", "StrictHostKeyChecking=accept-new",
    ```
    While safer than `no`, it automatically trusts any *new* host key. In an automated pipeline, if an attacker intercepts the first connection to a new agent, they can spoof the agent machine.
- **Recommendation**: Pre-populate the `known_hosts` file on the runner machine during the provisioning of agent machines. Use `StrictHostKeyChecking=yes`.
- **Impact**: Potential interception of `DispatchTicket` (containing the `ForgeToken`) by an attacker positioned between the runner and the agent.

---

### 5. Potential SSRF / API Key Leakage
- **Severity**: **Medium** (CVSS 5.3)
- **Location**: `pkg/ratelimit/ratelimit.go` (Function: `CountTokens`)
- **Issue**: The `model` parameter is interpolated directly into a URL sent to Google's APIs.
    ```go
    url := fmt.Sprintf("https://generativelanguage.googleapis.com/v1beta/models/%s:countTokens?key=%s", model, apiKey)
    ```
    If the `model` string is influenced by external configuration or issue metadata, an attacker could use path traversal (e.g., `model = "../other/api"`) to redirect the request or cause the `apiKey` to be sent to a different endpoint.
- **Recommendation**: Use `url.PathEscape` for the model name or validate it against a list of known-good model identifiers.
- **Impact**: Leakage of Google AI API keys to unauthorized endpoints or internal network scanning (SSRF).

---

### 6. Insecure Default Communication
- **Severity**: **Low** (CVSS 3.5)
- **Location**: `pkg/forge/config.go`
- **Issue**: `DefaultURL` is set to `http://localhost:4000`. 
- **Recommendation**: Change the default to `https://`. Even for local development, encouraging `http` increases the risk of tokens being sent over unencrypted channels in production if the user forgets to set the URL.
- **Impact**: Cleartext transmission of API tokens if the user relies on defaults.

---

### Summary Table

| Finding | Severity | OWASP Category | Impact |
| :--- | :--- | :--- | :--- |
| **SSH Command Injection** | Critical | A03:2021-Injection | Remote Code Execution |
| **ForgeToken Exposure** | High | A04:2021-Insecure Design | Full API Account Takeover |
| **Journal Path Traversal** | High | A01:2021-Broken Access Control | Arbitrary File Write |
| **SSH MitM Risk** | Medium | A02:2021-Cryptographic Failures | Credential Theft |
| **SSRF in Token Count** | Medium | A10:2021-SSRF | API Key Leakage |