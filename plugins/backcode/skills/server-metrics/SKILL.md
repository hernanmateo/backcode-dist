---
name: server-metrics
description: Optionally connect a Linux server to Backcode's server metrics (CPU, memory, every disk volume and whether its services answer, with push alerts) - one install command from the app, a tiny systemd service that sends the data over https, whatever the server runs (MySQL, MongoDB, an API, a worker…). A paid-plan feature (Pro, Pro Full or Team). Use only when the user asks for it ("métricas del servidor", "monitorizar el VPS", "server metrics", "avisos de disco"), or runs /backcode:server-metrics. Never set it up on your own initiative.
---

# Server metrics (optional)

Backcode can watch any Linux server with systemd, whatever it runs: the server sends its CPU, memory, disk volumes and
whether the services you choose answer (e.g. `mysql:3306`) every 5 minutes, over https, to Backcode. No ports are opened,
no domain or certificate is needed, and it works behind NAT. The app shows the state and 7 days of charts, and the team
gets a push when something goes over its limit, a service stops answering or the server stops sending, and when it's back
to normal. It's part of the paid plans (Pro, Pro Full, Team): on Personal the server's data is rejected.

Talk to the user in their language. Be brief and keep it simple: one command per server. Don't bring up the internals
unless they ask.

## 1. In the app

Ask the user to open the project in Backcode › **Links** (🔗) › **Servers** › **Add server**: a name, the alert limits
and, optionally, the services to watch (`name:port`, e.g. `mysql:3306, mongo:27017`; another host: `redis:10.0.0.5:6379`).
The server's screen then shows the **install command**, with its token inside. Ask them to paste it to you. Never invent a
token, and never write it into the repo or a log.

## 2. On the server

- **The user runs it** (most common): tell them to paste the command on the server. It needs `sudo`, `curl` and systemd.
- **You have SSH access**: explain what you're about to run, then run that exact command on the server.
- **Servers managed from a repo** (Ansible, cloud-init, Terraform…): add a step that runs the same command, with the token
  from the deployment's secret store (never committed). Do it the Backcode way (`/backcode:feature`, its own PR).

It prints `✓ Backcode: servidor conectado` and, within seconds, the app shows the server as connected. If it fails, look at
`journalctl -u backcode-metrics -n 20` and `systemctl status backcode-metrics.timer`:
- `401`: wrong token (it was changed in the app: copy the command again).
- `402`: the group isn't on a paid plan.
- No network or DNS: the server needs outbound https to `*.cloudfunctions.net`.

## 3. What it installs (only if they ask)

A `backcode-metrics` systemd service and timer (every 5 minutes, `DynamicUser`, so not root), the sender in
`/usr/local/lib/backcode-metrics/` and the token in `/etc/backcode-metrics.env` (root only). To remove it, the app has the
command under **What it installs**, or run the install command with `| sudo sh -s uninstall`.

## Rules

- Optional and opt-in only. Don't touch anything else on the server.
- Never commit or print the token beyond what's needed. If it leaks, the user changes it in the app (Edit › Change the
  token) and the install command is run again with the new one.
- Don't read `~/.backcode/config/`.
