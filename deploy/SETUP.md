# One-time setup

Everything after this goes through pull requests from the phone (CLAUDE.md §10).
All steps except the Tailscale policy editor work fine in a phone browser.

## 1. Apps

- On Łukasz's iPhone: **GitHub** (review and merge PRs), **Claude**, **Tailscale**.
- On the wife's iPhone: **Tailscale**.
- On the iPad: **Tailscale**, signed in with Łukasz's login (for now; see the note in `deploy/tailscale-acl.hujson`).

## 2. Tailscale (PC is more comfortable)

1. Create the tailnet. In **DNS**, enable MagicDNS and HTTPS Certificates.
2. Invite the wife's login.
3. **Access controls:** paste `deploy/tailscale-acl.hujson`, replace the example emails, save.
4. **Settings → Keys:** create a one-off, pre-approved **auth key** with tag `tag:server`. It is used once in step 3.
5. **Settings → OAuth clients:** create a client with scope `auth_keys` (write) and tag `tag:ci`. Keep the client ID and secret for step 4.

## 3. Hetzner Cloud (phone is fine)

1. Create an account and a project.
2. **Firewalls:** create a firewall with **no inbound rules**.
3. **Servers → Add server:**
   - Location in the EU (Falkenstein, Nuremberg or Helsinki).
   - Image: Ubuntu 24.04.
   - Type: shared vCPU, x86, 2 vCPU / 4 GB (CX23 at the time of writing).
   - Networking: public IPv4 + IPv6 (needed for outbound traffic only).
   - SSH keys: none.
   - Firewall: the one from step 2.
   - **Cloud config:** paste `deploy/cloud-init.yaml` with `TS_AUTHKEY_HERE` replaced by the key from 2.4.
4. After 3–5 minutes the server `srv` appears in the Tailscale admin console as `tag:server`.

## 4. GitHub (phone is fine)

Repo **Settings → Secrets and variables → Actions**, add:

- `TS_OAUTH_CLIENT_ID`
- `TS_OAUTH_SECRET`

No registry token is needed: the deploy job logs the server into GHCR with its own short-lived `GITHUB_TOKEN`.

## 5. Later (before real data)

- **KEK:** generated on the server in Milestone 1. Store a copy in the password manager; without it the backups cannot be decrypted.
- **Backups:** an account with a provider other than Hetzner (e.g. Backblaze B2, EU region) and a key limited to one bucket.
