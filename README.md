# Bifrost on Huawei Cloud

A single skill folder for running self-hosted [Bifrost](https://github.com/maximhq/bifrost) on a Huawei Cloud ECS. Agents read `SKILL.md`. This file is the same procedure for a person.

The worked example is region **AF-Johannesburg** (`af-south-1`). Use another region only when you mean to. Stay on **pay-as-you-go**. Yearly/monthly billing is a separate decision, made later, in writing.

## When to use

Use this when you want a small Ubuntu ECS in your own Huawei Cloud account, Docker, Bifrost bound to loopback, and nginx in front of it. It also covers upgrades that keep the data directory, virtual keys, the rerank document-shape bug, and what still bills after you stop the VM.

## Choose a path

Use **Path A** when you will click through the Huawei Cloud console yourself. Use **Path B** when an agent should create the server with AK/SK, and only after you answer yes to its cost note.

## Path A — Console

You click. The region example is AF-Johannesburg.

1. Sign in to the Huawei Cloud console. In the account menu, select the account that should receive the bill.
2. Set the region to **AF-Johannesburg**.
3. Reuse a VPC and subnet already in that region. Create a VPC and one subnet only when the region has none (Service List → Virtual Private Cloud → Create VPC).
4. Create a security group. Inbound rules: TCP **22**, **80**, and **443**. If you have an office or home prefix, use it as the source for port 22. Check the rule list before you buy.
5. Create a key pair in this region (ECS → Key Pair) or reuse one. The private key downloads once. Keep it off this repo.
6. Open ECS → **Buy ECS** → **Custom Config**. Quick Config does not expose the flavor, disk type, and EIP billing mode used here.
7. **Billing Mode: Pay-per-use.** Leave Yearly/Monthly, Spot pricing, and reserved instances unset. Quantity **1**.
8. Region AF-Johannesburg, any available availability zone, CPU architecture **x86**.
9. Flavor **`c6.xlarge.4`**, which is **4 vCPU / 16 GB**. If the list does not show that flavor, stop and pick another flavor that is still 4 vCPU and 16 GB.
10. Public image **Ubuntu 22.04** 64-bit. Leave Host Security Service off unless you intend to pay for it.
11. System disk **General Purpose SSD**, **80 GB**.
12. Network: the VPC and subnet from step 3, and the security group from step 4. EIP **Auto assign**, type **Dynamic BGP**, **Billed By: Traffic**. Set a bandwidth cap you choose (5 Mbit/s is a small example). The cap is a peak, not a fixed bandwidth package. Do not choose Bandwidth or Shared bandwidth.
13. Log in with the key pair. Name the ECS `bifrost`.

### Where the hourly price is shown

Read both of these **before** you buy:

- The price at the **bottom of the Buy ECS page**. Hover it for the flavor, disk, and EIP lines. **Pricing details** opens the price-list breakdown.
- The **Configuration Summary** panel on the right of that same page.

**Submit** is the buy click. Confirm the summary still says pay-per-use, `c6.xlarge.4` (4 vCPU / 16 GB), General Purpose SSD 80 GB, and EIP billed by traffic.

### After the VM is running

SSH in as `ubuntu` with the private key. Install Docker from Ubuntu and build Bifrost from a checkout of its source. The UI is compiled into the Go binary, so a UI change is invisible until you rebuild the image.

```bash
sudo apt-get update
sudo apt-get install -y docker.io nginx
sudo systemctl enable --now docker
```

From the Bifrost repo root (`transports/Dockerfile` needs `ui/`, `transports/`, `core/`, `framework/`, and `plugins/` in the build context):

```bash
docker build -f transports/Dockerfile \
  --build-arg VERSION="$(git describe --tags --always)" \
  -t bifrost:local .
```

Pass `VERSION`. The image sets `main.Version` to `v${VERSION}`. Without the build arg, `GET /api/version` stays `vunknown`.

If `main` is ahead of the published Go modules, point `core`, `framework`, and each plugin at the local trees with `go mod edit -replace`, then build with `-mod=mod`. Skip the replace block when the published modules already match the checkout.

The image runs as uid **1000**, gid **0**. The data directory on the host must be writable by that pair. The process listens on loopback only.

```bash
sudo mkdir -p /var/lib/bifrost
sudo chown -R 1000:0 /var/lib/bifrost
docker run -d --name bifrost --restart unless-stopped \
  --user 1000:0 \
  -p 127.0.0.1:8080:8080 \
  -e APP_DIR=/var/lib/bifrost \
  -v /var/lib/bifrost:/var/lib/bifrost \
  bifrost:local
```

If `config.db` cannot be created, run `chown` again and restart the same container. Do not delete `/var/lib/bifrost` to fix permissions.

```bash
curl -sS http://127.0.0.1:8080/health
curl -sS http://127.0.0.1:8080/api/version
```

Put nginx on port 80 so the EIP is the public URL. `proxy_buffering off` lets streams flush.

```nginx
server {
  listen 80;
  server_name _;
  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_buffering off;
  }
}
```

Open `http://<eip>/` in a browser.

### Hostname

Skip this unless you want a name and a certificate.

1. DNS A record from that name to the EIP.
2. Issue a certificate with HTTP-01 (for example certbot's nginx plugin) or install a certificate you already have.
3. Terminate TLS on nginx and keep the proxy to `http://127.0.0.1:8080`, with `proxy_buffering off`.

### After Bifrost is up

- **Rerank.** Qwen-style rerank endpoints expect `documents` as a JSON array of strings. Bifrost's OpenAI encoder sends `[{"text":"..."}]` and the service returns 400. A gateway that forwards plain strings returns 200. Fix the gateway (emit a JSON string when the document is only text; keep an object when id, data, or metadata is present) and rebuild. A 400 that mentions `documents` means the bug is still there.
- **Virtual keys.** New keys use the compiled prefix, `sk-bf-` unless you change `VirtualKeyPrefix` and rebuild. On load, a stored key is kept only when it already starts with `sk-bf-`. OSS has teams and virtual keys. It does not create email sub-users (`POST /api/users` returns 405). There is one dashboard admin.
- **Metrics.** Prometheus is `GET /metrics` (`bifrost_*`, `http_*`). Log totals are `GET /api/logs/stats` (counts, tokens, average latency). Histogram buckets are p90, p95, and p99, not a windowed p50. There is no 429 aggregate. Empty Cloud Eye series for a v2 realtime service do not mean the service is idle. Queue depth and KV-cache gauges belong to the vLLM scrape, not to this gateway.
- **Upgrades.** If the git tree is already newer than a tag, do not check out that tag. When you do upgrade, pass `VERSION`, recreate the container, and keep the same `/var/lib/bifrost` mount. Never delete that directory.

### Stop and delete

**Stop** on the ECS list stops compute charges for this pay-per-use VM. The disk and the EIP keep billing.

**More → Delete**, with release of the EIP and deletion of the data disks selected, releases the server, the public IP, and the disk charges. The system disk goes away with the server. After the server is gone, delete the security group you created for it if nothing else uses that group. Leave the VPC and subnet unless you have decided to remove those too.

## Path B — AI with AK/SK

You can paste an AK and SK into the agent chat. They stay in that conversation. The agent must not write them into this repo, must not echo the SK, and must not reuse credentials from any other account.

Before it creates anything, the agent lists IAM projects and the existing VPC, subnet, and ECS resources in this account and region, and shows you that list. It reuses a VPC. It does not create a VPC unless you say yes.

It then sends a **cost note in your language** and stops. The note names the flavor and its vCPU/RAM, the disk size and type, the EIP billing mode and bandwidth cap, and that the purchase is pay-as-you-go. Monthly conversion is a later confirmation, not part of this yes. The agent queries the Huawei pay-per-use price API (`POST /v2/bills/ratings/on-demand-resources` on `bss.myhuaweicloud.com`) and quotes only what that response returns. If the API is unavailable or returns no price, the note says the console price was not queried and does not invent a number.

Reply with an explicit yes before any security group, ECS, disk, or EIP is created. After that yes the agent creates those resources on pay-as-you-go (no prepaid charging mode), installs Docker, deploys Bifrost on `127.0.0.1:8080` with the data directory from Path A, and reports the public URL.

The software steps after SSH are the Path A install: Docker, the `VERSION` build arg, uid 1000 gid 0 on `/var/lib/bifrost`, nginx, and a certificate only if you asked for a hostname. The rerank, virtual-key, metrics, and upgrade notes in Path A apply here too.

Tear-down needs a **second** confirmation. The agent then deletes the server with its public IP and volume, waits until the server is gone, and deletes the security group it created. Stopping the ECS does not stop disk and EIP charges.

## Install

One folder, named `bifrost-huawei-cloud`, with this `SKILL.md` inside it. `name` in the frontmatter matches the folder name. Copy this folder once to the tool that should load it. Do not keep four edited copies.

| Tool | Project copy | User copy |
| --- | --- | --- |
| Cursor | `.cursor/skills/bifrost-huawei-cloud/` | `~/.cursor/skills/bifrost-huawei-cloud/` |
| Claude Code | `.claude/skills/bifrost-huawei-cloud/` | `~/.claude/skills/bifrost-huawei-cloud/` |
| Codex | — | `~/.codex/skills/bifrost-huawei-cloud/` |
| CodeArts | `.codeartsdoer/skills/bifrost-huawei-cloud/` | `~/.codeartsdoer/skills/bifrost-huawei-cloud/` |

Cursor reads project skills from `.cursor/skills` and personal skills from `~/.cursor/skills`. Claude Code reads `.claude/skills` and `~/.claude/skills`. Codex reads `~/.codex/skills` (`$CODEX_HOME/skills` when that variable is set). CodeArts reads `.codeartsdoer/skills` and `~/.codeartsdoer/skills`. The directory name includes the `doer` suffix. On CodeArts, a project skill overrides a user skill of the same name.

## Safety

**Costs.** The example size is pay-as-you-go: `c6.xlarge.4` (4 vCPU / 16 GB), 80 GB General Purpose SSD, and an EIP billed by traffic with a bandwidth cap. The buy page (Path A) or the price API (Path B) is the source of the number. Stopping the ECS leaves disk and EIP charges running. Deleting the ECS and releasing the public IP and data disk is what ends those charges. Yearly/monthly billing is a later, separate confirmation.

**Secrets.** AK, SK, SSH private keys, and the dashboard admin login do not belong in this repo, in commits, or in echoed logs. Do not print the SK. A virtual key is for API callers; the admin login is not.

**No customer data.** Do not add customer names, account IDs, project IDs, live IP addresses, hostnames, or another account's credentials to this repository. Resources stay in the account you selected for this task.
