# Global 24/7 Streaming: vMix → OvenMediaEngine → Bunny CDN

This runbook publishes one 720p live stream from vMix to OvenMediaEngine (OME) over encrypted SRT, then delivers LL-HLS worldwide through Bunny CDN.

```text
vMix PC -- encrypted SRT --> OME -- private Docker network --> Caddy -- HTTPS --> Bunny CDN --> global viewers
```

OME's playback port is private. Caddy accepts stream requests only when Bunny supplies a secret request header, and Bunny token authentication can protect the viewer URL.

> This is LL-HLS, not legacy HLS. The playback playlist is `master.m3u8`; do not use `index.m3u8` or a `ts:` URL.

## Command locations

Every instruction states where it is performed:

| Label | Where to perform it |
|---|---|
| **Cloudflare dashboard** | Cloudflare zone for `cockxing.online`; used only for DNS records in this design |
| **Google Cloud Console** | Google Cloud Console pages for Compute Engine, IAM, and VPC firewall rules |
| **Google Cloud Shell** | Cloud Shell opened from the Google Cloud Console; used for the `gcloud` provisioning commands |
| **Administrator workstation** | Administrator's local terminal with Google Cloud CLI authenticated to the project; used to open the OS Login SSH session |
| **Origin SSH terminal** | SSH session to the Ubuntu VM, signed in as a sudo-capable deployment user |
| **Bunny dashboard** | `https://panel.bunny.net/` |
| **vMix PC** | Windows computer running vMix |
| **External test device** | A network other than the origin VM; ideally test locations in each target region |

## 1. Plan the service

**Location: your planning workstation. No command is required.**

1. Place the origin VM close to the vMix encoder. This keeps the real-time SRT contribution path short; Bunny's CDN handles global viewer delivery.
2. For one bypassed 720p stream, start with 2 vCPU, 4 GB RAM, a 1 Gbps NIC, and Ubuntu 24.04 LTS. Load-test before an event.
3. Use Bunny Standard with all delivery regions enabled for the best worldwide coverage. A distant edge can still have greater live latency on cache misses because it must contact the origin.
4. Plan a warm standby only if global availability is required. It needs a second encoder contribution path, its own origin name, and a rehearsed player-DNS/traffic-manager failover procedure.

At 3,000 Kbps video plus 128 Kbps audio, one continuous viewer transfers about **1.41 GB/hour**. At 1,000 concurrent viewers for 30 days, delivery is approximately **1.01 PB/month** before overhead.

| Viewer region | Bunny Standard price | 1.01 PB if all traffic is in that region |
|---|---:|---:|
| Europe & North America | $0.01/GB | ~$10,080/month |
| Asia & Oceania | $0.03/GB | ~$30,240/month |
| South America | $0.045/GB | ~$45,360/month |
| Middle East & Africa | $0.06/GB | ~$60,480/month |

Calculate the expected bill from the expected traffic share in each region. For an even four-region split, the baseline is about **$36,540/month**. Verify live pricing before committing; Bunny Volume is cheaper but has fewer PoPs and needs audience testing.

## 2. Create the Google Cloud origin VM

### 2.1 Select the Google Cloud region and secure administrator access

**Location: Google Cloud Console. No command is required.**

1. Select the Google Cloud project that will own the origin. Record its project ID.
2. Choose a Compute Engine region nearest the vMix PC, then choose a zone in that region. Example: `asia-southeast1` and `asia-southeast1-a`. The example values below must be replaced with your chosen region and zone.
3. Enable **OS Login** for the instance and grant each administrator `roles/compute.osAdminLogin`. OS Login ties SSH access to Google IAM rather than persistent metadata SSH keys.
4. If your organization supports it, enable OS Login two-step verification. Ensure every administrator has Google two-step verification configured before enforcing it.
5. Determine and record the public IPv4 addresses/CIDR ranges for the administrator workstation and the vMix site.

### 2.2 Reserve the static public IP and create the VM

**Location: Google Cloud Shell. Working directory: Cloud Shell home directory.**

1. Open Cloud Shell and set the following variables. Replace every all-uppercase placeholder before executing.

