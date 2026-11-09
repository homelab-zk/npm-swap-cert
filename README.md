# npm-swap-cert

A small Python tool (standard library only) that moves many
[Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM) hosts from one certificate to another in a single
run, through NPM's API. Useful when you replace a certificate (for example an old wildcard certificate with one
from an external certificate manager) and do not want to edit every proxy host by hand in the UI.

Tested with NPM 2.15.1.

## Requirements

- Python 3, no extra packages.
- NPM reachable on its admin/API port from the machine you run the script on.
- An NPM **admin** login. Only an admin can see and edit all hosts; a non-admin user only sees its own items,
  so the script would find nothing to change. The script asks for the login on every run and never stores or
  prints it.

## Usage

Run it in an interactive terminal (it prompts for the NPM admin e-mail and password). Dry run is the default.

```bash
# 1. list certificates: id, name, domains, expiry and how many hosts use each
./npm-swap-cert --url http://npm.example.lan:81 --list

# 2. dry run: which hosts would move from certificate 5 to certificate 32
./npm-swap-cert --url http://npm.example.lan:81 --from 5 --to 32

# 3. move ONE host, then check it in a browser
./npm-swap-cert --url http://npm.example.lan:81 --from 5 --to 32 --apply --limit 1

# 4. move the rest
./npm-swap-cert --url http://npm.example.lan:81 --from 5 --to 32 --apply

# undo a run with the revert file it printed
./npm-swap-cert --url http://npm.example.lan:81 --revert ./swap-npm-<time>.json
```

The old and new certificate ids come from `--list`; NPM's web UI does not show them.

| Option | Meaning |
|---|---|
| `--url URL` | NPM base URL (required). |
| `--list` | List certificates and how many hosts use each. Changes nothing. |
| `--from ID --to ID` | Old and new certificate id. |
| `--apply` | Make the change. Without it the run is a dry run. |
| `--limit N` | Only change the first N hosts. Use `--limit 1` first. |
| `--revert FILE` | Put the hosts in a revert file back on their old certificate. |
| `--npm LABEL` | Optional label for the instance; it only names the revert file. |
| `--state-dir DIR` | Where the revert file is written (default: current directory). |

## Behavior

- A host is only moved if **every** one of its domain names is covered by the new certificate. A wildcard
  covers exactly one label: `app.example.com` matches `*.example.com`, `a.b.example.com` does not. Hosts that
  do not match are listed as `SKIP` and left alone.
- Handles proxy, redirection and 404 hosts. Stream hosts are skipped.
- Only the certificate id of a host is changed; its other settings stay as they are. NPM reloads nginx for
  each changed host, so a very short reload can happen per host.
- Every `--apply` run writes a revert file (`swap-<label>-<date>-<time>.json`). Keep it until you have
  confirmed that everything works. `--revert` is not heavily tested, so try it on one host first.
- Take a backup or snapshot of the NPM host before a large swap.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `login failed (HTTP 400)` | NPM logs in with the user's **e-mail address**, not a username. Check the password and that 2FA is off for that user. |
| The list shows no hosts, or `used by 0` everywhere | You are logged in as a non-admin user. Use an NPM admin. |
| Hosts show up as `SKIP` | The new certificate does not cover all of the host's domain names. |
| `FAIL` on a host | Read the HTTP status next to it. NPM may reject the change, or the new certificate id may not exist. |

## How it works

It logs in at `/api/tokens`, lists certificates (`/api/nginx/certificates`) and hosts
(`/api/nginx/proxy-hosts`, `/redirection-hosts`, `/dead-hosts`), and for each host that uses the old
certificate and is covered by the new one sends a `PUT` with only the new `certificate_id`.

## License

MIT, see [LICENSE](LICENSE).
