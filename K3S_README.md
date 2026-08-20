# Vaultwarden on K3s — Configuration Guide

This covers the pieces missing from the shared manifest: `DOMAIN`, the Ingress
rule for path-based routing, generating a proper `ADMIN_TOKEN`, opening the
admin page, how SMTP is wired up, and adding Microsoft SSO.

---

## 1. Why `DOMAIN` is required

Vaultwarden uses `DOMAIN` to:
- Generate absolute URLs in the web vault (icons, WebAuthn/passkey origin checks, links in emails)
- Build the CSP (Content-Security-Policy) header correctly
- Build the SSO callback URL (`<DOMAIN>/identity/connect/oidc-signin`)

Without it, WebAuthn/passkeys, email links, and SSO will misbehave even if the
UI loads.

**Because you're using path-based routing** (`my.domain.com/vaultwarden`
instead of a dedicated subdomain), `DOMAIN` must include the path — not just
the host.

Add this to the Deployment's `env:` block:

```yaml
- name: DOMAIN
  value: "https://my.domain.com/vaultwarden"
```

No trailing slash on `DOMAIN` itself.

> **Recommendation:** if it's an option for you, prefer a subdomain
> (`vault.my.domain.com`) over a subpath. Vaultwarden's web vault assets and
> some browser-extension/mobile-app flows are known to be finicky under a
> subpath (see [vaultwarden#2288](https://github.com/dani-garcia/vaultwarden/issues/2288)).
> It works, but subdomain routing has fewer edge cases. If your team's
> ingress convention is path-based for every app, the steps below make it
> work correctly.

---

## 2. Ingress rule for path-based routing

Two things matter for Vaultwarden specifically:

1. **Do not rewrite the path.** Vaultwarden needs to see the real
   `/vaultwarden` prefix in the request (it's aware of its own base path via
   `DOMAIN`), so skip `rewrite-target` annotations that strip the prefix —
   unlike a typical app, rewriting will break it here.
2. **Use a trailing slash on the path**, and keep `pathType: Prefix`.

Example (assuming K3s's default Traefik-replaced-with or an nginx ingress
controller is installed — adjust `ingressClassName` if you're on Traefik):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vaultwarden
  namespace: vault
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "525m"   # allow attachment uploads
    # do NOT add a rewrite-target annotation
spec:
  ingressClassName: nginx   # or "traefik" if that's what your cluster uses
  tls:
    - hosts:
        - my.domain.com
      secretName: my-domain-tls
  rules:
    - host: my.domain.com
      http:
        paths:
          - path: /vaultwarden/
            pathType: Prefix
            backend:
              service:
                name: vaultwarden
                port:
                  number: 80
```

If your cluster is on **Traefik** (K3s default) instead of nginx, the
equivalent is a plain `Ingress` with no rewrite middleware attached — same
YAML above works, just point `ingressClassName: traefik` and don't attach any
`StripPrefix`/`ReplacePath` middleware to this route.

Then restart the pod after adding `DOMAIN` so it picks up the new base path:

```bash
kubectl -n vault rollout restart deployment vaultwarden
```

---

## 3. Service type

Change the Service from `NodePort` to `ClusterIP` — Ingress talks to it over
the cluster network, so exposing a NodePort too just opens an extra,
unauthenticated path to the pod on every node:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vaultwarden
  namespace: vault
spec:
  type: ClusterIP
  selector:
    app: vaultwarden
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

---

## 4. Generating `ADMIN_TOKEN` properly

A plain-text `ADMIN_TOKEN` (like the one in the shared manifest) works but is
logged by Vaultwarden as insecure on every startup. The recommended approach
is an Argon2id-hashed token, generated with Vaultwarden's own `hash`
subcommand:

```bash
docker run --rm -it vaultwarden/server:1.36.0 /vaultwarden hash
```

It'll prompt you for a password twice and print something like:

```
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$<salt>$<hash>'
```

Take just the `$argon2id$...` string (drop the surrounding single quotes) and
put it directly in the Kubernetes Secret's `stringData` — no escaping needed,
since `stringData` isn't shell-interpolated the way a `.env` file is:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vaultwarden-secret
  namespace: vault
type: Opaque
stringData:
  ADMIN_TOKEN: "$argon2id$v=19$m=65540,t=3,p=4$<salt>$<hash>"
  SMTP_USERNAME: "no-reply@xyz.in"
  SMTP_PASSWORD: "<smtp-password>"
```

**Important:** the string you actually log in with on the `/admin` page is
the plaintext password you typed when generating the hash — not the hash
itself.

---

## 5. Opening the admin page

Once `DOMAIN` and the Ingress are set up:

```
https://my.domain.com/vaultwarden/admin
```

Log in with the plaintext password you used to generate the Argon2 hash. From
there you can manage users, invitations, diagnostics, and see whether SMTP
and the config are being read correctly.

---

## 6. How SMTP works here (short version)

Vaultwarden uses SMTP purely for transactional email — account
verification, invites, password-hint requests, new-device login
notifications, and 2FA email codes. It does **not** need SMTP to function day
to day; without it, users just won't get those emails (and you'd have to
handle invites manually via the admin page).

The env vars in your manifest map directly to an SMTP client config:

| Var | Purpose |
|---|---|
| `SMTP_HOST` / `SMTP_PORT` | Mail server address |
| `SMTP_SECURITY` | `starttls` (587), `force_tls` (465), or `off` |
| `SMTP_FROM` / `SMTP_FROM_NAME` | "From" address shown to recipients |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | Auth credentials for the mail server |

For Office 365 (`smtp.office365.com:587`, STARTTLS) this is standard — just
make sure the mailbox used has SMTP AUTH enabled (Microsoft disables it by
default per-mailbox now) and, if you're on Entra ID with MFA/Conditional
Access, that you're using an app password or a dedicated shared-mailbox
account rather than a regular interactive-login account.

You can verify SMTP is working from the admin page's **Diagnostics** tab —
it has a "send test email" button.

---

## 7. Adding SSO with Microsoft (Entra ID / Azure AD)

Vaultwarden supports SSO via generic OpenID Connect, and Microsoft Entra ID
works as a standard OIDC provider — no special integration needed, just
configure it as a generic OIDC app.

### Step 1 — Register an app in Entra ID
1. Azure Portal → **Entra ID** → **App registrations** → **New registration**.
2. Set the redirect URI (type: Web) to:
   ```
   https://my.domain.com/vaultwarden/identity/connect/oidc-signin
   ```
   (this is auto-derived from `DOMAIN`, so it must match exactly, subpath included)
3. Under **Certificates & secrets**, create a client secret and note it down.
4. Under **API permissions**, add `openid`, `email`, `profile` (Microsoft
   Graph, delegated).
5. Note the **Application (client) ID** and the **Directory (tenant) ID**.

### Step 2 — Add SSO env vars to the Deployment/Secret

```yaml
- name: SSO_ENABLED
  value: "true"
- name: SSO_ONLY
  value: "false"     # keep master-password login available as a fallback while testing
- name: SSO_CLIENT_ID
  value: "<application-client-id>"
- name: SSO_CLIENT_SECRET
  valueFrom:
    secretKeyRef:
      name: vaultwarden-secret
      key: SSO_CLIENT_SECRET
- name: SSO_AUTHORITY
  value: "https://login.microsoftonline.com/<tenant-id>/v2.0"
- name: SSO_SCOPES
  value: "email profile"
```

Add `SSO_CLIENT_SECRET` alongside `ADMIN_TOKEN` in the `vaultwarden-secret`
Secret's `stringData`.

### Step 3 — Test
Restart the deployment, then load `https://my.domain.com/vaultwarden`. A new
**"Enterprise Single Sign-On"** button should appear on the login screen.
Users still set/enter a Vaultwarden master password on first SSO login —
SSO handles authentication into the account, the master password still
handles vault encryption (that's by design, not a bug).

Full provider-specific notes: [Vaultwarden SSO wiki – Microsoft Entra ID](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-SSO-support-using-OpenId-Connect#microsoft-entra-id)

---

## Summary of manifest changes

- [ ] Add `DOMAIN=https://my.domain.com/vaultwarden` to the Deployment env
- [ ] Change Service `type: NodePort` → `type: ClusterIP`, remove `nodePort`
- [ ] Add an Ingress resource for `/vaultwarden/` with **no path rewrite**
- [ ] Replace plaintext `ADMIN_TOKEN` with an Argon2id hash
- [ ] Rotate the `ADMIN_TOKEN` and `SMTP_PASSWORD` values that were shared in plaintext
- [ ] (Optional) Add `SSO_*` env vars for Microsoft Entra ID login
