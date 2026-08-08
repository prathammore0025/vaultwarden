# vaultwarden

# Vaultwarden Self-Hosted Password Manager

## 1. Overview

This document describes the complete deployment, configuration, administration, and end-user usage of a self-hosted Vaultwarden password manager.

### Environment

| Component               | Configuration               |
| ----------------------- | --------------------------- |
| OS                      | Ubuntu Linux                |
| Container Runtime       | Docker                      |
| Container Orchestration | Docker Compose              |
| Vaultwarden             | `vaultwarden/server:1.36.0` |
| Public Domain           | `vault.lannister.buzz`      |
| DNS / Edge              | Cloudflare                  |
| Public HTTPS            | Cloudflare                  |
| Tunnel                  | Cloudflare Tunnel           |
| Local Vaultwarden Port  | `127.0.0.1:8222`            |
| Email Provider          | Microsoft 365 SMTP          |
| SMTP Server             | `smtp.office365.com`        |
| SMTP Port               | `587`                       |
| SMTP Security           | STARTTLS                    |
| Organization            | Lannister                      |

### Architecture

```text
                         Internet
                            |
                            |
                   https://vault.lannister.buzz
                            |
                            v
                     +-------------+
                     |  Cloudflare |
                     | DNS + HTTPS |
                     +-------------+
                            |
                    Cloudflare Tunnel
                            |
                            v
                  +--------------------+
                  | Ubuntu Server      |
                  | cloudflared        |
                  +--------------------+
                            |
                            v
                  127.0.0.1:8222
                            |
                            v
                  +--------------------+
                  | Vaultwarden Docker  |
                  | Container           |
                  | Port 80             |
                  +--------------------+
                            |
                            v
                       ./vw-data
```

The Vaultwarden container is **not directly exposed to the Internet**.

Cloudflare Tunnel provides the public HTTPS endpoint and forwards traffic to the local Vaultwarden service.

Cloudflare Tunnel can route public hostnames to local services without requiring the application server to have an exposed public IP or inbound port forwarding.

---

# 2. Prerequisites

Before starting, ensure you have:

* Ubuntu server
* Root/sudo access
* Docker
* Docker Compose
* Cloudflare account
* Domain managed by Cloudflare
* A Cloudflare Tunnel
* SMTP account
* A backup location

Example:

```text
Domain:
lannister.buzz

Vaultwarden:
vault.lannister.buzz

SMTP:
smtp.office365.com:587
```

---

# 3. Install Docker

Update Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
```

Install Docker:

```bash
sudo apt install -y docker.io docker-compose-plugin
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
docker --version
docker compose version
```

Optional: allow the current user to run Docker without sudo:

```bash
sudo usermod -aG docker $USER
```

Log out and log back in after running this command.

---

# 4. Create Vaultwarden Directory

Create the application directory:

```bash
mkdir -p ~/software/vaultwarden
cd ~/software/vaultwarden
```

Create the persistent data directory:

```bash
mkdir -p vw-data
```

The important directory is:

```text
~/software/vaultwarden/vw-data
```

This contains persistent Vaultwarden data.

## IMPORTANT

Do not delete `vw-data`.

Do not recreate the container with an empty volume unless you intentionally want to start a new Vaultwarden installation.

---

# 5. Docker Compose Configuration

Create:

```bash
nano docker-compose.yaml
```

Use:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:1.36.0
    container_name: vaultwarden
    restart: unless-stopped

    environment:
      SIGNUPS_ALLOWED: "false"
      INVITATIONS_ALLOWED: "true"
      LOG_LEVEL: "info"
      DOMAIN: "https://vault.lannister.buzz"

      # SMTP
      SMTP_HOST: "smtp.office365.com"
      SMTP_PORT: "587"
      SMTP_SECURITY: "starttls"
      SMTP_FROM: "no-reply@YOURDOMAIN.in"
      SMTP_FROM_NAME: "Vaultwarden"
      SMTP_USERNAME: "no-reply@YOURDOMAIN.in"
      SMTP_PASSWORD: "YOUR_SMTP_PASSWORD"

    volumes:
      - ./vw-data:/data

    ports:
      - "127.0.0.1:8222:80"
```

Replace:

```text
YOURDOMAIN.in
```

with your actual Microsoft 365 email domain.

Replace:

```text
YOUR_SMTP_PASSWORD
```

with the SMTP password.

### Security recommendation

Do not commit `docker-compose.yaml` containing the SMTP password to Git.

