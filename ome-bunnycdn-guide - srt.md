# Global 24/7 Streaming: vMix -> OvenMediaEngine -> Bunny CDN

This runbook deploys one 720p live stream from vMix to OvenMediaEngine (OME) using encrypted SRT, then delivers Low-Latency HLS (LL-HLS) through Bunny CDN.

Follow the sections in order. Each phase ends with a validation check; do not start the next phase until that check passes.

## 1. Target design and security model

```text
vMix PC -- encrypted SRT/UDP --> OME -- private Docker network --> Caddy -- HTTPS --> Bunny CDN --> viewers
```

The stream path has three trust boundaries:

| Boundary | Control |
|---|---|
| vMix -> origin | SRT encryption plus a cloud-firewall rule that admits UDP 9999 only from the vMix public IP/VPN range. |
| OME -> Internet | OME's LL-HLS port stays inside Docker. It is never published on the VM. |
| Bunny -> origin | Caddy proxies only requests that contain a secret `X-Origin-Verify` request header set by Bunny. |
| Bunny -> viewers | Optional Bunny Advanced Token Authentication provides short-lived signed viewer URLs. |

Important:

- This design uses LL-HLS. With the pinned OME image, the playback playlist is `llhls.m3u8`; do not use `index.m3u8` or a `ts:` URL.
- OME does no transcoding. vMix must produce H.264 video and AAC audio.
- The Cloudflare origin record is **DNS only**. This exposes the origin IP, so use this VM only for this streaming service and retain every firewall/header control below.
- CDN delivery is global, but the origin should be close to the vMix encoder. Distance to the origin can increase live latency on an edge-cache miss.

## 2. Deployment worksheet

Complete this table before running commands. Commands deliberately use placeholders so a value is never silently guessed.

| Item | Example / required value |
|---|---|
| Origin DNS name | `origin01.cockxing.online` |
| Viewer DNS name | `player01.cockxing.online` |
| Akamai region | Nearest region to vMix, for example `ap-south` |
| VM label | `ome-origin01` |
| VM plan | `g6-standard-2` (2 vCPU, 4 GB RAM) to start |
| Administrator CIDR | Your fixed public IPv4/VPN range, for example `203.0.113.10/32` |
| vMix CIDR | vMix site public IPv4/VPN range, for example `198.51.100.20/32` |
| vMix SRT passphrase | A separately stored strong secret |
| Partner embed origins | Exact HTTPS origins, for example `https://www.partner.example` |
| Bunny token key | Created in Bunny and stored only in the authorizing backend |

### Capacity and cost planning

For one bypassed 720p stream, begin with 2 vCPU, 4 GB RAM, a 1 Gbps NIC, and Ubuntu 24.04 LTS. Load-test with the real encoder and player before an event. Plan a warm standby only when the service needs global high availability; it requires a second contribution path, another origin name, and a rehearsed traffic/DNS failover plan.

At 3,000 Kbps video plus 128 Kbps audio, one continuous viewer transfers about **1.41 GB/hour**. At 1,000 concurrent viewers for 30 days, delivery is approximately **1.01 PB/month** before overhead.

| Viewer region | Bunny Standard price | Approximate cost for 1.01 PB entirely in that region |
|---|---:|---:|
| Europe & North America | $0.01/GB | ~$10,080/month |
| Asia & Oceania | $0.03/GB | ~$30,240/month |
| South America | $0.045/GB | ~$45,360/month |
| Middle East & Africa | $0.06/GB | ~$60,480/month |

Calculate the bill from the expected regional traffic mix. An even four-region split has a baseline near **$36,540/month**. Verify current Bunny pricing before committing; Bunny Volume may cost less but has fewer PoPs and requires audience testing.

## 3. Where each action happens

