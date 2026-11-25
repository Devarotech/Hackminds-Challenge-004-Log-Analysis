# HACKMINDS Challenge 004 — Investigation Report
Participant: NAJEEM OMOKUNMI ATNANDA
Date submitted: 2026-11-25

## 1. Executive summary
On November 17, 2026, synthetic authentication and web server logs recorded suspicious activity originating from IP address `203.0.113.77`. Between 09:12:01 and 09:13:47 UTC, the source generated five failed SSH login attempts against `lab-host` before successfully authenticating as the `analyst` user at 09:14:22 UTC. While the SSH session remained active until 09:27:30 UTC, the same IP address conducted web directory enumeration and accessed administrative endpoints (`/admin`, `/admin/users`, and `/reports`). While the logs confirm suspicious authentication and web access patterns, they do not independently prove human attribution or confirmed data exfiltration.

## 2. Evidence examined and tools/commands used
All analysis was performed in a read-only manner on Kali Linux to preserve original evidence integrity:
- `unzip` — Extracted `HACKMINDS_Challenge_004_Participant_Package.zip`.
- `ls` / `cd` — Navigated directory structures (`evidence/`, `report/`).
- `cat` — Inspected `START_HERE.txt`, `evidence/README.txt`, `evidence/auth.log`, `evidence/web_access.log`, and `report/REPORT_TEMPLATE.md`.

## 3. Key findings
- **Suspicious source IP:** `203.0.113.77`
- **Successful account:** `analyst` (UID 1001)
- **Failed attempts before success:** 5 total failed attempts (1 for invalid user `admin`, 4 for user `analyst`)
- **Successful login time (UTC):** Nov 17 09:14:22 UTC
- **Session closure time (UTC):** Nov 17 09:27:30 UTC (total session duration: 13 minutes, 8 seconds)
- **Web paths and status codes:**
  - `GET /admin` — HTTP `200` (1842 bytes)
  - `GET /backup` — HTTP `403` (213 bytes)
  - `GET /.env` — HTTP `404` (196 bytes)
  - `GET /old-admin` — HTTP `404` (196 bytes)
  - `GET /admin/users` — HTTP `200` (2410 bytes)
  - `GET /reports` — HTTP `200` (1733 bytes)

## 4. Reconstructed timeline

| Time (UTC) | Event | Evidence source | Interpretation |
|---|---|---|---|
| 09:12:01 | Failed SSH password for invalid user `admin` from `203.0.113.77` (port 51001) | `auth.log` | Initial reconnaissance/brute-force attempt against a common administrative account. |
| 09:12:18 | Failed SSH password for `analyst` from `203.0.113.77` (port 51002) | `auth.log` | Password guessing shifted to valid account `analyst`. |
| 09:12:39 | Failed SSH password for `analyst` from `203.0.113.77` (port 51003) | `auth.log` | Continued failed authentication attempt. |
| 09:13:04 | Failed SSH password for `analyst` from `203.0.113.77` (port 51004) | `auth.log` | Continued failed authentication attempt. |
| 09:13:47 | Failed SSH password for `analyst` from `203.0.113.77` (port 51005) | `auth.log` | Fourth consecutive failed login for `analyst` (fifth overall). |
| 09:14:22 | Accepted SSH password and opened session for `analyst` (UID 1001) from `203.0.113.77` (port 51006) | `auth.log` | Successful authentication following multiple failures; potential credential compromise. |
| 09:15:02 | `GET /admin` returned HTTP `200` (1842 bytes) to `203.0.113.77` | `web_access.log` | Web enumeration begins from the same IP; `/admin` endpoint exists and served content. |
| 09:16:11 | `GET /backup` returned HTTP `403` (213 bytes) to `203.0.113.77` | `web_access.log` | Attempt to access `/backup` was blocked by server permissions. |
| 09:17:08 | `GET /.env` returned HTTP `404` (196 bytes) to `203.0.113.77` | `web_access.log` | Attempt to probe for exposed environment configuration file; resource not found. |
| 09:18:41 | `GET /old-admin` returned HTTP `404` (196 bytes) to `203.0.113.77` | `web_access.log` | Attempt to locate legacy administrative interface; resource not found. |
| 09:20:03 | `GET /admin/users` returned HTTP `200` (2410 bytes) to `203.0.113.77` | `web_access.log` | Request to user management sub-path succeeded at the HTTP protocol level. |
| 09:22:19 | `GET /reports` returned HTTP `200` (1733 bytes) to `203.0.113.77` | `web_access.log` | Request to reporting endpoint succeeded at the HTTP protocol level. |
| 09:27:30 | SSH session closed for user `analyst` | `auth.log` | Termination of the 13-minute SSH session. |