```bash
PROJECT_ID="YOUR_GOOGLE_CLOUD_PROJECT_ID"
REGION="YOUR_REGION"
ZONE="YOUR_ZONE"
NETWORK="default"
INSTANCE="ome-origin01"
TAG="ome-origin"
ADMIN_CIDR="YOUR_ADMIN_PUBLIC_IP/32"
VMIX_CIDR="YOUR_VMIX_PUBLIC_IP/32"
gcloud config set project "$PROJECT_ID"
```

2. Reserve a regional static IPv4 address. This prevents a VM restart from changing the origin address.

```bash
gcloud compute addresses create ome-origin01-ip --region="$REGION"
gcloud compute addresses describe ome-origin01-ip --region="$REGION" --format='get(address)'
```

3. Create the VM. `e2-medium` provides 2 vCPU and 4 GB RAM; increase capacity after load testing. The VM has no attached service account because this deployment does not require access to Google APIs from inside the VM.

```bash
gcloud compute instances create "$INSTANCE" \
  --zone="$ZONE" \
  --machine-type="e2-medium" \
  --image-family="ubuntu-2404-lts-amd64" \
  --image-project="ubuntu-os-cloud" \
  --boot-disk-size="50GB" \
  --boot-disk-type="pd-balanced" \
  --address="ome-origin01-ip" \
  --tags="$TAG" \
  --metadata="enable-oslogin=TRUE" \
  --no-service-account \
  --no-scopes
```

4. Confirm that the instance is running and note its fixed external IP:

```bash
gcloud compute instances describe "$INSTANCE" --zone="$ZONE" \
  --format='get(status,networkInterfaces[0].accessConfigs[0].natIP)'
```

### 2.3 Create Google Cloud VPC firewall rules

**Location: Google Cloud Shell. Working directory: Cloud Shell home directory.**

1. Create target-tagged rules. Only this VM is affected because it has the `ome-origin` network tag.

| Rule | Source | Action |
|---|---|---|
| TCP 22 | administrator public IP or VPN CIDR only | Allow |
| TCP 80 | Internet | Allow |
| TCP 443 | Internet | Allow |
| UDP 9999 | vMix public IP or VPN CIDR only | Allow |
| Any other inbound traffic | Any | Deny |

```bash
gcloud compute firewall-rules create ome-origin01-allow-ssh \
  --network="$NETWORK" --direction=INGRESS --action=ALLOW --priority=1000 \
  --target-tags="$TAG" --source-ranges="$ADMIN_CIDR" --rules=tcp:22

gcloud compute firewall-rules create ome-origin01-allow-web \
  --network="$NETWORK" --direction=INGRESS --action=ALLOW --priority=1000 \
  --target-tags="$TAG" --source-ranges="0.0.0.0/0" --rules=tcp:80,tcp:443

gcloud compute firewall-rules create ome-origin01-allow-srt \
  --network="$NETWORK" --direction=INGRESS --action=ALLOW --priority=1000 \
  --target-tags="$TAG" --source-ranges="$VMIX_CIDR" --rules=udp:9999
```

2. Do not create rules for TCP 1935 or TCP 3333.
3. Review existing rules before proceeding. In a default VPC, `default-allow-ssh` can allow SSH from the internet to every VM. Do not delete it without checking its impact on other VMs; instead ensure it does not target this origin or replace it through your approved network-change process.

```bash
gcloud compute firewall-rules list --filter="network:$NETWORK" \
  --format="table(name,direction,priority,sourceRanges.list():label=SOURCE,allowed[].map().firewall_rule().list():label=ALLOW,targetTags.list():label=TARGET_TAGS)"
```

### 2.4 Create the origin DNS record in Cloudflare

**Location: Cloudflare dashboard → `cockxing.online` → DNS → Records.**

1. Select **Add record**.
2. Set **Type** to `A`, **Name** to `origin01`, and **IPv4 address** to the reserved static IP displayed in step 2.2.
3. Set **Proxy status** to **DNS only** (grey cloud), then save. Do not orange-cloud this record: Bunny must reach Caddy directly and Caddy must complete its own TLS challenge.
4. Set a temporary TTL such as 300 seconds while testing, if Cloudflare exposes TTL for this DNS-only record.
5. Understand the trade-off: DNS-only reveals the GCP origin IP. The Caddy secret-header gate and the restricted SRT firewall rule remain essential; do not use this VM for unrelated public services.
6. `player01.cockxing.online` is created later as a Bunny CNAME; do not point it to the VM.

## 3. Prepare and secure the Ubuntu origin

