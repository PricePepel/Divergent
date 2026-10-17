# Freestyle notes for Diego (hosting, what-if clone, pause)

From Leo. Commands are taken from the Freestyle CLI docs (https://www.freestyle.sh/docs/vms/cli) and have **not been run yet**,
so expect small fixes. Placed in `analysis/` because that's Leo's folder; move it into `api/DEPLOY.md` if you like.

Install the CLI with `npm i -g freestyle@latest` (or prefix every command with `npx -y freestyle@latest`).
Free `*.style.dev` names are first-come; pick another name if one is taken.

## 0. Setup (once)
```bash
freestyle login
freestyle billing redeem CALCOMPILED   # hackathon credits from the Luma email
```

## 1. Host the API
```bash
freestyle vm create --slug divergent-api --no-ssh
freestyle vm exec divergent-api --linux-user root --timeout-ms 300000 -- bash -lc '
  apt-get update && apt-get install -y git python3-venv &&
  git clone https://github.com/diegosr-byte/Divergent /root/divergent &&
  python3 -m venv /root/venv && /root/venv/bin/pip install fastapi uvicorn numpy filterpy pydantic &&
  printf "[Service]\nWorkingDirectory=/root/divergent\nExecStart=/root/venv/bin/uvicorn api.main:app --host 0.0.0.0 --port 8000\nRestart=always\n[Install]\nWantedBy=multi-user.target\n" > /etc/systemd/system/api.service &&
  systemctl daemon-reload && systemctl enable --now api'
freestyle tls create --domain divergent-api.style.dev --from public --to vm=divergent-api,port=8000
```
Redeploy after a push:
```bash
freestyle vm exec divergent-api --linux-user root -- bash -lc 'cd /root/divergent && git pull && systemctl restart api'
```
`vm exec` is capped at 5 minutes, which is why uvicorn runs under systemd.

## 2. Clone for the what-if ("two futures, branched in seconds")
A snapshot keeps memory as well as disk, so the clone boots with the API already running and the scenarios already cached.
```bash
freestyle vm pause divergent-api
freestyle snapshot create divergent-api --slug divergent-warm
freestyle vm start divergent-api
freestyle vm create --snapshot-id divergent-warm --slug divergent-whatif --no-ssh
freestyle tls create --domain divergent-whatif.style.dev --from public --to vm=divergent-whatif,port=8000
```
Then apply the what-if inside the clone (e.g. disable S3) and point Neil's second map at `divergent-whatif.style.dev`.

**One thing to settle:** your brief says `/whatif` results aren't cached. For the clone to show a different picture at its
own URL, it has to keep the what-if result (or start with the sensor disabled).

## 3. Pause / wake
```bash
freestyle vm update divergent-api --idle-timeout-seconds 600   # auto-pause after 10 min idle; wakes on next request, same state
freestyle vm pause divergent-api
freestyle vm start divergent-api
```

## Limits (free plan)
10 VMs running at once, 4 vCPU / 8 GB / 32 GB disk each by default. Enough for one VM per preset (`demo`, `single`,
`swarm`, `sparse`) plus the what-if clone.
