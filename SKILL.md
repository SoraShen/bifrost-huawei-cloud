---
name: bifrost-huawei-cloud
description: >-
  Deploy and operate self-hosted Bifrost on Huawei Cloud ECS. Use when the
  user wants Bifrost on Huawei Cloud, including af-south-1 (Johannesburg),
  either by walking the Huawei Cloud console or by an agent using AK/SK.
  Covers pay-as-you-go compute, VPC reuse, Docker, the UI embedded in the Go
  binary, nginx, virtual keys, rerank document encoding, Prometheus and log
  stats, upgrades that must not downgrade or wipe data, and tear-down.
  Trigger on Huawei Cloud, ECS, Bifrost deploy, Bifrost upgrade, or AK/SK
  provisioning for this stack.
---

# Bifrost on Huawei Cloud

Self-host Bifrost on an ECS in the account the user is operating in this conversation. The worked example is region `af-south-1` (console name AF-Johannesburg). Substitute another region only when the user names one.

Both paths below are first-class. Start on pay-as-you-go. A switch to yearly/monthly billing is a separate later confirmation. Do not send `extendparam.chargingMode: prePaid` and do not select Yearly/Monthly until the user confirms that change in this conversation.

Use only this account. Never reuse another account's AK/SK, VPC, subnet, image, keypair, OBS bucket, or container registry. Ignore credentials found in other projects or dotenv files.

Do not write customer names, account IDs, project IDs, live IP addresses, hostnames, or secrets into this repository.

## Choose a path

Use Path A when the user will click through the Huawei Cloud console. Use Path B when the user wants the agent to create the server with AK/SK. Path B may receive AK/SK pasted into the chat. That material stays in the session.

## Secrets

Never write the AK or SK into the repo. Never echo the SK. Never print the AK, the `Authorization` header, a private SSH key, or the dashboard admin login. Never commit them. In notes use `<ecs-eip>`, `<virtual-key>`, `<project-id>`, and `<server-id>`.

On Path B, accept AK/SK only from this conversation, for this account. Store an SSH private key outside the repo, mode `0600`, and tell the user the path.

## Path A — Console

The user does the clicking. Walk them one screen at a time. Wait while they click. Do not invent a price.

1. Open the Huawei Cloud console and sign in. In the account menu, select the account that should receive the bill. If an enterprise or member-account switcher is present, select that account before any purchase.
2. Set the region selector to **AF-Johannesburg** (`af-south-1`). The region cannot be changed after purchase.
3. If this region already has a VPC and subnet, reuse them. Create a VPC and subnet only when the region has none: Service List → Virtual Private Cloud → Create VPC. One subnet in AF-Johannesburg is enough. Leave the VPC and subnet in place at tear-down.
4. Create a security group for this server (Virtual Private Cloud → Access Control → Security Groups, or **Create Security Group** on the ECS page). Inbound TCP **22**, **80**, and **443**. When the user gives an operator prefix, limit port 22 to that prefix. Otherwise the source is `0.0.0.0/0`. Confirm the rule list before buying. Do not rely on a default group unless those three rules are actually present.
5. Create or reuse a key pair in this region (ECS → Key Pair). The private key downloads once. Keep it outside the repo.
6. ECS console → **Buy ECS**. Open the **Custom Config** tab. Quick Config hides the flavor, disk type, and EIP billing mode this stack needs.
7. **Billing Mode: Pay-per-use.** Leave Yearly/Monthly, Spot pricing, and Associate RI unset. Leave scheduled deletion unset unless the user asked for it. Quantity **1**.
8. Region AF-Johannesburg. AZ: any available zone.
9. CPU architecture **x86**. Search flavors for `c6.xlarge.4`. It must show **4 vCPU / 16 GB**. If that flavor is absent, stop and name a 4 vCPU / 16 GB alternative. Do not pick a larger flavor.
10. Image: **Public image**, **Ubuntu 22.04** 64-bit (x86). If Host Security Service is offered, leave it off unless the user wants that product.
11. System disk: **General Purpose SSD** (GPSSD), **80 GB**. No extra data disk.
12. Network: the existing VPC and subnet from step 3. Security group from step 4. EIP: **Auto assign**. EIP type **Dynamic BGP**. **Billed By: Traffic**. Set Bandwidth Size to a cap the user chooses (a small example is 5 Mbit/s). That number is a peak cap, not a bandwidth package. Do not select Bandwidth or Shared bandwidth.
13. Login: **Key pair**, the pair from step 5. ECS name `bifrost`.
14. Read the price, then buy. Two surfaces show it before the buy click:
    - The price at the **bottom of the Buy ECS page**. Hover it to see line items (flavor, disk, EIP). **Pricing details** opens the price-list breakdown.
    - The **Configuration Summary** panel on the right.
    Confirm Pay-per-use, `c6.xlarge.4` (4 vCPU / 16 GB), General Purpose SSD 80 GB, and EIP billed by traffic. **Submit** on that page is the buy action. Do not click it until those two surfaces have been read. The page price is the price. Do not quote a figure the page did not show.
