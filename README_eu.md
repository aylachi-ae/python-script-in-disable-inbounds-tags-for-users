# marzban_disable_inbound.py

Interactive CLI-script for **mass disabling of selected inbound** for all users of the **Marzban** panel via its REST API.

The user is not deleted or blocked in this case — the specified inbound-tags just disappear from the `inbounds` field, so links/configs stop including these protocols.

---

## Features

- Logs into Marzban with `username` / `password` (OAuth2 password flow on `/api/admin/token`).
- Loads a list of all inbound from `/api/inbounds` and shows them as a numbered list.
- Asks which inbound to disable (multiple, comma-separated).
- Pulls all users via `/api/users` with **pagination**, **retries**, and **progress**.
- For each user, makes a `PUT /api/user/{username}` with an updated array of `inbounds`.
- Users who already had selected inbound are marked as `unchanged` — there are no extra requests.
- At the end, it displays a summary: updated / unchanged / with errors.

---

## Requirements

- Python **3.10+** (type-annotations `dict[...]`, `list[...]` are used).
- The `requests` package.

Installing the dependency:

```bash
pip install requests
```

---

## Running

```bash
python marzban_disable_inbound.py
```

The script will ask:

1. **Marzban URL** — the root of the panel's API. For example, `https://api.example.com`.
   - Important: this is **not** the UI path with `DASHBOARD_PATH` (the secret random prefix is only for the web interface).
   - API always lives in the root of the domain: `https://<host>/api/...`.
2. **Admin username** — login of the Marzban administrator.
3. **Admin password** — password (input is hidden).

Next — select the inbound numbers, confirm `yes`, and the script rolls out the changes to all users.

---

## Example of a session

```
=== Marzban: disable inbound for all users ===

Marzban URL (e.g. https://panel.example.com): https://api.example.com
Admin username: admin
Admin password:
Authenticated.

Available inbounds:
    1. [vless] VLESS_XHTTP_REALITY
    2. [vless] VLESS_TCP_REALITY

Which to DISABLE for all users? (numbers comma-separated, e.g. 1,3): 1,2

Will be removed from every user:
  - [vless] VLESS_XHTTP_REALITY
  - [vless] VLESS_TCP_REALITY

Apply to ALL users? (yes/no): yes

Loading users (paginated)...
  total users reported: 127
  fetched 25/127
  fetched 50/127
  ...
  fetched 127/127

Users loaded: 127

  [OK]   VPN-00013
  [OK]   VPN-00014
  ...

Done. Updated: 121, unchanged: 6, failed: 0
```

---

## What APIs it uses

| Method  | Endpoint                  | Why                          |
|--------|---------------------------|--------------------------------|
| POST   | `/api/admin/token`        | Get JWT (form-data)       |
| GET    | `/api/inbounds`           | List all inbound by protocols |
| GET    | `/api/users?offset&limit` | List users with pagination |
| PUT    | `/api/user/{username}`    | Update user's `inbounds`      |

All requests go with `Authorization: Bearer <token>`.

---

## Network resilience

The script is designed to work with Cloudflare / WAF / slow channel between you and Marzban:

- Uses **`requests.Session`** — TLS-handshake is reused.
- Connected **retry-adapter** from `urllib3` (5 attempts, exponential backoff 2/4/8/16 sec) for codes `429, 500, 502, 503, 504` and network failures.
- Split timeouts `(connect, read)`:
  - login / inbounds / PUT — `(10, 30)`
  - `/api/users` — `(15, 120)` (this is the heaviest endpoint).
- The size of the `/api/users` page is **25** (not 100), so that the CDN doesn't kill long responses.
- Each page has up to 3 additional manual retries with their own sleep.

If you still get a connect timeout on `/api/users`, it's almost certainly a WAF/Cloudflare filter. Here are some options to bypass it:

- Run the script from the Marzban server itself (via SSH).
- Set up an SSH tunnel and access it via `http://127.0.0.1:<port>`.
- Add `/api/users` to the whitelist of Cloudflare/WAF.

---

## What exactly changes for the user

The `inbounds` field in Marzban is a dictionary `{ protocol: [tag1, tag2, ...] }`. The script:

1. Takes the current `user.inbounds`.
2. For each protocol, filters the list of tags, removing the selected ones.
3. Sends a `PUT /api/user/{username}` with a body of `{ "inbounds": <new dictionary> }`.

The rest of the user's fields (`proxies`, `expire`, `data_limit`, status, etc.) are **not touched** - Marzban accepts partial updates.

If the user already had the selected protocols, the script skips them without a request.

---

## Limitations / known issues

- **No dry-run.** It's not clear who will be affected by the changes.
- **No backup.** The script doesn't save the old `inbounds` values before writing them.
- **Interrupting (Ctrl+C) in the middle** - some users will already be modified, while others won't. It's safe to restart, as the modified users will be marked as `unchanged`.Compatibility**: tested on the current version of the Marzban API (`/api/admin/token` + `UserResponse.inbounds: Dict[str, List[str]]`). If you have an older version of the panel, the field structure may differ.

---

## Files

```
marzban_disable_inbound.py   # the script itself
README.md                    # this file
```

---

## License

Use it as you wish. Before running it in production, run it on a test user or make an SQL backup of the Marzban database.

## Warning
**The idea belongs to me, and the song is Cloude Opus 4.7 Extra high.**
**I do not take any responsibility for its launch on your product.**

## Support 
I do not prohibit the use of this script by anyone, for the development of forks and updates, or for general use.
I'd appreciate it if you clicked the star if you liked the idea and the script!
You can also help financially and check out my other projects at https://aylachi.bio
