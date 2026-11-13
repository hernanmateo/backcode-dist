---
name: server-metrics
description: Optionally set up server metrics (CPU, memory, disk) for a self-hosted server in a Backcode project - a small cron script that keeps N days of samples in a CSV, served by nginx or Caddy behind a token, so the Backcode app shows charts and the runner sends push alerts. Use only when the user asks for it ("métricas del servidor", "monitorizar el VPS", "server metrics", "avisos de disco"), or runs /backcode:server-metrics. Never set it up on your own initiative.
---

# Server metrics for a self-hosted server (optional)

For projects on a server the user manages (a VPS, a dedicated box), where there's no Backcode runner. The server keeps its
own history; Backcode stores nothing but the last reading:

- A **script** (cron, every 5 minutes) appends `ts,cpu,mem_used,mem_total,disk_used,disk_total` to
  `/var/lib/backcode/metrics.csv` (ts in Unix seconds, cpu in %, memory and disk in bytes) and drops lines older than N days.
- The **web server** (nginx or Caddy) serves that file at an https path **only** with `Authorization: Bearer <token>`,
  with CORS so the web version of the app can read it.
- The **app** downloads the CSV to draw the charts; the user's **runner** reads only its end every 10 minutes and the team
  gets a push when CPU, memory or disk go over the limit, when samples stop arriving or the file can't be read, and when
  it's back to normal.

Talk to the user in their language. Be brief. It's optional: if they hesitate, don't push it.

## 1. The server in the app first

The token comes from the app. Ask the user to open the project in Backcode › **Links** (🔗) › **Servers** › **Add server**,
with a name and the https address where the CSV will be served (e.g. `https://<their domain>/backcode-metrics.csv`), the
days to keep (7 or 30) and the alert limits. The server's screen then shows the **token** and a guide with the exact
snippets. Ask them to paste you the token (or the snippets). Never invent one, and never write the token into the repo.

## 2. Where to change things

- **Repo with the server config** (nginx `server {}` blocks, a `Caddyfile`, Ansible, Docker…): make the change there, in its
  own branch and PR the Backcode way (`/backcode:feature`). The token is a secret: read it from an env var or the
  deployment's secret store (`$BACKCODE_METRICS_TOKEN`), never commit it. If the repo has no secret mechanism, say so and
  let the user put the token on the server by hand.
- **Directly on the server** (the user gives you SSH access or runs the commands): explain each command before running it,
  and use `sudo` only where needed.

## 3. The steps

1. **Collector**: copy the script from the app's guide (step 1) to `/usr/local/bin/backcode-metrics` (`chmod 755`) and
   add `/etc/cron.d/backcode-metrics` with `*/5 * * * * root /usr/local/bin/backcode-metrics`. It needs Linux (`/proc`,
   GNU `df`). Run it once by hand and check the CSV has a header and a line.
2. **Web server**: add the location from the app's guide (step 2) inside the **https** server block of that domain:
   - nginx: `location = /<path> { … }` with the token check before `alias /var/lib/backcode/metrics.csv;`. Then
     `nginx -t` and reload (`systemctl reload nginx`).
   - Caddy: the `@backcodeMetrics` block inside the site. Then `caddy validate` and `caddy reload` (or the container's
     equivalent).
   Never serve it over plain http, never without the token check, and never expose the whole `/var/lib/backcode` folder.
3. **Check**: `curl -sf -H "Authorization: Bearer <token>" -H "Range: bytes=-200" https://…/backcode-metrics.csv` prints
   the last line; without the header it must be `401`.
4. Tell the user to open the server in the app: the charts show up right away, and the first reading from their runner
   arrives within 10 minutes (the runner must be connected to the project).

## Rules

- Optional and opt-in only. Don't touch anything else on the server (no agents, services or open ports).
- Never commit or log the token. If it leaks, the user changes it from the app (Edit › Change the token) and you update
  the web server config.
- Don't read `~/.backcode/config/`.