| Location label | Use it for |
|---|---|
| **Cloudflare dashboard** | DNS records in the `cockxing.online` zone only. |
| **Akamai Cloud Manager** | Account, region, Cloud Firewall, reserved IP, SSH-key, and VM configuration. |
| **Administrator workstation** | A local terminal used to create/access the SSH key and connect to the origin. No Linode CLI is required. |
| **Origin SSH terminal** | An SSH session to the Ubuntu VM, initially as `root` and then as `deploy`. |
| **Bunny dashboard** | Pull Zone, custom hostname, Edge Rules, token authentication, and metrics. |
| **vMix PC** | The Windows machine running vMix. |
| **External test device** | A device/network other than the origin, ideally in each viewer region. |

## 4. Prerequisites and readiness check

Before provisioning, confirm all of the following:

- You control the `cockxing.online` DNS zone in Cloudflare.
- You have an Akamai Cloud account and an administrator SSH public key that you can add to Cloud Manager.
- You know the administrator and vMix source public IP ranges. If either location uses changing IPs, use a fixed VPN egress range instead.
- Ports 80 and 443 may be publicly reached during Caddy certificate issuance and by Bunny. UDP 9999 may be reached only from the vMix CIDR.
- You have a Bunny account and authority to create a Pull Zone and custom hostname.
- If the stream is private, the viewer website has a server-side backend/secret manager able to issue signed Bunny URLs. Browser JavaScript is not sufficient because it would expose the signing key.

## 5. Phase 1 - Create the origin VM and public DNS

### 5.1 Choose the region and prepare administrator access

**Location: Akamai Cloud Manager and Administrator workstation**