**Location: Administrator workstation first, then Origin SSH terminal. Working directory: home directory (`~`) after connecting.**

1. From the administrator workstation, connect with OS Login. The workstation's public address must be included in `ADMIN_CIDR` in the Google Cloud firewall rule.

```bash
gcloud compute ssh ome-origin01 --zone=YOUR_ZONE --project=YOUR_GOOGLE_CLOUD_PROJECT_ID
```

2. Continue in the resulting **Origin SSH terminal**. Install Docker and required utilities. Do not run a blanket OS upgrade during a production build.

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin curl openssl ufw
sudo systemctl enable --now docker
sudo install -d -m 750 "$HOME/ome/config" "$HOME/ome/caddy"
```

3. Optionally add host-level UFW rules as defence in depth. The cloud firewall remains the authoritative control because Docker networking can bypass host firewall expectations. Replace the two placeholders before executing.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from ADMIN_PUBLIC_IP to any port 22 proto tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from VMIX_PUBLIC_IP to any port 9999 proto udp
sudo ufw enable
sudo ufw status numbered
```

4. Generate the origin-header secret and save it with owner-only access. Record the displayed value in a password manager; it will be used in Caddy and Bunny.

```bash
openssl rand -hex 32 | tee "$HOME/ome/caddy/origin-header-secret.txt"
chmod 600 "$HOME/ome/caddy/origin-header-secret.txt"
```

5. Confirm the resulting files and move to the deployment directory:

```bash
ls -la "$HOME/ome/caddy"
cd "$HOME/ome"
pwd
```

## 4. Create the OME configuration

**Location: Origin SSH terminal. Working directory: `~/ome`.**

1. Create the configuration file:

```bash
nano ~/ome/config/Server.xml
```

2. Paste the following XML, save with `Ctrl+O`, press `Enter`, and exit with `Ctrl+X`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server version="8">
  <Name>OvenMediaEngine</Name>
  <Type>origin</Type>
  <IP>*</IP>
  <PrivacyProtection>true</PrivacyProtection>
  <Modules>
    <HTTP2><Enable>true</Enable></HTTP2>
    <LLHLS><Enable>true</Enable></LLHLS>
  </Modules>
  <Bind>
    <Providers>
      <SRT><Port>9999</Port><WorkerCount>1</WorkerCount></SRT>
    </Providers>
    <Publishers>
      <LLHLS><Port>3333</Port><WorkerCount>1</WorkerCount></LLHLS>
    </Publishers>
  </Bind>
  <VirtualHosts>
    <VirtualHost>
      <Name>default</Name>
      <Host><Names><Name>*</Name></Names></Host>
      <Applications>
        <Application>
          <Name>app</Name>
          <Type>live</Type>
          <OutputProfiles>
            <OutputProfile>
              <Name>bypass</Name>
              <OutputStreamName>${OriginStreamName}</OutputStreamName>
              <Encodes>
                <Video><Bypass>true</Bypass></Video>
                <Audio><Bypass>true</Bypass></Audio>
              </Encodes>
            </OutputProfile>
          </OutputProfiles>
          <Providers><SRT /></Providers>
          <Publishers>
            <LLHLS>
              <ChunkDuration>0.5</ChunkDuration>
              <PartHoldBack>1.5</PartHoldBack>
              <SegmentDuration>6</SegmentDuration>
              <SegmentCount>10</SegmentCount>
              <CrossDomains><Url>https://player01.cockxing.online</Url></CrossDomains>
            </LLHLS>
          </Publishers>
        </Application>
      </Applications>
    </VirtualHost>
  </VirtualHosts>
