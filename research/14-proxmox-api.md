---
tags: [research, proxmox, api, automation, pvesh, curl, scripting]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Proxmox REST API Automation

Research date: 2026-09-17.

Base URL: https://<host>:8006/api2/json/. pveproxy (port 8006) serves the API and forwards cluster-wide. Responses wrap payloads in {"data": ...}. Interactive docs: https://<host>:8006/pve-docs/api-viewer/.

## Authentication

- Ticket (session): POST /access/ticket with username/password -> returns ticket (cookie PVEAuthCookie) + CSRFPreventionToken. Tickets expire ~2h; every write (POST/PUT/DELETE) must also send the CSRF header.
- API token (recommended for automation): `Authorization: PVEAPIToken=darko@pve!clitoken=<secret>` - stateless, no CSRF needed.
- Token scoping: `--privsep 0` = full user privileges; default `--privsep 1` = token gets ONLY explicit ACL grants. "Administrator" role implies all privs. Secret shown once; revoke anytime. Console/vncproxy endpoints cannot use tokens.

## curl recipes (our token user darko@pve)

```
AUTH="Authorization: PVEAPIToken=darko@pve!clitoken=SECRET"
H="https://10.10.10.10:8006/api2/json"
curl -sk -H "$AUTH" $H/version
curl -sk -H "$AUTH" $H/nodes
curl -sk -H "$AUTH" "$H/cluster/resources?type=vm"
curl -sk -H "$AUTH" $H/nodes/prodev/storage | jq '.data[]|{storage,used,total}'
curl -sk -H "$AUTH" $H/nodes/prodev/qemu | jq '.data[]|{vmid,name,status,cpu,mem}'
curl -sk -H "$AUTH" $H/nodes/prodev/status | jq '.data|{cpu,memory,uptime}'
curl -sk -X POST -H "$AUTH" "$H/nodes/prodev/qemu/100/snapshot" -d 'snapname=pre-upgrade'
curl -sk -X POST -H "$AUTH" "$H/nodes/prodev/qemu/101/status/start"
```

Ticket flow: POST $H/access/ticket -d username=... -d password=... then `-b "PVEAuthCookie=..."` and `-H "CSRFPreventionToken: ..."` on writes. `-k` disables self-signed TLS verify.

## pvesh mapping (on-node, root only)

- get=GET, create=POST, set=PUT, delete=DELETE, ls=browse paths, usage=schema.
- Params are --key value. Use `--output-format json|json-pretty|yaml` for jq-able output.
- `pvesh get /nodes/prodev/status --output-format json`
- `pvenode startall --vms 100,102 --force`; `pvenode config set --startall-onboot-delay 10` staggers boots.

## Homelab automation ideas

- Report: loop /cluster/resources?type=vm for cpu/mem + /nodes/prodev/status for load; emit JSON/CSV.
- Health: zpool = /nodes/prodev/disks/zfs or zpool status; storage = /nodes/prodev/storage; version = /version.
- Snapshots: POST snapshot -> UPID -> poll GET /nodes/prodev/tasks/<upid>/status until exited, check exitstatus.
- Backup status: /nodes/prodev/tasks?typefilter=vzdump or /cluster/tasks. Failures -> ntfy: `curl -d "Backup failed: $UPID" ntfy.sh/topic`.
- n8n/HomeAssistant/Grafana promote the same token-authed JSON endpoints.

## Caveats

- Long-running actions return UPID -> poll task status; never block on HTTP.
- No official rate limit, but be polite; token TLS with self-signed CA needs verify=False / -k (or trust the cluster CA).
- 401 = wrong token/id; 403 = lack ACL privilege on path (grant PVEAuditor/PVEVMAdmin per token).
- Tokens ignore CSRF; store secret in a chmod 600 env file, never leak/rotate casually.

## Python (requests) example

```python
import requests, os
H = "https://pve:8006/api2/json"
S = requests.Session(); S.verify = False
S.headers["Authorization"] = f"PVEAPIToken=darko@pve!clitoken={os.environ['PVE_TOKEN']}"
for r in S.get(f"{H}/cluster/resources?type=vm").json()["data"]:
    print(r["vmid"], r["name"], r["status"], round((r.get("mem") or 0)/1e9, 1), "GiB")
```

## Sources

- https://pve.proxmox.com/wiki/Proxmox_VE_API
- https://pve.proxmox.com/pve-docs/pveum.1.html
- https://pve.proxmox.com/pve-docs/pvesh.1.html
- https://proxmoxr.com/blog/proxmox-api-authentication
- https://github.com/ramphy/proxmox-api