1. Sign in to [Akamai Cloud Manager](https://cloud.linode.com/) and select the account that will own the origin.
2. Choose the region nearest vMix. The origin should be close to the encoder; Bunny handles global viewer delivery.
3. Create or locate an administrator SSH key pair on the workstation. Keep the private key only on the workstation. The public key normally has a `.pub` suffix, for example `id_ed25519.pub`.

   **Windows administrator workstation - create a new key pair**

   1. Open PowerShell and run the following command. Replace the comment value with an identifier for the key owner/workstation:

      ```powershell
      ssh-keygen -t ed25519 -a 100 -C "your-name@admin-workstation"
      ```

   2. At the file-location prompt, press `Enter` to use the default path, normally `C:\Users\<your-user>\.ssh\id_ed25519`. If that file already exists, do **not** overwrite it unless you intentionally want to replace that key.
   3. Enter and confirm a strong passphrase when prompted.
   4. Display the public key and copy the complete single line that begins with `ssh-ed25519`:

      ```powershell
      Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"
      ```

   5. Copy only the `.pub` file contents. Never upload, paste, or share the matching `id_ed25519` file without the `.pub` suffix; it is the private key.

4. In Cloud Manager, select your profile name -> **SSH Keys** -> **Add an SSH Key**. Give the key a descriptive label, paste the public-key contents copied above, and save it. You will select this key during VM creation.
5. Record the administrator and vMix IPv4 CIDRs in the deployment worksheet. A single IPv4 address must use `/32`, for example `203.0.113.10/32`.

### 5.2 Create the Cloud Firewall

**Location: Akamai Cloud Manager**

1. From the navigation menu, open **Firewalls**, choose **Create Firewall**, and name it `ome-origin01-fw`.
2. On the firewall's **Rules** page, set the default **Inbound Policy** to **Drop** and the default **Outbound Policy** to **Accept**.
3. Add the three inbound **Accept** rules in the table below. For a public web rule, use the UI's all-IPv4 and all-IPv6 source option, or enter `0.0.0.0/0` and `::/0` if it requests CIDRs. Use the exact administrator and vMix CIDRs from the worksheet for the restricted rules.

| Protocol/port | Source | Purpose |
|---|---|---|
| TCP 22 | Administrator CIDR only | Administration |
| TCP 80, 443 | Internet | Caddy certificate validation and Bunny origin pulls |
| UDP 9999 | vMix CIDR only | SRT contribution |
| All other inbound | Any | Deny |

4. Click **Save Changes** after adding the rules. Cloud Firewall rule changes do not take effect until they are saved.
5. Do not add rules for TCP 1935 or TCP 3333. RTMP is not used and OME's playback port must stay private.

### 5.3 Reserve an IPv4 address and create the VM

**Location: Akamai Cloud Manager**

1. Open **Reserved IPs** from the navigation menu, then choose **Reserve an IP Address**.
2. Select the same region chosen for the VM, optionally add the `ome-origin` tag, and reserve the address. A reserved IP is billed while it remains reserved, including when unassigned, but remains available to your account if the VM is later deleted.
3. Copy the assigned address into the worksheet as `ORIGIN_IPV4`.
4. Open the **Create** menu in the top bar and select **Linode**.
5. Configure the form as follows:

| Create-Linode setting | Value |
|---|---|
| Region | The region chosen in step 5.1 |
| Image | Ubuntu 24.04 LTS |
| Plan | A Shared CPU plan with 2 vCPU and 4 GB RAM (the API plan is `g6-standard-2`) |
| Label | `ome-origin01` |
| Tags | `ome-origin` |
| SSH keys | Select the administrator key added in step 5.1 |
| Public Internet / IPv4 | Choose **Reserved** and select `ORIGIN_IPV4` |
| Cloud Firewall | Select `ome-origin01-fw` for the public interface |

6. Leave a root password blank when Cloud Manager permits password-less provisioning with an SSH key. This is preferred. Do not add a Quick Deploy app, StackScript, or unrelated interface.
7. Click **Create Linode** and wait until its status is **Running**.
8. Open the new Linode's detail page and confirm its public IPv4 is the reserved `ORIGIN_IPV4`, its region and plan are correct, and `ome-origin01-fw` is attached to the public interface.

### 5.4 Create the origin DNS record

**Location: Cloudflare dashboard -> `cockxing.online` -> DNS -> Records**

1. Choose **Add record**.
2. Create an `A` record with **Name** `origin01` and the reserved `ORIGIN_IPV4` address from step 5.3.
3. Set **Proxy status** to **DNS only** (grey cloud), then save. Do not orange-cloud it: Bunny must connect to Caddy directly, and Caddy must complete its own TLS challenge.
4. Use a temporary TTL such as 300 seconds while testing, if Cloudflare presents that option.
5. Do not create `player01` here yet. It is created later as a Bunny-directed CNAME.

**Phase 1 gate:** From the administrator workstation, confirm DNS resolves to the reserved address and SSH is reachable:

```bash
ssh root@origin01.cockxing.online
```

If SSH fails, verify that the workstation's current public IP is inside the administrator CIDR and that the Cloud Firewall is attached to the VM's public interface.

> **SSH key troubleshooting:** The initial account is `root`; the `deploy` account does not exist until Phase 2. SSH uses the private key matching the administrator public key selected when creating the VM, so no root password is required. If Windows returns `Permission denied (publickey)`, confirm the matching key is available with `Get-ChildItem $HOME\.ssh`, then specify it explicitly:
>
> ```powershell
> ssh -i $HOME\.ssh\YOUR_PRIVATE_KEY root@origin01.cockxing.online
> ```
>
> After the deployment-account commands below have completed, use the same key to connect as `deploy`.

## 6. Phase 2 - Prepare and secure Ubuntu

### 6.1 Create a deployment account

**Location: Origin SSH terminal**

If this is the initial `root` login, create a restricted deployment workflow before installing software:

```bash
adduser deploy
usermod -aG sudo deploy
install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
chown deploy:deploy /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys
exit
```

**Location: Administrator workstation**

Reconnect as the deployment user:

```bash
ssh deploy@origin01.cockxing.online
```

### 6.2 Install the runtime and create directories

**Location: Origin SSH terminal. Working directory: home directory (`~`)**

Do not run a blanket OS upgrade during a production build. Install only the required packages and start Docker:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 curl openssl ufw
sudo systemctl enable --now docker
sudo install -d -m 750 -o deploy -g deploy "$HOME/ome/config" "$HOME/ome/caddy"
docker compose version
```

Optional defence in depth: add UFW rules. The Akamai Cloud Firewall remains the authoritative control because Docker networking can bypass host-firewall assumptions. Replace the placeholders before execution.

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

### 6.3 Generate the origin request secret

**Location: Origin SSH terminal**

Generate the random secret that Bunny will send and Caddy will check. Save the displayed value in a password manager; it is needed exactly once in Caddy and once in Bunny.

```bash
openssl rand -hex 32 | tee "$HOME/ome/caddy/origin-header-secret.txt"
chmod 600 "$HOME/ome/caddy/origin-header-secret.txt"
ls -la "$HOME/ome/caddy"
cd "$HOME/ome"
pwd
```

Never put this value in source control, chat, screenshots, browser code, or a public ticket.

### 6.4 Generate the SRT passphrase

**Location: Origin SSH terminal**

Generate a separate passphrase for the encrypted vMix-to-OME SRT connection. You will paste this exact value into OME in Phase 3 and vMix in Phase 5.

```bash
openssl rand -hex 32 | tee "$HOME/ome/config/srt-passphrase.txt"
chmod 600 "$HOME/ome/config/srt-passphrase.txt"
```

Never put this value in source control, chat, screenshots, browser code, or a public ticket.

**Phase 2 gate:** `~/ome/config`, `~/ome/caddy`, and the two mode-`600` secret files exist under the `deploy` account.

## 7. Phase 3 - Configure OME

**Location: Origin SSH terminal. Working directory: `~/ome`**

1. Create the OME server configuration:

```bash
nano ~/ome/config/Server.xml
```

2. Run `cat ~/ome/config/srt-passphrase.txt`, then paste its value in place of `PASTE_THE_SRT_PASSPHRASE` below. Paste the following configuration, then save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).

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
      <SRT>
        <Port>9999</Port>
        <WorkerCount>1</WorkerCount>
        <Options>
          <Option><Key>SRTO_PBKEYLEN</Key><Value>32</Value></Option>
          <Option><Key>SRTO_PASSPHRASE</Key><Value>PASTE_THE_SRT_PASSPHRASE</Value></Option>
        </Options>
      </SRT>
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

3. Confirm the required sections are present:

```bash
grep -nE 'Server version|<SRT>|<LLHLS>|<Name>app' ~/ome/config/Server.xml
```

`ChunkDuration` and the player live buffer are the main LL-HLS latency controls. Do not attempt to reduce latency by making segments arbitrarily short.

## 8. Phase 4 - Configure Caddy and start the containers

**Location: Origin SSH terminal. Working directory: `~/ome`**

### 8.1 Configure the Caddy origin gate

1. Display the secret locally, copy it directly into the next file, and avoid exposing it anywhere else:

```bash
cat ~/ome/caddy/origin-header-secret.txt
```

2. Create the Caddy configuration:

```bash
nano ~/ome/caddy/Caddyfile
```

3. Replace `PASTE_THE_SECRET_FROM_THE_FILE` with the exact copied secret, then save:

```caddyfile
origin01.cockxing.online {
    @bunny header X-Origin-Verify "PASTE_THE_SECRET_FROM_THE_FILE"
    handle @bunny {
        reverse_proxy ome:3333 {
            header_up -X-Origin-Verify
        }
    }
    respond "Not found" 404
}
```

Any request that omits or mismatches the header gets `404`; OME is not directly exposed. Caddy removes the verification header before proxying so the secret is not written into OME request logs.

### 8.2 Create and launch Docker Compose

1. Create the Compose file:

```bash
nano ~/ome/docker-compose.yml
```

2. Paste and save:

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

3. Validate the file, start both containers, and inspect startup logs:

```bash
cd ~/ome
sudo docker compose config --quiet
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 ome caddy
```

Expected result: both services are running and Caddy has obtained a TLS certificate for `origin01.cockxing.online`. If Caddy fails to issue a certificate, check the DNS `A` record plus cloud-firewall access to TCP 80 and 443.

### 8.3 Confirm that the origin is gated

**Location: External test device**

Run this unauthenticated request:

```bash
curl -I https://origin01.cockxing.online/app/linear/llhls.m3u8
```

**Phase 4 gate:** The response is `404`, not a playlist. Do not continue if stream content is publicly available from the origin.

## 9. Phase 5 - Configure vMix SRT contribution

**Location: vMix PC**

1. Open the production preset.
2. Do **not** use the gear beside **Stream**: that dialog configures RTMP destinations and does not offer SRT.
3. In the main vMix window, open **Settings** -> **Outputs / NDI / OMT / SRT** (the tab may be named **Outputs / NDI / SRT** in earlier releases).
4. Choose an output channel, normally **Output 1** for the Program feed, and click its cog/settings button. Tick **Enable SRT**, then set the type to **Caller**.
5. Enter these values, click **OK** to save the output settings, then click the **SRT** button in the main vMix window to start the contribution.

| vMix setting | Value |
|---|---|
| Hostname | Origin VM public IP or a dedicated ingest DNS name |
| Port | `9999` |
| Stream ID | `default/app/linear` |
| Latency | Start at `120 ms`; raise it if WAN loss/jitter appears |
| Encryption | Strong passphrase; use a 32-byte key length where offered |
| Video | H.264, 1280x720, 30 fps, 3,000 Kbps CBR |
| Audio | AAC-LC, 128 Kbps |
| Keyframe interval | 2 seconds |

If your vMix release does not show **Enable SRT**, update to vMix 23 or newer. Configure the individual fields above; vMix's SRT output dialog does not require a single URL.

For reference, the equivalent SRT URL is below. Substitute the Akamai origin IP and SRT secret; keep the `streamid` query parameter.

```text
srt://ORIGIN_STATIC_IP:9999?streamid=default/app/linear&passphrase=YOUR_SRT_SECRET&pbkeylen=32
```

Enable vMix automatic reconnect, save the preset, and start streaming.

**Location: Origin SSH terminal**

Confirm that OME sees the contribution:

```bash
cd ~/ome
sudo docker compose logs --tail=100 -f ome
```

Stop following the log with `Ctrl+C` after the stream appears. The external origin URL must still return `404` because Bunny has not yet supplied the secret header.

**Phase 5 gate:** OME logs show the `default/app/linear` SRT stream and no public origin playlist is available.

## 10. Phase 6 - Configure Bunny CDN

### 10.1 Create the Pull Zone

**Location: Bunny dashboard**

1. In the Bunny dashboard, click **+ Add** in the sidebar, then choose **Pull Zone**. (You can also open **CDN** and select **Add Pull Zone**.)
2. Enter the Pull Zone name `player01`, or another unique alphanumeric name. This creates the temporary hostname `NAME.b-cdn.net`.
3. For **Origin type**, choose **Origin URL**. Enter `https://origin01.cockxing.online` for **Origin URL** and `origin01.cockxing.online` for the optional **Host header**.
4. Choose the **Standard** tier and select the delivery regions required by the audience: Europe/North America, Asia/Oceania, South America, and Middle East/Africa.
5. Click **Add Pull Zone** to create it.
6. Open the new Pull Zone. In **General**, confirm its origin URL and Host header, and retain origin SSL verification.
7. In **Caching**, enable **Request Coalescing**. In the applicable General/Security controls, block root-path access and POST requests, and set a monthly bandwidth limit/alert.
8. Optionally enable **Origin Shield** in a location close to the origin. Measure worldwide playback and disable it only if it demonstrably misses the live-latency requirement.

### 10.2 Add the Bunny-to-origin secret header

**Location: Bunny dashboard**

1. In the Pull Zone, open the **Edge Rules** tab and click **Add Edge Rule**.
2. Set **Action** to **Set Request Header**.
3. Set the header name to `X-Origin-Verify` and the header value to the exact value in `~/ome/caddy/origin-header-secret.txt`.
4. Add a trigger/condition that matches every viewer request. Use **Request URL** with the value `*` (or the UI's equivalent match-all option).
5. Save the rule, confirm it is enabled, and keep it ahead of any conflicting origin-routing rules.

The value must remain a request secret. Never send it as a response header and never include it in browser code.

### 10.3 Add the viewer hostname

**Location: Bunny dashboard, then Cloudflare dashboard -> `cockxing.online` -> DNS -> Records**

1. In the Pull Zone's **General** section, find the **Hostnames** panel. Enter `player01.cockxing.online` and click **Add hostname**.
2. Copy the exact CNAME record value shown below the hostname field.
3. In Cloudflare, add a `CNAME` record with **Name** `player01` and **Target** equal to Bunny's displayed target. Do not guess the target and do not point this record at the origin VM.
4. Set Cloudflare **Proxy status** to **DNS only** (grey cloud) and save. Orange-clouding another CDN can introduce certificate or connectivity failures and inserts Cloudflare in front of Bunny.
5. Return to Bunny and wait for the custom-hostname certificate to become active.

**Location: External test device**

Confirm DNS and TLS:

```bash
curl -I https://player01.cockxing.online/
```

A `403` or `404` at `/` is expected because root access is blocked. A valid HTTPS connection is the success condition.

### 10.4 Configure caching and viewer access

**Location: Bunny dashboard**

1. In **Edge Rules**, click **Add Edge Rule** and create the playlist rule with these exact form values. If the dashboard displays a unit selector, choose **Seconds**.

| Field | Value |
|---|---|
| Description | `playlistcache` |
| Action 1 | **Override Cache Time**: `0` seconds |
| Action 2 | **Override Browser Cache Time**: `0` seconds |
| Condition group | `IF` -> **Match any** |
| Condition | **Request URL** |
| Request URL value/pattern | `*.m3u8` |

   This bypasses edge and browser caching for live playlists.
2. Click **Add Edge Rule** again and create the media-segment rule with these exact form values.

| Field | Value |
|---|---|
| Description | `segmentcache` |
| Action 1 | **Override Cache Time**: `600` seconds |
| Action 2 | **Override Browser Cache Time**: `600` seconds |
| Condition group | `IF` -> **Match any** |
| Condition | **Request URL** |
| Request URL value/pattern | `*.m4s` |

3. Do **not** enable any global **Ignore Query Strings** setting when token authentication is used.
4. For a non-public stream, enable **Advanced Token Authentication** in the Pull Zone's **Security** settings. Use short-lived path-based directory tokens for `/app/linear/`, created only by a server-side application. Directory tokens let the playlist's relative media requests retain authorization.
5. Treat allowed-referrer rules as a deterrent/cost control, not authentication.

**Phase 6 gate:** From an external device, open the viewer URL in an LL-HLS-capable player or test page:

```text
https://player01.cockxing.online/app/linear/llhls.m3u8
```

Video and audio should begin. Browser/player requests for both the playlist and `.m4s` files must go to `player01.cockxing.online`, never `origin01.cockxing.online`.

## 11. Phase 7 - Secure a stream embedded on another website

Signed URLs are the access control. CORS only permits browser JavaScript to read cross-origin media responses; it does not stop someone from copying an otherwise public URL.

### 11.1 Enable viewer authentication

**Location: Bunny dashboard**

1. Open **CDN -> Pull Zones -> `player01` -> Security**.
2. Enable **Advanced Token Authentication** and copy the URL Token Authentication Key.
3. Put that key only in the authorizing backend's secret manager or server-side environment variables.
4. Never put the key in HTML, browser JavaScript, an editor-visible CMS setting, or a partner site.
5. Keep the existing query-string policy; the signer controls the playback URL parameters.

### 11.2 Issue a short-lived directory token

**Location: Backend/service that authenticates the viewer, never the visitor's browser**

For each viewing session:

1. Authenticate the viewer and confirm their entitlement before issuing a URL.
2. Use Bunny's Advanced Token Authentication signer with these inputs:

| Signer input | Value |
|---|---|
| Base URL | `https://player01.cockxing.online/app/linear/llhls.m3u8` |
| Token type | Path-based directory token |
| Token path | `/app/linear/` |
| Expiry | 5-15 minutes, matching the viewing-session design |
| Signing key | Server-side Bunny URL Token Authentication Key |

3. Return the signed playback URL only to the authorized viewer. Its path begins with `bcdn_token=...`; this allows relative `.m4s` requests to inherit the token.
4. Log the entitlement decision and expiry, but never log the complete signed URL or the signing key.
5. Do not enable IP locking by default. It can interrupt viewers roaming between mobile/VPN networks. If it is needed, sign with the viewer's IPv4 and test thoroughly.

### 11.3 Allow the exact website origins with CORS

**Location: Origin SSH terminal. Working directory: `~/ome`**

1. Identify each exact embedding page origin, including scheme and hostname: for example `https://www.partner.example`.
2. Edit OME's configuration:

```bash
nano ~/ome/config/Server.xml
```

3. Under `<CrossDomains>`, retain `player01.cockxing.online` if it hosts a player page and add one `<Url>` per permitted embed origin:

```xml
<CrossDomains>
  <Url>https://player01.cockxing.online</Url>
  <Url>https://www.partner.example</Url>
</CrossDomains>
```

4. Restart only OME and inspect the log:

```bash
cd ~/ome
sudo docker compose restart ome
sudo docker compose logs --tail=100 ome
```

5. Test from the actual partner page. A browser CORS error means the page's actual origin is not listed. A `403` points to the signed token, its expiry, or an optional hotlink policy.

### 11.4 Optional hotlink protection

**Location: Bunny dashboard**

1. Under Pull Zone **Security**, add allowed referrers without a scheme, for example `partner.example` and `www.partner.example`.
2. Include every legitimate embed domain. `*.partner.example` does not include `partner.example`, so add both if required.
3. Enable **Block Direct URL File Access** only after testing mobile apps, privacy browsers, email links, and casting. These can legitimately omit `Referer`.
4. Keep signed tokens enabled: referrers may be missing or spoofed.

## 12. Optional policy - Block a country

Country blocking is enforced by Bunny using the viewer's IP geolocation. It applies to the whole Pull Zone, which blocks both playlists and media fragments.

**Location: Bunny dashboard**

1. Open **CDN -> Pull Zones -> `player01`** and find **Blocked Countries** under Security or geographic restrictions.
2. Add the target ISO 3166-1 alpha-2 country code. `XX` is only a placeholder; replace it with a real code.
3. Save and enable it. Do not disable the matching global delivery region: disabling a region may reroute rather than block the viewer.
4. Test from a reputable exit node in the blocked country and an allowed country. The blocked case should receive `403` for both `llhls.m3u8` and `.m4s` requests.
5. Record the business/legal reason, approver, date, and review date. IP geolocation can be bypassed by VPNs and is not legal advice.

If country access must vary per viewer, do not use a Pull-Zone-wide block. Instead, have the server-side directory-token signer use `token_countries` or `token_countries_blocked` for that viewer.

## 13. Final acceptance checklist

Complete these checks before calling the stream ready:

- [ ] Cloud firewall allows only administrator SSH, public TCP 80/443, and vMix-only UDP 9999.
- [ ] `origin01` is DNS-only, points at the reserved IP, and returns `404` for an unauthenticated playlist request.
- [ ] OME's TCP 3333 is not published by Docker or permitted by the firewall.
- [ ] vMix sends encrypted SRT using `default/app/linear`; OME logs confirm the live input.
- [ ] Caddy and OME containers remain running after a restart.
- [ ] Bunny sends `X-Origin-Verify`; the viewer hostname has a valid certificate and plays the stream.
- [ ] Playlist and media requests use `player01`, not the origin hostname.
- [ ] `.m3u8` is not cached; `.m4s` has the intended TTL.
- [ ] If protected, a valid short-lived token plays and an expired/missing token fails.
- [ ] If embedded, every actual website origin is in `<CrossDomains>` and partner playback has no CORS error.
- [ ] Tests from meaningful audience regions record startup time, rebuffer count, and live latency.

## 14. Operations, incidents, and routine checks

**Location: Origin SSH terminal for commands; Bunny dashboard for CDN metrics**

Run these commands after deployment and when investigating an incident:

```bash
cd ~/ome
sudo docker compose ps
sudo docker compose logs --tail=200 ome
sudo docker compose logs --tail=200 caddy
sudo docker stats --no-stream
```

Monitor:

- vMix output status, SRT loss, and retransmissions.
- OME logs, CPU, memory, and network use.
- Caddy certificate/HTTP errors.
- Bunny cache-hit ratio, 4xx/5xx rates, regional delivery, bandwidth alerts, and playback probes from target continents.

| Symptom | First checks |
|---|---|
| vMix cannot connect | Confirm the vMix public IP still matches the UDP 9999 cloud-firewall rule, then verify host, port, stream ID, passphrase, and latency. |
| OME receives no stream | Inspect `docker compose logs ome`; confirm vMix uses `default/app/linear`, H.264, and AAC. |
| Caddy certificate fails | Confirm `origin01` DNS resolves correctly and inbound TCP 80/443 reaches the VM. |
| Origin returns a playlist publicly | Stop and correct Caddy's `X-Origin-Verify` matcher before proceeding. |
| Bunny returns 404 | Confirm Origin URL, Origin Host Header, Edge Rule header, Caddy secret, and live stream path. |
| Viewer gets 403 | Check Bunny token validity/expiry and any geographic or referrer rule. |
| Browser CORS error | Add the exact scheme + hostname of the embedding page to OME `<CrossDomains>`, then restart OME. |
| High live latency | Measure the vMix-to-origin SRT path, player live buffer, distance to origin, and Origin Shield effect before changing LL-HLS settings. |

Before a live event, rehearse encoder restart, VM restart, certificate renewal, token expiry, and primary-to-standby failover. Verify recovery on real devices and networks, not just from the origin region.

## References

- [Akamai Cloud create a compute instance](https://techdocs.akamai.com/cloud-computing/docs/create-a-compute-instance)
- [Akamai Cloud Firewall rules](https://techdocs.akamai.com/cloud-computing/docs/manage-firewall-rules)
- [Akamai reserved IPs](https://techdocs.akamai.com/cloud-computing/docs/reserved-ips)
- [Cloudflare proxy status](https://developers.cloudflare.com/dns/proxy-status/)
- [Cloudflare proxying limitations](https://developers.cloudflare.com/dns/proxy-status/limitations/)
- [OME configuration](https://ovenmedia.com/docs/ome/configuration)
- [OME SRT ingest](https://ovenmedia.com/docs/ome/live-source/srt)
- [OME LL-HLS](https://docs.ovenmediaengine.com/0.17.2/streaming/low-latency-hls)
- [Bunny CDN pricing](https://docs.bunny.net/cdn/pricing)
- [Bunny advanced token authentication](https://docs.bunny.net/cdn/security/token-authentication/advanced)
- [Bunny hotlink protection](https://docs.bunny.net/cdn/security/hotlink-protection)
- [Bunny Origin Shield](https://docs.bunny.net/cdn/performance/origin-shield)