Do not paste the SMTP password into tickets, Slack, documentation, or GitHub.

For production, consider using Docker secrets or another secret-management mechanism.

---

# 6. Start Vaultwarden

From:

```bash
cd ~/software/vaultwarden
```

Pull the image:

```bash
docker compose pull
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Expected:

```text
vaultwarden    Up
```

Check logs:

```bash
docker logs vaultwarden --tail 50
```

Expected:

```text
Rocket has launched from http://0.0.0.0:80
```

---

# 7. Test Vaultwarden Locally

Run:

```bash
curl -I http://127.0.0.1:8222
```

Expected:

```text
HTTP/1.1 200 OK
server: Rocket
```

This confirms:

```text
Ubuntu
  |
  +--> 127.0.0.1:8222
          |
          +--> Vaultwarden
```

If this does not return `200 OK`, troubleshoot Docker/Vaultwarden before troubleshooting Cloudflare.

---

# 8. Why Port 8222 Is Bound to Localhost

The Compose file uses:

```yaml
ports:
  - "127.0.0.1:8222:80"
```

This means:

```text
127.0.0.1:8222
```

is accessible only from the Ubuntu server itself.

It prevents direct Internet access to:

```text
SERVER_IP:8222
```

Cloudflare Tunnel is responsible for external access.

This is preferable to exposing:

```yaml
- "8222:80"
```

directly to the Internet.

---

# 9. Cloudflare Configuration

The domain:

```text
lannister.buzz
```

must be managed by Cloudflare.

The Vaultwarden hostname is:

```text
vault.lannister.buzz
```

The Cloudflare Tunnel connects:

```text
vault.lannister.buzz
        |
        v
Cloudflare Tunnel
        |
        v
http://127.0.0.1:8222
```

---

# 10. Install Cloudflared

Check architecture:

```bash
uname -m
```

For x86_64:

```bash
curl -L --output cloudflared.deb \
https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
```

Install:

```bash
sudo dpkg -i cloudflared.deb
```

Verify:

```bash
cloudflared --version
```

---

# 11. Cloudflare Login

Run:

```bash
cloudflared tunnel login
```

Cloudflare will provide a browser URL.

Open it and authenticate with the Cloudflare account that manages:

```text
lannister.buzz
```

Select the appropriate domain.

---

# 12. Create a Tunnel

Example:

```bash
cloudflared tunnel create vaultwarden
```

List tunnels:

```bash
cloudflared tunnel list
```

Example:

```text
NAME         UUID
vaultwarden  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Keep the tunnel UUID private enough for operational security, and never expose its credential JSON file.

---

# 13. Cloudflare Tunnel Configuration

The production systemd service uses:

```text
/etc/cloudflared/config.yml
```

Create/edit:

```bash
sudo nano /etc/cloudflared/config.yml
```

Example:

```yaml
tunnel: YOUR-TUNNEL-UUID

credentials-file: /root/.cloudflared/YOUR-TUNNEL-UUID.json

ingress:

  - hostname: gitea.lannister.buzz
    service: http://localhost:3009

  - hostname: backstage.lannister.buzz
    service: http://localhost:30070

  - hostname: argo.lannister.buzz
    service: http://localhost:31737

  - hostname: vault.lannister.buzz
    service: http://localhost:8222

  - service: http_status:404
```

### Important

The Vaultwarden rule must appear **before**:

```yaml
- service: http_status:404
```

The final 404 rule is a catch-all.

Cloudflare's documentation also recommends checking that the tunnel's origin service URL and port exactly match the locally running service.

---

# 14. Validate Cloudflare Configuration

Run:

```bash
sudo cloudflared tunnel ingress validate
```

Expected:

```text
Validating rules from /etc/cloudflared/config.yml
OK
```

Check the hostname rule:

```bash
sudo cloudflared tunnel ingress rule https://vault.lannister.buzz
```

It should match:

```text
vault.lannister.buzz
```

and:

```text
http://localhost:8222
```

It must not match the final:

```text
http_status:404
```

---

# 15. Configure Cloudflare DNS

Route the hostname to the tunnel:

```bash
cloudflared tunnel route dns YOUR-TUNNEL-UUID vault.lannister.buzz
```

Example:

```bash
cloudflared tunnel route dns YOUR-TUNNEL-UUID vault
```

Verify the DNS record in:

```text
Cloudflare
    >
DNS
    >
Records
```

The hostname should be:

```text
vault.lannister.buzz
```

and should point to the Cloudflare Tunnel.