## 5. HTTP response interpretation
- **HTTP 200 (OK) — `/admin`, `/admin/users`, `/reports`:** Indicates the web server successfully processed the `GET` requests and returned a response payload. In this investigation, it confirms these paths exist and responded, but it does **not** prove the requester was unauthorized or that sensitive data was actually exposed or exfiltrated (for example, the response could be a login redirect page or benign content).
- **HTTP 403 (Forbidden) — `/backup`:** Indicates the server understood the request and the resource/route may exist, but access was denied due to insufficient permissions or server access controls.
- **HTTP 404 (Not Found) — `/.env`, `/old-admin`:** Indicates the server could not find anything matching the requested URIs, showing that those specific sensitive/legacy paths were absent at the requested locations.

## 6. CTF flag and derivation
- **Flag:** `HACKMINDS{203_0_113_77_analyst_091422}`
- **Derivation:**
  - **Source IP (dots replaced with underscores):** `203_0_113_77`
  - **Account name:** `analyst`
  - **Successful login time (HHMMSS UTC):** `091422` (from `09:14:22`)
  - **Combined format:** `HACKMINDS{203_0_113_77_analyst_091422}`

## 7. Security assessment
The activity observed from `203.0.113.77` exhibits high-confidence indicators of suspicious reconnaissance and potential unauthorized access:
1. **Credential Attack Pattern:** Five rapid SSH failures followed immediately by a successful login indicates likely password guessing or credential stuffing against an SSH service that still permits password-based authentication.
2. **Coordinated Web Probing:** Within 40 seconds of establishing an SSH session, the same IP address (`User-Agent: Mozilla/5.0 (X11; Linux x86_64) HACKMINDS-Lab`) probed for high-value web targets (`/admin`, `/backup`, `/.env`, `/old-admin`). The tight temporal correlation suggests a single source conducting multi-service enumeration.

## 8. Limitations — state at least three
1. **No human attribution:** An IP address identifies a network interface or proxy, not a specific individual, nor does it conclusively prove a single operator executed both the SSH and HTTP events.
2. **Unconfirmed credential acquisition method:** While a successful login after failures is suspicious, `auth.log` does not prove *how* the `analyst` password was obtained (e.g., brute-force vs. legitimate user mistyping vs. leaked credentials).
3. **No proof of data exposure or exfiltration:** HTTP `200` status codes only confirm successful HTTP responses; without packet captures, response bodies, or application logs, there is no proof sensitive data was viewed or stolen.
4. **Lack of post-login host telemetry:** Because `auth.log` only records session open/close events, it does not show what commands (if any) were executed inside the SSH shell during the 13-minute session.

## 9. Defensive recommendations
1. **Disable SSH Password Authentication:** Enforce SSH key-based authentication (`PasswordAuthentication no` in `/etc/ssh/sshd_config`) to eliminate brute-force and password-guessing risks.
2. **Implement Brute-Force Protections:** Deploy rate-limiting or intrusion prevention tools such as `fail2ban` to automatically block IPs after repeated failed login attempts.
3. **Enforce Multi-Factor Authentication (MFA):** Require MFA for both SSH access and administrative web endpoints (`/admin`, `/admin/users`, `/reports`).
4. **Harden Web Access Controls & Logging:** Restrict administrative endpoints to trusted internal networks or VPNs, ensure `/backup` and hidden files remain strictly inaccessible, and enable detailed application-level audit logging to capture authenticated user actions and response payloads.
5. **Credential Rotation:** Reset the `analyst` account password immediately and audit the host for unauthorized persistence mechanisms.

## 10. Conclusion
Correlation of `auth.log` and `web_access.log` identified suspicious SSH authentication activity and targeted web enumeration from `203.0.113.77` on November 17, 2026, resulting in a 13-minute SSH session under the `analyst` account and HTTP `200` responses on three administrative paths. While evidentiary limitations prevent attributing the actions to a specific individual or confirming data exfiltration, the findings warrant immediate credential rotation, SSH hardening, and access control reviews.