15. After the ECS is Running, copy its EIP from the ECS list. SSH as `ubuntu` with the private key.

Then follow [Run Bifrost](#run-bifrost). For a public URL with no hostname, install nginx on port 80 as described there. Add a certificate only when the user wants a hostname.

**Stop versus delete.** ECS list → **Stop** stops compute charges for this common pay-per-use instance. The system disk and the EIP keep billing. **More → Delete**, with **release the EIP** and **delete the data disks** selected, removes the server, the public IP, and the disks, and those charges stop. The system disk is deleted with the server. Delete the security group created for this server only after the server is gone, and only if nothing else uses it.

## Path B — AI with AK/SK

The user may paste AK/SK into the chat. Hold them in the session. Never write them into a file in the repo, never echo the SK, and never reuse credentials from another account.

### 1. List before any create

Sign Huawei REST with a local signer. Do not request a long-lived token. Do not log signature inputs that contain the SK.

IAM projects: `GET https://iam.myhuaweicloud.com/v3/projects`. Use the project whose `name` is the region (`af-south-1` unless the user named another). That `id` is `<project-id>` for regional calls in this session only.

Worked-example hosts:

- ECS `https://ecs.af-south-1.myhuaweicloud.com`
- VPC `https://vpc.af-south-1.myhuaweicloud.com`
- IMS `https://ims.af-south-1.myhuaweicloud.com`
- Price `https://bss.myhuaweicloud.com`

List and show the user, for this account and region only:

- IAM projects
- VPCs and subnets
- Existing ECSs (`GET /v1/<project-id>/cloudservers/detail`)

Reuse an existing VPC and subnet. If several exist, ask which one. If none exist, ask whether to create one. **Do not create a VPC unless the user says yes.** Listing is allowed before the cost confirmation. Creating a security group, keypair, ECS, disk, or EIP is not.

### 2. Sign requests

Algorithm `SDK-HMAC-SHA256` (SHA-256, hex lowercase):

1. `CanonicalRequest` = method, canonical URI, canonical query, canonical headers, signed headers, hex SHA-256 of the body, each joined by `\n`.
2. Canonical URI: encode each path segment; leave `/` as `/`.
3. Canonical query: sort by key, then value; encode keys and values.
4. Canonical headers: lowercase names, trim values, `name:value\n`, sorted. Always sign `host` and `x-sdk-date`. Sign `content-type` when there is a body.
5. `X-Sdk-Date` is `yyyyMMddTHHmmssZ`.
6. `StringToSign` = `SDK-HMAC-SHA256`, the same timestamp, hex SHA-256 of the canonical request, joined by `\n`.
7. Signature = hex HMAC-SHA256 of `StringToSign`, keyed by the SK.
8. `Authorization: SDK-HMAC-SHA256 Access=<ak>, SignedHeaders=<names>, Signature=<hex>`.

### 3. Cost note, then stop

Query the pay-per-use price before any create or mutate call:

`POST https://bss.myhuaweicloud.com/v2/bills/ratings/on-demand-resources`

(Customer Operation Capabilities: Querying the Price of a Pay-Per-Use Product.) Ask for the ECS flavor, the GPSSD system disk, and the traffic EIP in this region. Quote only amounts, units, and currency the response returns.

If that call is unavailable, unauthorized, or returns no amount, say **the console price was not queried** and do not invent a figure. Still send the rest of the note.

Send the cost note in the user's language. Include:

- Flavor id, with vCPU and RAM (`c6.xlarge.4`, 4 vCPU / 16 GB, after the flavor list confirms `16384` MB; if it is missing, stop and name a 4 vCPU / 16 GB alternative)
- System disk 80 GB, General Purpose SSD (GPSSD)
- EIP Dynamic BGP (`5_bgp`), billed by traffic (`chargemode: traffic`, `sharetype: PER`); the bandwidth number is a Mbit/s cap, not a fixed bandwidth package
- This purchase is pay-as-you-go
- Yearly/monthly conversion is a separate later confirmation and will not be done now
- The API price, or the sentence that the console price was not queried

Then **stop**. Wait for an explicit yes to create (for example "yes", "create it", or the same approval in the user's language). A question, a price comment, or silence is not a yes. No security group, keypair, ECS, disk, or EIP create before that yes.

### 4. After yes

Resolve ids at create time. Do not hardcode an image id, availability zone, or project id.

- Image: `GET /v2/cloudimages` on IMS. Gold image, name contains `Ubuntu 22.04`, architecture `x86_64`.
- Zone: list availability zones and pick one that is available.
- Flavor: `GET /v1/<project-id>/flavors` on ECS. `c6.xlarge.4` must show 4 vCPU and 16384 MB.
- Security group on the VPC API, then ingress rules for TCP 22, 80, and 443. Limit port 22 when the user gave a prefix.
- Keypair: reuse one in this region, or create one after the yes. Store the private key outside the repo.

`POST /v1/<project-id>/cloudservers` on ECS. Omit prepaid fields so the server stays pay-as-you-go.

```json
{
  "server": {
    "name": "bifrost",
    "imageRef": "<image-id>",
    "flavorRef": "c6.xlarge.4",
    "availability_zone": "<az>",
    "vpcid": "<vpc-id>",
    "nics": [{ "subnet_id": "<subnet-id>" }],
    "security_groups": [{ "id": "<security-group-id>" }],
    "key_name": "<keypair-name>",
    "root_volume": { "volumetype": "GPSSD", "size": 80 },
    "publicip": {
      "eip": {
        "iptype": "5_bgp",
        "bandwidth": {
          "size": "<bandwidth-cap-mbit>",
          "sharetype": "PER",
          "chargemode": "traffic"
        }
      }
    }
  }
}
```

The create call returns a job. Poll it. Keep `<server-id>` and `<ecs-eip>` in the session only.

Install Docker, deploy Bifrost, and put nginx on port 80 so the EIP serves the app. Report the public URL (`http://<ecs-eip>/`, or `https://<public-name>/` when the user asked for a hostname and a certificate).

### 5. Tear-down

Only when the user asks to destroy the stack. Ask for a **second** confirmation that is only about deletion. The create yes does not cover tear-down.

`POST /v1/<project-id>/cloudservers/delete` on ECS:

```json
{
  "servers": [{ "id": "<server-id>" }],
  "delete_publicip": true,
  "delete_volume": true
}
```

Wait until the server is gone, then delete the security group created for it. Do not delete a shared VPC or subnet unless the user asks.

Stopping the ECS (`os-stop`) does not stop disk or EIP charges.

## Run Bifrost

Build from a checkout of the Bifrost source. The UI is compiled into the Go binary with `//go:embed`. A UI edit does nothing until the image is rebuilt.

`transports/Dockerfile` expects the repo root as context (`ui/`, `transports/`, `core/`, `framework/`, `plugins/`).

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker

docker build -f transports/Dockerfile \
  --build-arg VERSION="<git-describe>" \
  -t bifrost:<version> .
```

Pass `VERSION`. The ldflags set `main.Version` to `v${VERSION}`. Without it, `GET /api/version` returns `vunknown`.

When `main` is ahead of the published modules, the image build must use this checkout: `go mod edit -replace` for `core`, `framework`, and each plugin module onto the local trees, then `go build -mod=mod`. Skip the replace block when the published modules already match the checkout. The image `USER` is `1000:0`. Default `APP_DIR` is `/app/data`.

The host publish address is loopback. nginx is the only public listener.

```bash
sudo mkdir -p /var/lib/bifrost
sudo chown -R 1000:0 /var/lib/bifrost
docker run -d --name bifrost --restart unless-stopped \
  --user 1000:0 \
  -p 127.0.0.1:8080:8080 \
  -e APP_DIR=/var/lib/bifrost \
  -v /var/lib/bifrost:/var/lib/bifrost \
  bifrost:<version>
```

The process runs as uid 1000, gid 0, and the entrypoint cannot fix ownership unless it starts as root. If `config.db` cannot be created, `chown -R 1000:0 /var/lib/bifrost` and restart the same container. Do not delete the directory to fix permissions.

Do not publish `0.0.0.0:8080`. Inside the container the process still binds the port so the docker proxy can reach it.

Check `http://127.0.0.1:8080/health` and `http://127.0.0.1:8080/api/version`.

Install nginx and proxy port 80 to loopback so the EIP is a public URL:

```bash
sudo apt-get install -y nginx
```

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

### Hostname and certificate

Only when the user wants a hostname:

1. Create a DNS A record for `<public-name>` to `<ecs-eip>`.
2. Issue a certificate with HTTP-01 (nginx or a webroot on port 80).
3. Terminate TLS on nginx and proxy to `http://127.0.0.1:8080`.
4. Set `proxy_buffering off` so streams flush.

Do not invent a domain. Do not write a real hostname into the repo.

## Providers

Register each realtime service that is `RUNNING` as a custom OpenAI-compatible provider. Set `network_config.base_url` to that service's OpenAI-compatible base. Keep a friendly alias as the name clients call. Leave the wire model id unchanged.

Chat and embeddings work on that OpenAI base.

Rerank does not, until the gateway is fixed. Qwen-style rerankers on this API expect `documents` as a JSON array of strings. Bifrost's OpenAI encoder `ToOpenAIRerankRequest` emits `[{"text":"..."}]`, and the service returns 400. A gateway that forwards strings returns 200. Do not change the upstream rerank service to match the object shape.

Fix one of these in the gateway and rebuild:

- Marshal plain-text documents as JSON strings. A document that is only text (no id, structured data, or metadata) must be a JSON string. Keep an object when id, data, or metadata is present.
- Or send rerank through the vLLM rerank encoder when that path already serializes documents as strings.

Prove it with one rerank call that returns 200 and scores. A 400 that mentions `documents` is still this bug.

## Keys and access

`GenerateVirtualKey` prefixes every new key with the compiled `VirtualKeyPrefix`. The upstream default is `sk-bf-`. Any other prefix requires editing `VirtualKeyPrefix` and rebuilding. Config cannot change the prefix of keys generated after that.

On load, `governance.virtual_keys[].value` is kept only when it already starts with `sk-bf-`. Any other prefix is dropped.

OSS cannot create email sub-users: `POST /api/users` returns 405. There is one dashboard admin. Teams and virtual keys are the access model (model allow-list, RPM, TPM). Give API callers a virtual key only. Do not share the admin login when the caller should only use the API.

## Observability

- Prometheus: `GET /metrics`. Series are `bifrost_*` and `http_*`.
- Log stats: `GET /api/logs/stats`. Totals, token totals, and average latency.
- Histogram buckets expose p90, p95, and p99. They are not a windowed p50.
- There is no 429 aggregate in these stats.
- Cloud Eye (CES) for a v2 realtime service id is often empty. Empty CES does not mean the service is idle.
- Queue depth, KV-cache use, and per-stage latency are vLLM `ms_service_metric` / Prometheus. That scrape is future work. Do not invent those series on the gateway.

## Upgrade

1. Record the running image tag and `git rev-parse HEAD`.
2. Fetch tags. Compare the candidate tag to the current commit before any checkout.
3. If the working tree is already newer than the tag, stop. Do not checkout the tag. Do not downgrade.
4. If the tag is newer, checkout or build from it. Pass `VERSION`.
5. Recreate the container with the same `/var/lib/bifrost` mount. Never `docker volume rm` and never delete that directory.
6. A tag checkout drops uncommitted work. Re-apply local UI patches, then rebuild. The embedded UI will not show those patches until the image is rebuilt.
7. Confirm `/api/version` and that an existing virtual key still authenticates.

## Customization, only when asked

These are generic product changes. Each UI change requires a rebuild because the UI is embedded.

- Brand tokens and a logo in the embedded UI.
- Hide enterprise-only navigation when `IS_ENTERPRISE` is false.
- A model catalog page that shows one call-name per model.
- Dashboard labels that map wire ids to call names, including models with zero traffic.