</Server>
```

3. Verify the file exists and contains the required LL-HLS and SRT sections:

```bash
grep -nE 'Server version|<SRT>|<LLHLS>|<Name>app' ~/ome/config/Server.xml
```

OME bypasses transcoding. vMix must therefore send H.264 video and AAC audio. LL-HLS latency is mainly governed by `ChunkDuration` and the player live buffer, not by making segments arbitrarily short.

## 5. Configure Caddy and Docker Compose

**Location: Origin SSH terminal. Working directory: `~/ome`.**

1. Display the secret and copy it only into the next configuration and Bunny; do not paste it into chat, source control, screenshots, or a public ticket.

```bash
cat ~/ome/caddy/origin-header-secret.txt
```

2. Create the Caddy configuration:

```bash
nano ~/ome/caddy/Caddyfile
```

3. Paste the following, replacing `PASTE_THE_SECRET_FROM_THE_FILE` with the copied value. Save and exit.

```caddyfile
origin01.cockxing.online {
    @bunny header X-Origin-Verify "PASTE_THE_SECRET_FROM_THE_FILE"
    handle @bunny {
        reverse_proxy ome:3333
    }
    respond "Not found" 404
}
```

4. Create the Compose file:

```bash
nano ~/ome/docker-compose.yml
```

5. Paste, save, and exit:

```yaml
services:
  ome:
    image: ovenmedialabs/ovenmediaengine:v0.20.5
    restart: unless-stopped
    ports:
      - "9999:9999/udp"
    volumes:
      - ./config/Server.xml:/opt/ovenmediaengine/bin/origin_conf/Server.xml:ro
    networks: [streaming]

  caddy:
    image: caddy:2.10.2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks: [streaming]

networks:
  streaming:

volumes:
  caddy_data:
  caddy_config:
```

6. Validate the Compose syntax, start both containers, and check their state:

```bash
cd ~/ome
sudo docker compose config --quiet
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 ome caddy
```

Expected result: both services show as running. Caddy obtains a TLS certificate for `origin01.cockxing.online`; if it does not, first confirm the DNS A record and cloud firewall rules for ports 80 and 443.

7. Confirm the origin is not publicly playable.

**Location: External test device.**

```bash
curl -I https://origin01.cockxing.online/app/linear/master.m3u8
```

Expected result: `404`, not a playlist. Do not continue if it returns stream content.

## 6. Configure vMix SRT contribution

**Location: vMix PC.**

1. Open your vMix production preset.
2. Click the gear icon beside **Stream**, choose an unused destination, and select **SRT** with mode **Caller**.
3. Enter these values. If your vMix release exposes encryption in a different pane, enable it there.

| Setting | Value |
|---|---|
| Host | origin VM public IP or a dedicated ingest DNS name |
| Port | `9999` |
| Stream ID | `default/app/linear` |
| Latency | Start at `120 ms`; increase if WAN loss or jitter appears |
| Encryption | Enable with a strong passphrase; use 32-byte key length where available |
| Video | H.264, 1280x720, 30 fps, 3,000 Kbps CBR |
| Audio | AAC-LC, 128 Kbps |
| Keyframe interval | 2 seconds |

4. If vMix accepts a single SRT URL, use this structure; replace all-uppercase placeholders and retain the `streamid` query parameter:

```text
srt://GOOGLE_CLOUD_STATIC_IP:9999?streamid=default/app/linear&passphrase=YOUR_SRT_SECRET&pbkeylen=32
```

5. Enable vMix's automatic reconnect option, save the preset, and start streaming.
6. Return to the **Origin SSH terminal** and confirm OME sees the ingest:

```bash
cd ~/ome
sudo docker compose logs --tail=100 -f ome
```

After the stream appears, stop following logs with `Ctrl+C`. The origin still returns 404 externally until Bunny is configured with the secret header.

## 7. Configure Bunny CDN

### 7.1 Create the Pull Zone

**Location: Bunny dashboard.**

1. Open **CDN → Pull Zones → Add Pull Zone**.
2. Name it `player01` (or another descriptive, unique name).
3. Select an **Origin URL** and enter `https://origin01.cockxing.online`.
4. Choose the **Standard** tier.
5. Enable all delivery regions: Europe/North America, Asia/Oceania, South America, and Middle East/Africa.
6. Create the zone, then set **Origin Host Header** to `origin01.cockxing.online` and leave origin SSL verification enabled.
7. Enable Origin Shield in a location close to the origin. Measure global playback; turn it off only if it demonstrably harms the required live latency.
8. Enable request coalescing, block root-path access, block POST requests, and set a monthly bandwidth limit/alert.

### 7.2 Configure the Bunny-to-origin secret header

**Location: Bunny dashboard.**

1. In the Pull Zone, open **Edge Rules** and add a rule that applies to every request.
2. Select the action **Set Request Header**.
3. Set header name to `X-Origin-Verify`.
4. Set the value to the exact value from `~/ome/caddy/origin-header-secret.txt`.
5. Save and enable the rule.
6. The value is a secret. Do not put it in browser code or configure it as a response header.

