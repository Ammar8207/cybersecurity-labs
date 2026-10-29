# HTB - Fireflow (Medium)
**Difficulty:** Medium
**OS:** Linux

## Tools Used
- nmap
- curl
- Python3 (websockets, base64, json)
- nc (reverse shell listener)

## Steps
1. Ran `nmap -sC -sV -p-` — found SSH (22) and nginx (443); several high ports (9100, 30000, 30718, 30951, 31038, 31337) filtered, hinting at Kubernetes NodePort services
2. TLS certificate on port 443 revealed `commonName=fireflow.htb` and SAN `*.fireflow.htb` — wildcard cert implies subdomains in use
3. Added `fireflow.htb` to `/etc/hosts` and browsed the site — found a reference to `flow.fireflow.htb` embedded in the page (an "Open Agent" link to a Langflow playground)
4. Added `flow.fireflow.htb` to `/etc/hosts`; the playground URL contained a public flow UUID
5. Queried `/api/v1/version` — confirmed Langflow 1.8.2; `/api/v1/auto_login` confirmed auth was enforced normally
6. Opened the playground URL in-browser to obtain a `client_id` cookie (required by the public-flow build mechanism)
7. Identified **CVE-2026-33017** — the `POST /api/v1/build_public_tmp/{flow_id}/flow` endpoint is intentionally unauthenticated for running public flows, but also accepts an optional `data` parameter containing a full attacker-supplied flow graph, which is passed to `exec()` with no sandboxing
8. Crafted a malicious flow graph containing a `CustomComponent` node whose `code` field held a top-level `os.system()` call (`_x = os.system(...)`) — critical detail: `ast.Assign` statements execute during graph *compilation*, before the flow even runs, so a structurally valid wrapper `Component` class was also required to pass validation
9. Sent the crafted JSON via `curl` with the `client_id` cookie attached — triggered a reverse shell as `www-data`
10. Enumerated the environment: `env` leaked `LANGFLOW_SUPERUSER_PASSWORD`
11. Checked `/etc/passwd` for human accounts — found local user `nightfall`
12. Reused the leaked Langflow password via `su nightfall` — succeeded (password reuse); read user flag
13. `sudo -l` showed no privileges; found a hidden `~/.mcp/config.json` containing credentials (`langflow-bot` / password) and a URL pointing to an internal service on port 30080
14. Confirmed the box was a k3s node (`ip addr` showed `flannel.1`, `cni0` interfaces) — port 30080 was a Kubernetes NodePort service for an "MCP AI Tool Registry"
15. Queried `/api/v1/version` on the MCP service — it advertised JWT `supported_algorithms: ["HS256", "none"]`
16. Authenticated with the leaked credentials, got back a valid `role: user` JWT
17. Exploited the `alg: none` weakness — forged an unsigned JWT with `role: admin` locally (no secret key needed), since the server accepts an empty signature when `alg` is `none`
18. Verified the forged token against the tool-listing endpoint, then used the admin-only `POST /api/v1/tools` endpoint to register a tool containing a reverse-shell payload
19. Triggered the tool via the MCP JSON-RPC endpoint (`POST /mcp`, `tools/call`) — landed a shell inside the `mcp-server` Kubernetes pod (uid=1000)
20. Grabbed the pod's service account token from `/var/run/secrets/kubernetes.io/serviceaccount/token`
21. Queried `SelfSubjectRulesReview` against the Kubernetes API server — found the service account only had `GET nodes/proxy` (no `create`), blocking the standard exec-via-API-server path
22. Used the allowed `GET nodes/proxy` to enumerate pods running on the node (`/api/v1/nodes/fireflow/proxy/pods`) — identified `prometheus-prometheus-node-exporter` in the `monitoring` namespace as a privileged pod (runs as root, mounts host filesystem)
23. Confirmed exec via the API server's node proxy was forbidden (RBAC blocks `create` on `nodes/proxy`), and that hitting the kubelet directly on port 10250 with the standard HTTP `/run` path was equally blocked
24. Found that the kubelet's `/exec` endpoint uses a **WebSocket** protocol (`v4.channel.k8s.io`) rather than plain HTTP, and the kubelet did not enforce the same `create` restriction for establishing that WebSocket channel
25. Wrote a Python WebSocket client that connects directly to `wss://<node>:10250/exec/<namespace>/<pod>/<container>` using the pod's own service account token, bypassing the API server's RBAC check entirely
26. Ran `id` through the WebSocket exec — came back as `uid=0(root)` inside the `node-exporter` container
27. Read the root flag from the mounted host filesystem at `/host/root/root/root.txt`

## What I Learned
- A wildcard TLS SAN (`*.domain`) is a strong, low-noise signal that virtual hosting/subdomains are in play — always worth grepping page source and response headers for leaked subdomain references before brute-forcing
- "Public" or "unauthenticated by design" endpoints are a recurring class of vulnerability in AI-tooling platforms (Langflow here) — when an endpoint is deliberately unauthenticated for a legitimate reason (running a shared/public flow), any optional parameter that lets the client override server-stored data is a red flag, because it turns "run this trusted thing" into "run whatever I send you"
- Top-level Python assignments (`ast.Assign`) inside a "component" or "plugin" code field can execute during *compilation/validation*, not just at runtime — this matters when a target's threat model assumes "the code only runs if the flow executes successfully"
- Environment-variable enumeration (`env`) right after landing a web-app shell is consistently one of the highest-value first moves — app secrets leak there constantly, and they get reused as system account passwords far more often than they should
- `alg: none` support in a JWT implementation is a complete authentication bypass, not a theoretical weakness — if the server will accept an unsigned token, you don't need the HMAC secret at all, you just forge the payload and leave the signature empty
- Kubernetes RBAC is enforced per-verb and per-resource (`get` vs `create` on `nodes/proxy`), and different components in the cluster can enforce that policy inconsistently — the API server correctly blocked `create` on the exec subresource, but the kubelet's own WebSocket exec endpoint on port 10250 didn't re-validate that restriction the same way, making it a usable bypass path once you have any valid token with `get nodes/proxy`
- A DaemonSet like `node-exporter` running with `hostPID`, `hostNetwork`, and a host filesystem mount, combined with root privileges inside the container, is effectively full node compromise if that pod is ever reachable — monitoring/observability tooling is frequently over-privileged by default and is a common container-escape target in real clusters, not just CTFs