If Cloudflare reports that an A/AAAA/CNAME record already exists, remove the conflicting record before creating the Tunnel route.

---

# 16. Install Cloudflared as a Service

Install:

```bash
sudo cloudflared service install
```

Enable:

```bash
sudo systemctl enable cloudflared
```

Start:

```bash
sudo systemctl start cloudflared
```

Check:

```bash
sudo systemctl status cloudflared
```

Expected:

```text
Active: active (running)
```

---

# 17. Verify Cloudflare Tunnel

Check logs:

```bash
sudo journalctl -u cloudflared -f
```

Look for:

```text
Registered tunnel connection
```

Multiple registered connections are normal.

Cloudflare Tunnel should establish outbound connections from the Ubuntu server to Cloudflare.

---

# 18. Test Public Vaultwarden

From Ubuntu:

```bash
curl -I https://vault.lannister.buzz
```

Expected:

```text
HTTP/2 200
```

or another valid Vaultwarden response such as a redirect.

Open in a browser:

```text
https://vault.lannister.buzz
```

Vaultwarden should load.

---

# 19. Why HTTPS Is Required

Initially, accessing:

```text
http://SERVER_IP:8222
```

produced:

```text
You are not using a secure context which is required for the Subtle Crypto API to work.
You need to enable HTTPS!
```

This happens because the Vaultwarden web client uses browser cryptography APIs that require a secure context.

The final architecture solves this:

```text
Browser
   |
   | HTTPS
   v
https://vault.lannister.buzz
   |
   v
Cloudflare
   |
   | Tunnel
   v
Vaultwarden
```

---

# 20. HTTPS and Let's Encrypt

This deployment does not require Certbot or a Let's Encrypt certificate on the Ubuntu server.

Cloudflare provides the public HTTPS endpoint:

```text
https://vault.lannister.buzz
```

Cloudflare Tunnel then forwards the request to:

```text
http://127.0.0.1:8222
```

Therefore:

```text
Internet
   |
   | HTTPS
   v
Cloudflare
   |
   | Tunnel
   v
HTTP localhost
   |
   v
Vaultwarden
```

This is different from a traditional:

```text
Internet
   |
   | HTTPS / Let's Encrypt
   v
Nginx
   |
   v
Vaultwarden
```

Both architectures are possible, but Cloudflare Tunnel avoids the need to expose ports 80/443 from the Ubuntu host.

---

# 21. Initial Vaultwarden Account

There is no default:

```text
username
password
```

The first user must create their own Vaultwarden account.

Initially, temporarily enable:

```yaml
SIGNUPS_ALLOWED: "true"
```

Restart:

```bash
docker compose up -d
```

Open:

```text
https://vault.lannister.buzz
```

Click:

```text
Create Account
```

Create:

```text
Email
Name
Master Password
```

---

# 22. Disable Public Signup

After creating the administrator/owner account, change:

```yaml
SIGNUPS_ALLOWED: "false"
```

Keep:

```yaml
INVITATIONS_ALLOWED: "true"
```

Restart:

```bash
docker compose up -d
```

Recommended production configuration:

```yaml
SIGNUPS_ALLOWED: "false"
INVITATIONS_ALLOWED: "true"
```

This means:

```text
Public registration     Disabled
Organization invites   Enabled
```

---

# 23. Master Password

Vaultwarden does not provide an administrator with a user's original master password.

Users should never share their master password with administrators.

Example:

```text
Prathmesh
    |
    +--> Own master password

Amit
    |
    +--> Own master password

Rahul
    |
    +--> Own master password
```

The Vaultwarden administrator does not need to know these passwords.

---

# 24. Master Password Recovery

A forgotten master password should not be treated like a normal website password reset.

For enterprise use, configure and test Vaultwarden's organization account-recovery functionality before relying on it.

Account recovery requires the relevant organization policy/enrollment to be configured, and SMTP/email functionality is important for the recovery workflow.

### Recommended recovery structure

```text
Lannister
 |
 +-- Owner 1
 |
 +-- Owner 2
 |
 +-- Admins
 |
 +-- Users
```

Do not make a single person the only Owner.

If the only Owner loses access, recovery becomes significantly more complicated.

---

# 25. Microsoft 365 SMTP Configuration

Vaultwarden can use Microsoft 365 SMTP for:

* Organization invitations
* Account-related emails
* Email verification
* Email-based authentication features
* Recovery-related email workflows

Vaultwarden supports SMTP settings including:

```text
SMTP_HOST
SMTP_PORT
SMTP_SECURITY
SMTP_FROM
SMTP_USERNAME
SMTP_PASSWORD
```

The current Vaultwarden configuration supports STARTTLS for SMTP, with port 587 as the normal STARTTLS port.

Configuration:

```yaml
SMTP_HOST: "smtp.office365.com"
SMTP_PORT: "587"
SMTP_SECURITY: "starttls"

SMTP_FROM: "no-reply@YOURDOMAIN.in"
SMTP_FROM_NAME: "Vaultwarden"

SMTP_USERNAME: "no-reply@YOURDOMAIN.in"
SMTP_PASSWORD: "YOUR_SMTP_PASSWORD"
```

---

# 26. SMTP Security

Do not store the SMTP password in:

```text
GitHub
GitLab
Documentation
Slack
Jira
Tickets
README files
```

Use:

```text
YOUR_SMTP_PASSWORD
```

as a placeholder in documentation.

The actual secret should be stored securely.

If an SMTP password is accidentally exposed, immediately rotate it.

---

# 27. Test Microsoft 365 SMTP Connectivity

From Ubuntu:

```bash
nc -vz smtp.office365.com 587
```

Expected:

```text
Connection to smtp.office365.com 587 port [tcp/submission] succeeded
```

If this fails, investigate:

* Firewall
* Network restrictions
* Proxy
* Microsoft 365 restrictions

If the network connection works but authentication fails, check Microsoft 365 SMTP AUTH configuration.

---

# 28. Test Vaultwarden SMTP

Open:

```text
https://vault.lannister.buzz/admin
```

Authenticate using the Vaultwarden admin token.

Use the SMTP test functionality available in the Admin interface.

Send a test email to an address you control.

Then check:

```bash
docker logs vaultwarden --tail 100
```

Look for SMTP errors.

Common error:

```text
535 Authentication unsuccessful
```

This usually indicates an SMTP authentication/configuration issue rather than a Vaultwarden networking issue.

---

# 29. Organization Structure

For enterprise use, create an organization.

Example:

```text
Lannister
```

Recommended structure:

```text
Lannister
|
+-- Lannister-Production
|
+-- Lannister-UAT
|
+-- Lannister-Development
|
+-- Lannister-Shared-Tools
```

---

# 30. Collections

Collections are used to organize shared credentials and control access.

Example:

```text
Lannister-Production
    |
    +-- AWS Production
    +-- MongoDB Production
    +-- Redis Production
    +-- Kubernetes Production
    +-- SSH Production
    +-- Cloudflare Production
    +-- Database Production
```

UAT:

```text
Lannister-UAT
    |
    +-- AWS UAT
    +-- MongoDB UAT
    +-- Kubernetes UAT
    +-- SSH UAT
```

Development:

```text
Lannister-Development
    |
    +-- AWS Development
    +-- MongoDB Development
    +-- Kubernetes Development
    +-- Jenkins Development
```

Shared tools:

```text
Lannister-Shared-Tools
    |
    +-- GitHub
    +-- Jenkins
    +-- Nexus
    +-- SonarQube
    +-- Cloudflare
```

---

# 31. Example Credential

Instead of storing:

```text
AWS Production
```

inside someone's personal vault, store it in:

```text
Organization:
Lannister

Collection:
Lannister-Production
```

Example:

```text
Name:
AWS Production

Username:
admin@example.com

Password:
***************

URL:
https://console.aws.amazon.com
```

Do not put secrets directly into documentation.

---

# 32. Add Organization Users

Go to:

```text
Organizations
    >
Lannister
    >
People
```

Invite the user's email address.

Example:

```text
amit@example.com
```

Vaultwarden will send an organization invitation using the configured SMTP server.

The expected workflow is:

```text
Admin
  |
  | Invite user
  v
Vaultwarden
  |
  | SMTP
  v
Microsoft 365
  |
  v
User Email
  |
  v
User accepts invitation
  |
  v
User joins Lannister
```

---

# 33. User Account vs Organization

These are different concepts.

A user has:

```text
Personal Vault
```

and can also belong to:

```text
Lannister Organization
```

Example:

```text
Amit
|
+-- Personal Vault
|     +-- Personal credentials
|
+-- Lannister Organization
      +-- Shared company credentials
```

A user's personal vault should not be used for company-owned credentials.

---

# 34. Organization Roles

Use the least-privilege model.

Recommended:

```text
Owner
    |
    +-- Very limited number of trusted people

Admin/Manager
    |
    +-- Day-to-day organization administration

User
    |
    +-- Normal employee
```

Do not give every employee Owner/Admin permissions.

---

# 35. Lannister Permission Model

Example:

| User             | Production | UAT        | Development |
| ---------------- | ---------- | ---------- | ----------- |
| Owner            | Manage     | Manage     | Manage      |
| Senior DevOps    | Manage     | Manage     | Manage      |
| DevOps Engineer  | Read/Write | Read/Write | Read/Write  |
| Developer        | No Access  | Read/Write | Read/Write  |
| QA               | No Access  | Read/Write | Read        |
| Junior Developer | No Access  | No Access  | Read/Write  |

The exact permission options available in the client/UI should be verified against the Vaultwarden version being deployed.

---

# 36. Recommended Enterprise Organization

```text
Lannister
|
+-- Owners
|     +-- Primary Owner
|     +-- Backup Owner
|
+-- DevOps
|     |
|     +-- Production
|     +-- UAT
|     +-- Development
|     +-- Shared Tools
|
+-- Developers
|     |
|     +-- UAT
|     +-- Development
|
+-- QA
      |
      +-- UAT
```

---

# 37. Groups

Vaultwarden has group support, but the current configuration describes organization groups as a beta feature and explicitly warns that known issues may exist.

If using groups:

```text
DevOps
Developers
QA
Managers
```

Then map groups to appropriate collections.

Example:

```text
DevOps
    |
    +-- Production
    +-- UAT
    +-- Development
    +-- Shared Tools
```

```text
Developers
    |
    +-- UAT
    +-- Development
```

```text
QA
    |
    +-- UAT
```

Before relying on groups for critical production access, test the exact behavior in your Vaultwarden version.

---

# 38. Production Credential Strategy

Do not create one giant collection containing everything.

Bad:

```text
Lannister-All-Credentials
```

Better:

```text
Lannister-Production
Lannister-UAT
Lannister-Development
Lannister-Shared-Tools
```

This makes access control easier.

---

# 39. Environment Separation

Production credentials should be separated from non-production credentials.

Recommended:

```text
PROD
UAT
DEV
```

Example:

```text
AWS-PROD
AWS-UAT
AWS-DEV
```

Do the same for:

```text
MongoDB
Redis
Kubernetes
SSH
Cloudflare
Jenkins
Nexus
Databases
```

---

# 40. Employee Onboarding

Recommended process:

```text
1. Create employee email account
2. Invite employee to Vaultwarden
3. Employee creates Vaultwarden account
4. Employee enables MFA
5. Enroll employee in account recovery
6. Add employee to appropriate organization role
7. Assign collections
8. Verify access
9. Document onboarding completion
```

---

# 41. Employee Offboarding

When an employee leaves:

```text
1. Disable corporate email/account
2. Remove user from Lannister organization
3. Remove collection access
4. Revoke organization membership
5. Rotate credentials they knew
6. Rotate shared production credentials if necessary
7. Review organization events
8. Confirm no company credentials remain in their personal vault
```

For highly sensitive production credentials, rotate the credential after employee departure if the employee had knowledge of the plaintext secret.

---

# 42. Credential Rotation

Vaultwarden stores credentials; it does not automatically rotate external passwords.

For example:

```text
AWS password
```

must still be changed in AWS.

Then update Vaultwarden.

Recommended process:

```text
External System
      |
      | Change password
      v
New Password
      |
      v
Vaultwarden
      |
      v
Update shared credential
```

---

# 43. MFA

MFA should be enabled for all privileged users.

Recommended:

```text
Owners
    -> MFA mandatory

Admins
    -> MFA mandatory

Users
    -> MFA strongly recommended
```

Prefer stronger MFA methods where available and appropriate, such as hardware security keys/WebAuthn.

---

# 44. Admin Panel

Vaultwarden provides an administrative interface:

```text
https://vault.lannister.buzz/admin
```

Protect the admin token carefully.

Do not share it with normal users.

Do not put it into Git.

Do not expose it in screenshots or documentation.

---

# 45. Backups

The most important Vaultwarden data is stored in:

```text
~/software/vaultwarden/vw-data
```

At minimum, back up this directory.

Example backup:

```bash
sudo tar -czf \
/backup/vaultwarden-$(date +%Y-%m-%d-%H%M).tar.gz \
~/software/vaultwarden/vw-data
```

Better production strategy:

```text
Vaultwarden
    |
    v
Local backup
    |
    v
Off-server backup
```