### 7.3 Add the custom playback hostname

**Location: Bunny dashboard, then Cloudflare dashboard → `cockxing.online` → DNS → Records.**

1. In the Pull Zone custom-hostname area, add `player01.cockxing.online`.
2. Bunny displays the CNAME target required for this zone. Copy that exact target.
3. In **Cloudflare**, select **Add record**, set **Type** to `CNAME`, **Name** to `player01`, and **Target** to Bunny's displayed target. Do not guess the target and do not point it to the origin VM.
4. Set **Proxy status** to **DNS only** (grey cloud) and save. Orange-clouding a CNAME to another CDN can cause connectivity or certificate problems and would insert Cloudflare in front of Bunny.
5. Return to Bunny and wait until the hostname certificate is active.
6. Confirm DNS and TLS from an **External test device**:

```bash
curl -I https://player01.cockxing.online/
```

A 403/404 at `/` is expected because root access is blocked; the important result is a valid HTTPS connection.

### 7.4 Set caching and access policy

**Location: Bunny dashboard.**

1. Add an extension cache rule for `.m3u8`: edge cache TTL `0`/do not cache and browser cache `no-store`.
2. Add an extension cache rule for `.m4s`: edge and browser cache TTL `10 minutes`.
3. Do not globally ignore query strings if token authentication will be used.
4. For a non-public stream, enable **Advanced Token Authentication**. Generate short-lived, path-based directory tokens for `/app/linear/` in your server-side application. Path-based tokens allow the playlist's relative segment requests to inherit authorization.
5. Treat allowed-referrer settings as an additional deterrent only, not as authentication.

## 8. Secure embedding on another website

Use signed URLs as the access control. CORS only permits a browser to read cross-origin media responses; it does **not** stop somebody from copying a public stream URL.

### 8.1 Enable viewer authentication

**Location: Bunny dashboard.**

1. Open **CDN → Pull Zones → `player01` → Security**.
2. Enable **Advanced Token Authentication** and copy the URL Token Authentication Key.
3. Store that key only in the secret manager or server-side environment variables for the application that authorizes viewers. Never add it to an HTML page, browser JavaScript bundle, CMS setting visible to editors, or a partner website.
4. Keep the existing rule that does not globally ignore query strings. The token signer must control the parameters in the playback URL.

### 8.2 Issue a short-lived, directory token from your website backend

**Location: the backend/service that authenticates the viewer; not the visitor's browser.**

1. Authenticate the viewer and check that they are entitled to watch before issuing any stream URL.
2. Use Bunny's Advanced Token Authentication signer with these inputs:

| Signer setting | Value |
|---|---|
| Base URL | `https://player01.cockxing.online/app/linear/master.m3u8` |
| Token type | Path-based directory token |
| Token path | `/app/linear/` |
| Expiry | 5–15 minutes, based on your viewing-session design |
| Signing key | The server-side Bunny URL Token Authentication Key |

3. Return the resulting signed playback URL only to the authorized viewer. Its path starts with `bcdn_token=...`; this is required so the playlist's relative `.m4s` requests automatically inherit the token.
4. Log the entitlement decision and token expiry, but never log the full signed URL or signing key.
5. Do not enable IP locking by default. It can interrupt users on mobile networks, VPNs, or networks that change address. If you enable it, sign with the viewer's IPv4 address and test that all intended users can play.

### 8.3 Permit the embedding website through CORS

**Location: Origin SSH terminal. Working directory: `~/ome`.**

1. Identify the exact origin of the website that embeds the player, for example `https://www.partner.example`. The scheme and hostname must match exactly.
2. Edit OME's configuration:

```bash
nano ~/ome/config/Server.xml
```

3. Under `<CrossDomains>`, add one `<Url>` entry for each permitted website. Keep `player01.cockxing.online` if it hosts a player page; add the partner site separately:

```xml
<CrossDomains>
  <Url>https://player01.cockxing.online</Url>
  <Url>https://www.partner.example</Url>
</CrossDomains>
```

4. Save the file and restart only OME:

```bash
cd ~/ome
sudo docker compose restart ome
sudo docker compose logs --tail=100 ome
```

5. Test playback from the partner site. A browser CORS error means the embedding page's actual origin was not allowlisted. A `403` means the signed token, expiry, or optional hotlink policy is wrong.

