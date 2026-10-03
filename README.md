# Memory - Cell-based Personal Agent System

This project runs a personal assistant for many users at once. Each user gets their own private space, called a UserCell, where their messages, facts, and daily routines are handled.

Think of it like an apartment building: each user has their own apartment (UserCell), the building has several workers to keep things running (Nodes), and everything is backed up safely off-site (S3).

## How It Works in Plain English

1.  **Something comes in.** This can be a chat message from web, mobile, or API, an incoming email, or an outside event like a calendar change or GitHub update.
2.  **A front door sorts it out.** An Application Load Balancer receives all incoming traffic and sends it to a healthy machine. It only listens in public, nothing private lives there.
3.  **It goes to a private fleet of workers.** Inside a private network lives the `celld fleet`. Right now there are Node A, Node B, and Node C. A and B do the work today, C is where we add more room when we grow. The nodes talk to each other over an internal listener to share work and copy logs.
4.  **It lands in your personal cell.** Every user has exactly one UserCell. This is where your stuff lives and where decisions are made for you. It has three parts:
    *   **Inbox:** A waiting line for new events.
    *   **Alarms:** A daily routine that wakes up on a schedule.
    *   **Agent loop:** The thinker. It takes items from the Inbox and Alarms, reasons about what to do, and then acts.
5.  **The cell remembers things.** Inside each UserCell is a small SQLite database for memory. It stores:
    *   **Facts:** Important things learned about you, with dates.
    *   **Message log:** The raw history of back-and-forth turns.
    *   **Warm summary:** A short summary so the cell can start up fast without reading everything.
6.  **The cell thinks with outside help and then acts.** To reason, the cell calls an outside LLM provider (the model API, with streaming). Once it decides, it can do outbound actions like sending email, push notifications, calendar updates, or calling other tools.
7.  **Everything is backed up.** The memory from each cell is copied using LTX replication to an S3 bucket in the same region. That bucket also holds:
    *   **Deployments:** The code bundles for the workers.
    *   **Replicas:** Copies of the SQLite memory as LTX files.
    *   **Ownership:** A record of which node serves which user.
    *   **Node leases:** Heartbeats from nodes that are renewed, and expire if a node dies.

If a node fails, another node can pick up the UserCell from the backup and ownership record and keep going.

## Parts At A Glance

**Outside inputs:**
*   Chat client - Web, mobile, API messages from you.
*   Email provider - Incoming email forwarded to a webhook.
*   Webhooks - Outside events like Calendar or GitHub.

**Front door:**
*   Application load balancer - Public, health-checked entry point.

**Private workers:**
*   celld fleet - A group of machines in a private subnet.
*   Node A / B / C - Workers that each hold many cells. Add more nodes to scale.
*   Internal listener - Private channel for peer calls and log copies.

**Per user:**
*   UserCell - One isolated space per user.
*   Inbox - Queued events waiting to be handled.
*   Agent loop - Reasons and acts.
*   Alarms - Daily routine triggers.
*   SQLite memory - Facts, Message log, Warm summary.

**Outside services:**
*   LLM provider - The model used for thinking, called with streaming.
*   Outbound actions - Email, push, calendar, tools.

**Backup and coordination:**
*   S3 bucket (same region, IAM role) - Safe storage.
*   LTX replication - The way SQLite copies are shipped to S3.

## Build (celld, full fleet prod)

Built on [celld](https://celld.dev) — self-hosted distributed Durable Objects. One `UserCell` Durable Object per user, SQLite memory per cell, LTX replication to a bucket you own.

Layout:

*   `src/index.ts` — public Worker: `/api/cell/:user/chat`, `/hooks/email`, `/hooks/:source`, `/health`, `/.well-known/celld/health`, cron heartbeat.
*   `src/usercell.ts` — `UserCell` DO: inbox queue, agent loop, daily alarm, tables `inbox`, `message_log`, `facts`, `warm_summary`, `outbox`.
*   `public/index.html` — minimal chat UI.
*   `wrangler.jsonc` — celld deploy config (DO binding, sqlite migration, cron, assets).
*   `fleet/` — prod fleet: `docker-compose.yml` (ALB + node-a/b/c), `nginx-alb.conf`, `celld.env.example`, `s3-policy.json`.

LLM is DeepSeek via OpenRouter (`OPENROUTER_MODEL` defaults to `deepseek/deepseek-chat-v3-0324:free`). Without `OPENROUTER_API_KEY` the agent runs stubbed so inbox/alarms/memory work offline.

### 1. Local run

Needs: celld binary (`curl -fsSL https://celld.dev/install.sh | sh`), esbuild on PATH.

```sh
cp .dev.vars.example .dev.vars
# put your OpenRouter key in .dev.vars for real replies
npm run dev
# UI: http://127.0.0.1:9876
```

Try:

```sh
curl -X POST http://127.0.0.1:9876/api/cell/demo/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"remember my dog is called Biscuit"}'

curl http://127.0.0.1:9876/api/cell/demo/state
```

### 2. Fleet deploy (prod)

Same region bucket + IAM role restricted to that bucket (see `fleet/s3-policy.json`).

```sh
export CELLD_BUCKET=s3://YOUR-FLEET-BUCKET
export S3_ENDPOINT=https://s3.YOUR-REGION.amazonaws.com
export AWS_REGION=us-east-1
# vars (OpenRouter key etc.) via celld vars / env — never in git
npm run deploy
```

Run nodes (each same bucket, distinct `--advertise` on the private net):

```sh
cp fleet/celld.env.example fleet/celld.env
# fill bucket + OpenRouter key
docker compose -f fleet/docker-compose.yml up
```

In AWS: ALB -> target group on node `:8080`, health check `/.well-known/celld/health`; nodes in private subnet with `--internal-listen` + `--advertise` reachable only inside the VPC; bucket in same region; nodes on IAM role from `fleet/s3-policy.json`.

Ops:

```sh
npm run diagnose
npm run cell:list
npm run fleet:status   # diagnose + cell list + cells-per-node rollup
npm run token -- demo  # mint per-user Bearer (needs API_AUTH_SECRET)
bash scripts/rotate-secrets.sh  # fresh EMAIL/WEBHOOK/API secrets, strips static AWS keys
```

Auth: set `API_AUTH_SECRET`, then `Authorization: Bearer <user>.<hmac>` on all
`/api/cell/:user/*` (UI stores it per user). Fleet nodes use an IAM role only —
see `fleet/IAM.md`; never commit `fleet/celld.env`, `.dev.vars`, or
`wrangler.fleet.jsonc` (all gitignored, secrets injected at deploy time).

Bucket holds: `deploy/` worker bundles, cell SQLite as LTX replicas, ownership records (who serves what), node leases (renewed, expire). Kill a node and another takes over its cells from the bucket — RPO=0.