Keep multiple backup generations.

---

# 46. Backup Recommendation

Use a backup schedule such as:

```text
Daily
    |
    +-- Local backup

Daily/Weekly
    |
    +-- Off-server backup

Monthly
    |
    +-- Restore test
```

A backup that has never been restored/tested should not be considered reliable.

---

# 47. Restore Procedure

Stop Vaultwarden:

```bash
cd ~/software/vaultwarden
docker compose down
```

Move the existing data directory:

```bash
mv vw-data vw-data-old
```

Restore the backup:

```bash
mkdir vw-data
```

Extract:

```bash
tar -xzf /backup/YOUR_BACKUP.tar.gz
```

Ensure the directory structure is correct.

Then:

```bash
docker compose up -d
```

Check:

```bash
docker logs vaultwarden --tail 100
```

Test:

```bash
curl -I http://127.0.0.1:8222
```

Then:

```text
https://vault.lannister.buzz
```

---

# 48. Updating Vaultwarden

Before updating:

```bash
cd ~/software/vaultwarden
```

Create a backup first.

Then:

```bash
docker compose pull
```

Recreate:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Check logs:

```bash
docker logs vaultwarden --tail 100
```

Check the application:

```text
https://vault.lannister.buzz
```

Vaultwarden 1.36.0 was released with multiple security fixes, so staying current is important.

---

# 49. Docker Health Checks

Check containers:

```bash
docker ps
```

Check resource usage:

```bash
docker stats
```

Check Vaultwarden logs:

```bash
docker logs vaultwarden --tail 100
```

Follow logs:

```bash
docker logs -f vaultwarden
```

---

# 50. Cloudflare Health Checks

Check service:

```bash
sudo systemctl status cloudflared
```

Logs:

```bash
sudo journalctl -u cloudflared -n 100
```

Follow:

```bash
sudo journalctl -u cloudflared -f
```

Check configuration:

```bash
sudo cloudflared tunnel ingress validate
```

Check hostname routing:

```bash
sudo cloudflared tunnel ingress rule https://vault.lannister.buzz
```

---

# 51. Common Problem: HTTP 404

If:

```bash
curl -I https://vault.lannister.buzz
```

returns:

```text
HTTP/2 404
server: cloudflare
```

Check:

```bash
sudo cat /etc/cloudflared/config.yml
```

Make sure this exists:

```yaml
- hostname: vault.lannister.buzz
  service: http://localhost:8222
```

and appears before:

```yaml
- service: http_status:404
```

Then:

```bash
sudo cloudflared tunnel ingress validate
```

Restart:

```bash
sudo systemctl restart cloudflared
```

---

# 52. Common Problem: Vaultwarden Works Locally but Not Publicly

Test:

```bash
curl -I http://127.0.0.1:8222
```

If:

```text
HTTP/1.1 200 OK
```

but:

```bash
curl -I https://vault.lannister.buzz
```

returns an error, the problem is likely:

```text
Cloudflare DNS
or
Cloudflare Tunnel
```

not Vaultwarden.

---

# 53. Common Problem: Cloudflare Tunnel Is Running but 404 Appears

Check:

```bash
sudo systemctl cat cloudflared
```

Make sure it uses:

```text
/etc/cloudflared/config.yml
```

Then remember:

```text
~/.cloudflared/config.yml
```

and:

```text
/etc/cloudflared/config.yml
```

may be different files.

The systemd service must use the configuration you actually edited.

---

# 54. Common Problem: Wrong Tunnel

Check:

```bash
cloudflared tunnel list
```

Make sure the tunnel UUID in:

```text
/etc/cloudflared/config.yml
```

matches the tunnel being used by the systemd service.

Do not accidentally create multiple tunnels for the same hostname.

---

# 55. Common Problem: SMTP Invitation Not Received

Check:

```bash
docker logs vaultwarden --tail 100
```

Look for:

```text
SMTP
535
authentication
connection
timeout
```

Test network:

```bash
nc -vz smtp.office365.com 587
```

Check:

```text
SMTP_HOST
SMTP_PORT
SMTP_SECURITY
SMTP_FROM
SMTP_USERNAME
SMTP_PASSWORD
```

Microsoft 365 SMTP AUTH configuration must also permit the mailbox/application to authenticate.

---

# 56. Organization Invitation Flow

The normal process is:

```text
Admin
  |
  v
Lannister
  |
  v
People
  |
  v
Invite user@example.com
  |
  v
Microsoft 365 SMTP
  |
  v
Invitation email
  |
  v
User accepts
  |
  v
User creates/logs into account
  |
  v
User joins Lannister
  |
  v
Collection access assigned
```

---

# 57. End User Manual

## 57.1 Access Vaultwarden

Open:

```text
https://vault.lannister.buzz
```

---

## 57.2 Login

Enter:

```text
Email
Master Password
```

Complete MFA if enabled.

---

## 57.3 Personal Vault

Personal credentials should be stored in:

```text
My Vault
```

Example:

```text
Personal Gmail
Personal GitHub
Personal AWS
Personal Banking
```

Company credentials should normally be stored in the appropriate organization collection.

---

# 58. Access Lannister Credentials

After joining Lannister, the user will see the organization and collections they have permission to access.

Example:

```text
My Vault

Lannister
 |
 +-- Lannister-Development
 |
 +-- Lannister-UAT
```

If the user does not have access to:

```text
Lannister-Production
```

they should not see/access the production credentials in that collection.

---

# 59. Using a Shared Credential

Example:

```text
Lannister
  |
  +-- Lannister-UAT
       |
       +-- AWS UAT
```

A user opens:

```text
AWS UAT
```

and uses the stored username/password.

Depending on the client and permissions, the user can copy/autofill the credential without needing to know the password manually.

---

# 60. Adding a New Credential

Recommended process:

```text
1. Login
2. Open Lannister organization
3. Select appropriate collection
4. Create new item
5. Enter credential
6. Save
7. Verify authorized users can access it
```

Always choose the correct environment:

```text
Production
UAT
Development
```

---

# 61. Example: Lannister AWS Credentials

Do not create:

```text
AWS
```

if there are multiple environments.

Prefer:

```text
AWS Production
AWS UAT
AWS Development
```

Store each in the correct collection.

---

# 62. Example: SSH Credentials

Collection:

```text
Lannister-Production
```

Item:

```text
Lannister Production SSH
```

Fields:

```text
Username
Password
Hostname
Port
Notes
```

Sensitive private keys should also be protected carefully and access should be limited to authorized personnel.

---

# 63. Example Permission Model

```text
Lannister
|
+-- Production
|     |
|     +-- Senior DevOps
|     +-- DevOps Lead
|
+-- UAT
|     |
|     +-- DevOps
|     +-- QA
|     +-- Developers
|
+-- Development
|     |
|     +-- DevOps
|     +-- QA
|     +-- Developers
|
+-- Shared Tools
      |
      +-- DevOps
      +-- Admins
```

---

# 64. Security Rules

## Rule 1

Never share your Vaultwarden master password.

## Rule 2

Never share an organization admin password.

## Rule 3

Never commit SMTP credentials to Git.

## Rule 4

Never commit Vaultwarden database/data to a public repository.

## Rule 5

Do not give everyone Owner/Admin access.

## Rule 6

Separate Production, UAT, and Development credentials.

## Rule 7

Enable MFA.

## Rule 8

Keep backups.

## Rule 9

Test restores.

## Rule 10

Rotate credentials after sensitive employee offboarding.

---

# 65. Recommended Production Access Model

```text
                   Lannister
                     |
          +----------+----------+
          |                     |
       Owners                 Admins
          |                     |
    2 trusted users       Senior DevOps
          |                     |
          +----------+----------+
                     |
                  Users
                     |
       +-------------+-------------+
       |             |             |
     DevOps        QA          Developers
       |             |             |
       v             v             v
    PROD/UAT       UAT         DEV/UAT
```

---

# 66. Recommended Recovery Model

Do not depend on one person.

```text
Lannister
 |
 +-- Owner 1
 |
 +-- Owner 2
 |
 +-- Account Recovery
 |
 +-- MFA
 |
 +-- SMTP
 |
 +-- Backups
```

Test the recovery workflow using test accounts before relying on it for production users.

---

# 67. Recommended Monitoring

At minimum monitor:

```text
Vaultwarden container
Cloudflared service
Disk usage
Backup status
SMTP
Domain availability
```

Useful commands:

```bash
docker ps
docker logs vaultwarden --tail 100
sudo systemctl status cloudflared
df -h
```

---

# 68. Disk Monitoring

Check:

```bash
df -h
```

Check Vaultwarden data:

```bash
du -sh ~/software/vaultwarden/vw-data
```

Make sure the server does not run out of disk space.

---

# 69. Cloudflare DNS Architecture

The final public setup should look like:

