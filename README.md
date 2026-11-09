# npm-swap-cert

Two small Python tools (standard library only) that connect [Cert Warden](https://www.certwarden.com/)
with [Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM):

| Tool | What it does |
|---|---|
| `certwarden-npm-push` | Downloads certificates from Cert Warden and uploads them to NPM as **custom certificates**. On a renewal it uploads over the same NPM certificate, so proxy hosts keep working. Meant to run daily from a systemd timer. |
| `npm-swap-cert` | One-off helper that moves many NPM hosts from one certificate to another in a single run (for example from an old wildcard certificate to the Cert Warden one), instead of editing each host by hand. |

Why: NPM keeps its certificates in its own database, so an external certificate manager cannot just drop
files into a folder. Pushing through NPM's API also lets the NPM host stay without internet access, because
only the Cert Warden host talks to the certificate authority.

Tested with NPM 2.15.1 and Cert Warden 0.30.

## Requirements

- Python 3.11 or newer (uses `tomllib`), no extra packages.
- A running Cert Warden with the certificates already issued, and an API key for each certificate and its
  private key (Cert Warden: edit the certificate / private key, API key).
- NPM reachable on its admin/API port from the machine that runs the script.

## Security notes

- `certwarden-npm-push` needs the **private keys** of your certificates and an NPM login. Keep the env file
  readable by root only (`chmod 600`), and run the tool on a host you trust.
- Give the NPM user for `certwarden-npm-push` only the *certificates: manage* permission. NPM shows a
  non-admin user only the items that user owns, so it cannot see or change your proxy hosts. That is
  intentional.
- `npm-swap-cert` needs an NPM **admin** login, because only an admin can see and edit all hosts. It asks for
  the login on every run and never stores or prints it.
- Passwords and keys are never printed, and the env file is parsed as plain text, not executed by a shell.

## certwarden-npm-push

### Setup

1. Copy the script and create the config directory:
   ```bash
   sudo install -m 750 certwarden-npm-push /usr/local/bin/certwarden-npm-push
   sudo mkdir -p /etc/certwarden-push /var/lib/certwarden-push
   sudo chmod 700 /var/lib/certwarden-push
   ```
2. Create `/etc/certwarden-push/targets.toml` from [`examples/targets.example.toml`](examples/targets.example.toml)
   (one `[[target]]` per certificate).
3. Create `/etc/certwarden-push/env` from [`examples/env.example`](examples/env.example), fill in the NPM
   login and the Cert Warden API keys for each target, then `sudo chmod 600 /etc/certwarden-push/env`.
4. Test without changing anything, then run it for real:
   ```bash
   sudo certwarden-npm-push --dry-run
   sudo certwarden-npm-push
   ```
   The first run creates a custom certificate in NPM per target. Assign it once to your proxy hosts (SSL tab,
   or with `npm-swap-cert` below). Later runs say `unchanged` until Cert Warden has renewed the certificate.
5. Install the timer:
   ```bash
   sudo cp examples/certwarden-npm-push.service examples/certwarden-npm-push.timer /etc/systemd/system/
   sudo systemctl daemon-reload && sudo systemctl enable --now certwarden-npm-push.timer
   journalctl -u certwarden-npm-push.service     # logs of each run
   ```

### Options

| Option | Meaning |
|---|---|
| `--only NAME` | Only this target. |
| `--force` | Upload even if the certificate has not changed. |
| `--dry-run` | Show what would happen, change nothing. |
| `--config FILE`, `--env FILE`, `--state-dir DIR` | Override the default paths under `/etc/certwarden-push` and `/var/lib/certwarden-push`. |

Cert Warden renews a certificate when about a third of its lifetime is left, and the daily timer then delivers
it within about a day. Monitor the certificate your proxy actually serves (for example with Zabbix or Uptime
Kuma) to catch any failure in the chain.

## npm-swap-cert

Run it in an interactive terminal (it asks for the NPM admin e-mail and password). Dry run is the default.

```bash
# 1. list certificates: id, name, domains, expiry and how many hosts use each
./npm-swap-cert --npm main --url http://npm.example.lan:81 --list

# 2. dry run: which hosts would move from certificate 5 to certificate 32
./npm-swap-cert --npm main --url http://npm.example.lan:81 --from 5 --to 32

# 3. move ONE host, then check it in a browser
./npm-swap-cert --npm main --url http://npm.example.lan:81 --from 5 --to 32 --apply --limit 1

# 4. move the rest
./npm-swap-cert --npm main --url http://npm.example.lan:81 --from 5 --to 32 --apply

# undo a run with the revert file it printed
./npm-swap-cert --npm main --url http://npm.example.lan:81 --revert ./swap-main-<time>.json
```

- A host is only moved if **every** one of its domain names is covered by the new certificate. A wildcard
  covers exactly one label: `app.example.com` matches `*.example.com`, `a.b.example.com` does not. Hosts that
  do not match are listed as `SKIP` and left alone.
- Handles proxy, redirection and 404 hosts. Stream hosts are skipped.
- Only the certificate id of a host is changed; its other settings stay as they are. NPM reloads nginx for
  each changed host.
- `--npm` is just a label for the instance (it names the revert file). Without `--url` the script reads
  `NPM_<LABEL>_URL` from the env file. `--state-dir` sets where the revert file is written.
- Take a backup or snapshot of the NPM host first.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `NPM login failed (HTTP 400)` | NPM logs in with the user's **e-mail address**, not a username. Check the password and that 2FA is off for the script user. |
| `Cert Warden certificate download failed (HTTP 401/404)` | Wrong API key, or the certificate name in `targets.toml` does not match the name in Cert Warden. |
| `no private key found in Cert Warden response` | The private key API key is wrong, or the certificate has no key attached. |
| `npm-swap-cert` lists no hosts | You are logged in as a non-admin user, which only sees its own items. Use an NPM admin. |
| Hosts show up as `SKIP` | The new certificate does not cover all of the host's domain names. |
| A renewed certificate is not served | NPM may not have reloaded nginx yet. Check the served certificate's expiry, and reload or restart NPM. |

## How it works

`certwarden-npm-push` calls Cert Warden's download API (`/certwarden/api/v1/download/certificates/<name>` for
the full chain, and `/privatecerts/<name>` for the key), then NPM's API: log in at `/api/tokens`, create a
custom certificate (`provider: other`) if none with that name exists, and upload `certificate` and
`certificate_key` to `/api/nginx/certificates/<id>/upload`. A SHA-256 of the chain is kept per target so
unchanged certificates are skipped. `npm-swap-cert` lists hosts through `/api/nginx/*-hosts` and updates only
`certificate_id`.

## License

MIT, see [LICENSE](LICENSE).