### 8.4 Optional hotlink protection

**Location: Bunny dashboard.**

1. In the Pull Zone's **Security** settings, add the embedding domains to **Allowed Referrers** without a scheme, for example `partner.example` and `www.partner.example`.
2. Add every legitimate embed domain. `*.partner.example` does not include `partner.example`, so add both if both are used.
3. Consider enabling **Block Direct URL File Access** only after testing mobile apps, privacy browsers, email links, and casting. They can legitimately omit a `Referer` header.
4. Treat hotlink protection as a cost-control layer, not authentication: referrers can be absent or spoofed. Keep signed tokens enabled for protected video.

## 9. Block a country

Country blocking is enforced by Bunny using the viewer's IP geolocation. It applies to the whole Pull Zone, so it blocks both the playlist and media fragments.

**Location: Bunny dashboard.**

1. Open **CDN → Pull Zones → `player01`** and locate the **Blocked Countries** setting under Security or Geographic restrictions.
2. Add the country to block using its ISO 3166-1 alpha-2 code. For example, use `XX` only as a placeholder—replace it with the real two-letter country code before saving.
3. Save and enable the setting. Do not disable the corresponding global delivery region; disabling a delivery region can reroute the user rather than block them.
4. Test with a reputable exit node in the blocked country and one in an allowed country. Confirm the blocked viewer receives `403` for both `master.m3u8` and `.m4s` requests.
5. Document the business/legal reason, approver, date, and review date. IP geolocation is not a substitute for legal advice and can be bypassed by VPNs.

If different viewers need different country access, do not use Pull-Zone-wide blocking. Instead, have the server-side Advanced Token signer include `token_countries` or `token_countries_blocked` for that viewer's directory token.

## 10. Verify playback

**Location: External test device.**

1. Open this URL in an LL-HLS capable player or your test webpage:

```text
https://player01.cockxing.online/app/linear/master.m3u8
```

2. Confirm video and audio begin, then inspect the browser/player network requests. The playlist and `.m4s` media requests must use `player01.cockxing.online`, never `origin01.cockxing.online`.
3. Test from every significant viewer region and record startup time, rebuffer count, and live latency.
4. If CORS errors occur, replace `https://player01.cockxing.online` in OME's `<CrossDomains>` section with the exact origin of the webpage that embeds the player, then restart OME:

```bash
cd ~/ome
sudo docker compose restart ome
```

**Location: Origin SSH terminal for the restart command above.**

## 11. Operate safely

**Location: Origin SSH terminal for the commands; Bunny dashboard for CDN metrics.**

Run these checks after deployment and during an incident:

```bash
cd ~/ome
sudo docker compose ps
sudo docker compose logs --tail=200 ome
sudo docker compose logs --tail=200 caddy
sudo docker stats --no-stream
```

Monitor vMix output, SRT loss/retransmissions, OME logs, CPU/RAM/network, Bunny cache-hit ratio, 4xx/5xx rates, delivery by region, and playback probes from target continents. Rehearse encoder restart, VM restart, certificate renewal, token expiry, and primary-to-standby failover before a live event.

## References

- [Google Cloud OS Login](https://cloud.google.com/compute/docs/oslogin)
- [Google Cloud VPC firewall rules](https://cloud.google.com/firewall/docs/using-firewalls)
- [Google Cloud firewall-rule CLI reference](https://cloud.google.com/sdk/gcloud/reference/compute/firewall-rules/create)
- [Cloudflare proxy status](https://developers.cloudflare.com/dns/proxy-status/)
- [Cloudflare proxying limitations](https://developers.cloudflare.com/dns/proxy-status/limitations/)
- [OME configuration](https://ovenmedia.com/docs/ome/configuration)
- [OME SRT ingest](https://ovenmedia.com/docs/ome/live-source/srt)
- [OME LL-HLS](https://docs.ovenmediaengine.com/0.17.2/streaming/low-latency-hls)
- [Bunny CDN pricing](https://docs.bunny.net/cdn/pricing)
- [Bunny advanced token authentication](https://docs.bunny.net/cdn/security/token-authentication/advanced)
- [Bunny hotlink protection](https://docs.bunny.net/cdn/security/hotlink-protection)
- [Bunny Origin Shield](https://docs.bunny.net/cdn/performance/origin-shield)