```text
vault.lannister.buzz
        |
        v
Cloudflare DNS
        |
        v
Cloudflare Tunnel
        |
        v
Ubuntu
        |
        v
127.0.0.1:8222
        |
        v
Vaultwarden
```

There should be no requirement for:

```text
Public-IP:8222
```

---

# 70. Ports

## Ubuntu

Vaultwarden:

```text
127.0.0.1:8222
```

Cloudflared:

```text
Outbound connection to Cloudflare
```

No public inbound Vaultwarden port is required when using Cloudflare Tunnel.

---

# 71. Useful Commands Cheat Sheet

## Docker

```bash
cd ~/software/vaultwarden

docker compose ps

docker compose up -d

docker compose down

docker compose restart

docker compose pull

docker logs vaultwarden --tail 100

docker logs -f vaultwarden
```

## Vaultwarden local test

```bash
curl -I http://127.0.0.1:8222
```

## Cloudflare

```bash
sudo systemctl status cloudflared

sudo systemctl restart cloudflared

sudo journalctl -u cloudflared -f

sudo cloudflared tunnel ingress validate

sudo cloudflared tunnel ingress rule https://vault.lannister.buzz

cloudflared tunnel list
```

## Public test

```bash
curl -I https://vault.lannister.buzz
```

## Disk

```bash
df -h

du -sh ~/software/vaultwarden/vw-data
```

---

# 72. Final Production Checklist

Before using real company credentials:

```text
[ ] Vaultwarden running
[ ] Docker restart policy configured
[ ] Cloudflare Tunnel working
[ ] HTTPS working
[ ] vault.lannister.buzz working
[ ] Direct port 8222 not publicly exposed
[ ] SMTP configured
[ ] SMTP test successful
[ ] Organization created
[ ] Collections created
[ ] Owner accounts created
[ ] Backup Owner created
[ ] MFA enabled
[ ] Account recovery configured and tested
[ ] Employee invitation tested
[ ] Collection permissions tested
[ ] Backup configured
[ ] Backup restore tested
[ ] Disk monitoring configured
[ ] Update procedure documented
[ ] Offboarding procedure documented
[ ] SMTP secret protected
[ ] Admin token protected
```

---

# 73. Final Recommended Lannister Structure

```text
Vaultwarden
│
└── Lannister Organization
    │
    ├── Owners
    │   ├── Primary Owner
    │   └── Backup Owner
    │
    ├── Groups
    │   ├── DevOps
    │   ├── Developers
    │   ├── QA
    │   └── Managers
    │
    └── Collections
        │
        ├── Lannister-Production
        │   ├── AWS Production
        │   ├── MongoDB Production
        │   ├── Kubernetes Production
        │   ├── Redis Production
        │   ├── SSH Production
        │   └── Cloudflare Production
        │
        ├── Lannister-UAT
        │   ├── AWS UAT
        │   ├── MongoDB UAT
        │   ├── Kubernetes UAT
        │   └── SSH UAT
        │
        ├── Lannister-Development
        │   ├── AWS Development
        │   ├── MongoDB Development
        │   ├── Kubernetes Development
        │   └── Jenkins Development
        │
        └── Lannister-Shared-Tools
            ├── GitHub
            ├── Jenkins
            ├── Nexus
            ├── SonarQube
            └── Cloudflare
```

---

# 74. Conclusion

The final deployment is:

```text
                         USERS
                           |
                           v
              https://vault.lannister.buzz
                           |
                           v
                    +-------------+
                    | Cloudflare  |
                    +-------------+
                           |
                    Cloudflare Tunnel
                           |
                           v
                    Ubuntu Server
                           |
                   127.0.0.1:8222
                           |
                           v
                    Vaultwarden
                           |
                           v
                       vw-data
```

For enterprise usage:

```text
Lannister Organization
        |
        +-- Owners
        +-- Admins
        +-- Users
        +-- Groups
        +-- Collections
        +-- MFA
        +-- Account Recovery
        +-- SMTP
        +-- Backups
        +-- Monitoring
```

The most important operational principles are:

1. Keep public signup disabled.
2. Use organization invitations.
3. Separate Production/UAT/Development collections.
4. Apply least-privilege access.
5. Keep at least two trusted Owners.
6. Enable and test MFA.
7. Configure and test account recovery.
8. Keep encrypted backups.
9. Test restoration regularly.
10. Rotate credentials when access changes.
11. Never expose SMTP passwords or admin tokens.
12. Keep Vaultwarden updated.
