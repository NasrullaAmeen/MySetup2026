---
tags: [research, security, tfa, totp, sso, authelia, authentik, ssh]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# TFA & SSO for the Homelab

Research date: 2026-09-17.

## PVE-native TFA

- Three per-user factors: TOTP, WebAuthn, single-use Recovery Keys.
- Setup via GUI (My Settings -> Two Factor -> Add -> TOTP, scan QR in Bitwarden, verify 6-digit code) or API.
- CLI (verified syntax):

```
pveum user tfa list darko@pve
pveum user tfa unlock darko@pve        # break-glass: force-disable factors
pvesh create /access/tfa/darko@pve --type totp \
  --totp 'otpauth://totp/PVE:darko@pve?secret=<BASE32>&issuer=PVE' --value <6digit>
```

- Realm-level enforcement NOT recommended until every user is enrolled (lockout risk): `pveum realm modify pve --tfa type=totp`.
- Fallback if TOTP lost: recovery keys; else `pveum user tfa unlock`; else edit /etc/pve/priv/tfa.cfg (back it up first). Root console/SSH is final break-glass.
- [INFO] Enable TOTP on darko@pve now - also neutralizes the historical tfa-challenge auth bypass (accounts with no factor).

## Realms, tokens

- pve realm = internal users (no Linux shell); pam = Linux accounts; ldap/openid = external.
- API tokens are long-lived secrets that BYPASS TFA - treat each token as a second factor. Use `pveum user token add darko@pve <name> --privsep 1`, scoped ACLs, and rotate/revoke unused tokens (they never expire by default).

## SSO layer 2026

- Authelia (recommended start): light (~25-40MB), forward-auth companion for Traefik/Nginx/Caddy, OIDC-certified 2025, TOTP/WebAuthn/Duo, YAML-only config, no user UI, no SAML/LDAP-server.
- Authentik: full IdP (OIDC/SAML/LDAP-server/SCIM), flow builder, proxy provider; needs PostgreSQL (~150-500MB; no Redis since 2025.10). Choose if you outgrow Authelia.
- Keycloak: enterprise-grade, 1-4GB, overkill. Zitadel: Go, AGPL v3 (2025), aimed at multi-tenant SaaS.
- Verdict: Authelia for a small fleet; Authentik only if you need LDAP server, SAML, or user self-service.

## PVE behind external SSO

- Forward-auth proxy provider on 8006 gates the entrance but noVNC console needs Upgrade/Connection headers; you still get a second PVE login (double prompt). Direct 8006/pvesh access bypasses it entirely.
- OIDC realm is the cleaner native path: `pveum realm add authentik --type openid --issuer-url https://auth/application/o/<slug>/`.
- RECOMMENDATION: keep native PVE TFA as the source of truth; bind 8006 to LAN/VPN only. Not exposing pveproxy to the internet is the single biggest control.

## SSH

- Key-only already: PasswordAuthentication no, PermitRootLogin no (done on ProxDev).
- Homelab: key+passphrase is effectively 2FA; pam_google_authenticator overkill unless SSH is internet-exposed or multi-operator.
- If added: install libpam-google-authenticator, run google-authenticator per user, nullok during rollout, verify timedatectl sync. fail2ban optional (no passwords to brute-force).

## Rollout for this environment

1. Login as darko@pve -> My Settings -> TOTP (QR into Bitwarden) -> Recovery Keys (stored offline). Verify `pveum user tfa list darko@pve`.
2. Audit tokens: `pveum user token list darko@pve`; recreate with --privsep 1, revoke stale ones.
3. Containers: NOPASSWD sudoers for operator user inside LCXs.
4. Deploy Authelia in the reverse proxy for exposed services only; PVE stays native-TFA behind VPN.
5. Bitwarden as TOTP store is fine - also back up recovery keys offline.

## Sources

- https://pve.proxmox.com/pve-docs/chapter-pveum.html
- https://pve.proxmox.com/pve-docs/pveum.1.html
- https://pve.proxmox.com/wiki/OATH(TOTP)_Authentication
- https://integrations.goauthentik.io/hypervisors-orchestrators/proxmox-ve/
- https://selfhosting.sh/best/authentication/