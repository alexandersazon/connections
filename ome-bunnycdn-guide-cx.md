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

At 3,000 Kbps video plus 128 Kbps audio, one continuous viewer transfers about **1.41 GB/hour**. At 1,000 concurrent viewers for 30 days, delivery is approximately **1.013 PB/month** before overhead.

| Viewer region | Bunny Standard price | Approximate cost for 1.013 PB entirely in that region |
|---|---:|---:|
| Europe & North America | $0.01/GB | ~$10,135/month |
| Asia & Oceania | $0.03/GB | ~$30,405/month |
| South America | $0.045/GB | ~$45,606/month |
| Middle East & Africa | $0.06/GB | ~$60,808/month |

Calculate the bill from the expected regional traffic mix. An even four-region split has a baseline near **$36,738/month**. Verify current Bunny pricing before committing; Bunny Volume may cost less but has fewer PoPs and requires audience testing.

## 3. Where each action happens

| Location label | Use it for |
|---|---|
| **Cloudflare dashboard** | DNS records in the `cockxing.online` zone only. |
| **Akamai Cloud Manager** | Account, region, Cloud Firewall, reserved IP, SSH-key, and VM configuration. |
| **Administrator workstation** | A local terminal used to create/access the SSH key and connect to the origin. No Linode CLI is required. |
| **Origin SSH terminal** | An SSH session to the Ubuntu VM as `root`. |
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

> **SSH key troubleshooting:** The account is `root`. SSH uses the private key matching the administrator public key selected when creating the VM, so no root password is required. If Windows returns `Permission denied (publickey)`, confirm the matching key is available with `Get-ChildItem $HOME\.ssh`, then specify it explicitly:
>
> ```powershell
> ssh -i $HOME\.ssh\YOUR_PRIVATE_KEY root@origin01.cockxing.online
> ```
>
> Keep this root session open for the remaining Ubuntu setup steps.

## 6. Phase 2 - Prepare and secure Ubuntu

### 6.1 Install the runtime and create directories

**Location: Origin SSH terminal as `root`. Working directory: root's home directory (`~`)**

Do not run a blanket OS upgrade during a production build. Install only the required packages and start Docker:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 curl openssl ufw
sudo systemctl enable --now docker
install -d -m 750 -o root -g root "$HOME/ome/config" "$HOME/ome/caddy"
install -d -m 755 -o root -g root "$HOME/ome/player"
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

### 6.2 Generate the origin request secret

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

### 6.3 Generate the SRT passphrase

**Location: Origin SSH terminal**

Generate a separate passphrase for the encrypted vMix-to-OME SRT connection. You will paste this exact value into OME in Phase 3 and vMix in Phase 5.

```bash
openssl rand -hex 32 | tee "$HOME/ome/config/srt-passphrase.txt"
chmod 600 "$HOME/ome/config/srt-passphrase.txt"
```

Never put this value in source control, chat, screenshots, browser code, or a public ticket.

**Phase 2 gate:** `~/ome/config`, `~/ome/caddy`, `~/ome/player`, and the two mode-`600` secret files exist under the `root` account. The `player` directory is used only by the optional player UI in section 10.6.

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
              <PartHoldBack>3</PartHoldBack>
              <SegmentDuration>6</SegmentDuration>
              <SegmentCount>12</SegmentCount>
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
        handle_path /player/* {
            root * /srv/player
            file_server
        }
        handle {
            reverse_proxy ome:3333 {
                header_up -X-Origin-Verify
            }
        }
    }
    respond "Not found" 404
}
```

Any request that omits or mismatches the header gets `404`; OME and the optional `/player/` static page are not directly exposed. Caddy removes the verification header before proxying so the secret is not written into OME request logs. The `/player/` route strips that prefix and serves files mounted at `/srv/player`; every other authorized request continues to OME.

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
      - ./player:/srv/player:ro
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
| Latency | Start at `1,000 ms`; raise it if WAN loss/jitter appears |
| Encryption | Strong passphrase; use a 32-byte key length where offered |
| Video | H.264, 1280x720, 30 fps, 3,000 Kbps CBR |
| Audio | AAC-LC, 128 Kbps |
| Keyframe interval | 2 seconds |

If your vMix release does not show **Enable SRT**, update to vMix 23 or newer. Configure the individual fields above; vMix's SRT output dialog does not require a single URL.

For reference, the equivalent SRT URL is below. Substitute the Akamai origin IP and SRT secret; keep the `streamid` query parameter.

```text
srt://ORIGIN_IPV4:9999?streamid=default/app/linear&passphrase=YOUR_SRT_SECRET&pbkeylen=32
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

### 10.5 Startup stutter tuning

If playback stutters for the first 10-15 seconds and then becomes smooth, the player is usually starting too close to the live edge for the available SRT/WAN jitter. The configuration above deliberately starts about 3 seconds behind the edge, keeps up to 12 seconds of startup buffer, and uses a 1,000 ms SRT latency. This adds a small amount of live delay in exchange for a stable start.

Apply the changed OME and player configuration, then recreate the containers:

```bash
cd ~/ome
sudo docker compose config --quiet
sudo docker compose up -d
sudo docker compose logs --tail=100 ome caddy
```

Validate the cause before increasing the buffer further:

1. From an external device, confirm that the playlist's media sequence advances between two requests several seconds apart. A frozen sequence indicates an origin or CDN playlist problem, not a player buffer problem.
2. Check the browser network panel for repeated playlist requests, failed `.m4s` requests, or a non-zero `Age` header on `llhls.m3u8`. Playlist responses must not be cached; recheck the Bunny `*.m3u8` rule if they are stale.
3. Check OME and vMix for SRT loss or retransmissions. If they continue, raise vMix SRT latency to `1,500-2,000 ms` before increasing LL-HLS holdback.
4. If the stream is still unstable after SRT loss is controlled, raise `PartHoldBack` to `4` and `liveSyncDuration` to `4`, then recreate OME and retest. Do not change several latency values at once during an incident.

Safari and other native-HLS clients do not use the hls.js settings. For those clients, tune `PartHoldBack` first and accept the corresponding increase in live latency.

### 10.6 Optional - Create an editable player UI

**Location: Origin SSH terminal. Working directory: `~/ome`**

`player01.cockxing.online` is Bunny's custom hostname, not a separate web host. The Caddy and Compose configuration in Phase 4 therefore gives Bunny an authorized `/player/` origin path for this static page. The Compose bind mount maps the origin VM folder `~/ome/player` to `/srv/player` **inside the Caddy container**, read-only. Viewers load it through Bunny at `https://player01.cockxing.online/player/`; they must never use `https://origin01.cockxing.online/player/`.

The example uses a small custom control bar, so each UI element can be enabled or removed without relying on browser-specific native controls. Create the directory and file:

```bash
install -d -m 755 -o root -g root "$HOME/ome/player"
nano ~/ome/player/index.html
```

Paste the following. It uses the same-host HLS path by default for a public stream. This static example is not a complete private-playback integration: if Bunny token authentication is enabled, the page must receive a newly issued signed playback URL from the authorizing backend before `startStream()` runs. Do not place a permanent signed URL or token key in this file.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Live Player</title>
  <style>
    html, body { background: #000; margin: 0; min-height: 100%; padding: 0; }
    body { min-height: 100vh; }
    .player {
      background: #000; color: #fff; font-family: Arial, sans-serif;
      margin: 0; max-width: 960px; padding: 0; position: relative;
    }
    .video-frame { aspect-ratio: 16 / 9; position: relative; }
    video { aspect-ratio: 16 / 9; display: block; object-fit: contain; width: 100%; }
    .controls {
      align-items: center; background: rgb(0 0 0 / 50%); bottom: 0; display: flex;
      gap: .75rem; left: 0; padding: .75rem; position: absolute; right: 0;
      transition: opacity .3s ease;
    }
    .player.controls-hidden .controls { opacity: 0; pointer-events: none; }
    .player:fullscreen { background: #000; max-width: none; width: 100%; }
    .controls button, .controls select { font: inherit; }
    .controls .icon-button {
      align-items: center; background: transparent; border: 0; color: #fff; cursor: pointer;
      display: inline-flex; height: 2rem; justify-content: center; padding: .25rem; width: 2rem;
    }
    .controls .icon-button:hover, .controls .icon-button:focus-visible { background: rgb(255 255 255 / 20%); }
    .controls .icon-button svg { fill: none; height: 1.35rem; stroke: currentColor; stroke-linecap: round;
      stroke-linejoin: round; stroke-width: 2; width: 1.35rem; }
    #brandingText { margin-left: auto; opacity: .8; white-space: nowrap; }
    [hidden] { display: none !important; }
    #liveStatus {
      background: #22c55e; border-radius: 50%; box-shadow: 0 0 6px #22c55e;
      display: inline-block; height: .7rem; width: .7rem;
    }
    #streamStatus {
      align-items: center; background: #000; display: flex; font-weight: 700;
      inset: 0; justify-content: center; letter-spacing: .04em; position: absolute;
      text-align: center;
    }
  </style>
</head>
<body>
  <div class="player" id="player">
    <div class="video-frame">
      <video id="video" muted autoplay playsinline aria-label="Live stream"></video>
      <div id="streamStatus" role="status" aria-live="polite">WAIT FOR THE LIVE STREAM...</div>
    </div>
    <div class="controls" aria-label="Player controls">
      <button id="playButton" type="button" aria-label="Play live stream">Play</button>
      <button id="muteButton" class="icon-button" type="button" aria-label="Unmute live stream"></button>
      <span id="liveStatus" aria-label="Live" aria-live="polite"></span>
      <span id="time" aria-live="off">--:--</span>
      <label id="resolutionControl">
        Resolution
        <select id="resolution" aria-label="Resolution"><option value="-1">Auto</option></select>
      </label>
      <button id="fullscreenButton" class="icon-button" type="button" aria-label="Enter full screen"></button>
      <span id="brandingText" aria-label="Player branding"></span>
    </div>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/hls.js@1.7.1"></script>
  <script>
    const settings = {
      showPlayButton: false,       // Set true to show the Play/Pause button.
      showMuteButton: true,        // Set false to hide the Mute/Unmute button.
      showLiveStatus: true,        // Set false to hide the live-status circle.
      showTime: false,             // Set true to show the live-edge time display.
      showResolutionMenu: true,    // It is hidden automatically for a single rendition.
      showFullscreenButton: true,  // Set false to hide the Full screen button.
      brandingText: 'YOUR BRAND'   // Change the text or use '' to hide branding.
    };
    const playbackUrl = '/app/linear/llhls.m3u8';
    const player = document.querySelector('#player');
    const video = document.querySelector('#video');
    const playButton = document.querySelector('#playButton');
    const muteButton = document.querySelector('#muteButton');
    const time = document.querySelector('#time');
    const liveStatus = document.querySelector('#liveStatus');
    const streamStatus = document.querySelector('#streamStatus');
    const resolutionControl = document.querySelector('#resolutionControl');
    const resolution = document.querySelector('#resolution');
    const fullscreenButton = document.querySelector('#fullscreenButton');
    const brandingText = document.querySelector('#brandingText');
    const retryDelayMs = 10000;
    const controlsHideDelayMs = 2500;
    let hls;
    let retryTimer;
    let controlsTimer;
    let usingNativeHls = false;

    playButton.hidden = !settings.showPlayButton;
    muteButton.hidden = !settings.showMuteButton;
    liveStatus.hidden = !settings.showLiveStatus;
    time.hidden = !settings.showTime;
    resolutionControl.hidden = !settings.showResolutionMenu;
    fullscreenButton.hidden = !settings.showFullscreenButton;
    brandingText.textContent = settings.brandingText;
    brandingText.hidden = !settings.brandingText;

    function showControls() {
      player.classList.remove('controls-hidden');
      clearTimeout(controlsTimer);
      controlsTimer = setTimeout(function () {
        player.classList.add('controls-hidden');
      }, controlsHideDelayMs);
    }

    player.addEventListener('pointermove', showControls);
    player.addEventListener('pointerdown', showControls);
    player.addEventListener('focusin', showControls);
    player.addEventListener('keydown', showControls);
    document.addEventListener('contextmenu', function (event) {
      event.preventDefault();
    });
    document.addEventListener('keydown', function (event) {
      const key = event.key.toLowerCase();
      const inspectShortcut = event.ctrlKey && event.shiftKey && ['i', 'j', 'c'].includes(key);
      const sourceShortcut = event.ctrlKey && key === 'u';
      if (event.key === 'F12' || inspectShortcut || sourceShortcut) event.preventDefault();
    });

    function syncPlayButton() {
      playButton.textContent = video.paused ? 'Play' : 'Pause';
      playButton.setAttribute('aria-label', playButton.textContent + ' live stream');
    }

    function syncMuteButton() {
      const label = video.muted ? 'Unmute' : 'Mute';
      muteButton.innerHTML = video.muted
        ? '<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M11 5 6 9H3v6h3l5 4V5Z"/><path d="m16 9 5 5m0-5-5 5"/></svg>'
        : '<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M11 5 6 9H3v6h3l5 4V5Z"/><path d="M15.5 9.5a4 4 0 0 1 0 5m2.5-7.5a7 7 0 0 1 0 10"/></svg>';
      muteButton.setAttribute('aria-label', label + ' live stream');
    }

    function toggleMute() {
      video.muted = !video.muted;
      if (!video.muted) video.play().catch(function () {});
      syncMuteButton();
    }

    function syncFullscreenButton() {
      const inFullscreen = document.fullscreenElement === player;
      const label = inFullscreen ? 'Exit full screen' : 'Enter full screen';
      fullscreenButton.innerHTML = inFullscreen
        ? '<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M9 4v5H4m11-5v5h5M9 20v-5H4m16 0h-5v5"/></svg>'
        : '<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M4 9V4h5m6 0h5v5M4 15v5h5m11-5v5h-5"/></svg>';
      fullscreenButton.setAttribute('aria-label', label);
    }

    function toggleFullscreen() {
      if (document.fullscreenElement) {
        document.exitFullscreen();
      } else {
        player.requestFullscreen().catch(function () {});
      }
    }

    function updateTime() {
      // A live HLS stream has no fixed duration; show distance from its live edge.
      const range = video.seekable;
      if (!range.length) return;
      const behindLive = Math.max(0, range.end(range.length - 1) - video.currentTime);
      time.textContent = behindLive < 1 ? 'LIVE' : '-' + Math.round(behindLive) + 's';
    }

    function togglePlayback() {
      if (video.paused) video.play().catch(function () {}); else video.pause();
    }

    function showWaiting() {
      streamStatus.textContent = 'WAIT FOR THE LIVE STREAM';
      streamStatus.hidden = false;
      liveStatus.hidden = true;
      resolutionControl.hidden = true;
    }

    function showLive() {
      streamStatus.hidden = true;
      liveStatus.hidden = !settings.showLiveStatus;
    }

    function startMutedPlayback() {
      // Browsers allow automatic live playback only when it starts muted.
      video.play().catch(function () {});
    }

    function resetResolutionMenu() {
      resolution.innerHTML = '<option value="-1">Auto</option>';
    }

    function stopHls() {
      if (hls) {
        hls.destroy();
        hls = undefined;
      }
    }

    function scheduleRetry() {
      clearTimeout(retryTimer);
      retryTimer = setTimeout(startStream, retryDelayMs);
    }

    function handleStreamOffline() {
      stopHls();
      usingNativeHls = false;
      video.pause();
      video.removeAttribute('src');
      video.load();
      showWaiting();
      scheduleRetry();
    }

    function addResolutionLevels(levels) {
      resetResolutionMenu();
      resolutionControl.hidden = !settings.showResolutionMenu || levels.length < 2;
      levels.forEach(function (level, index) {
        const option = document.createElement('option');
        option.value = index;
        option.textContent = level.height ? level.height + 'p' : Math.round(level.bitrate / 1000) + ' kbps';
        resolution.appendChild(option);
      });
    }

    function startHlsJsPlayback() {
      const instance = new Hls();
      hls = instance;
      instance.on(Hls.Events.MANIFEST_PARSED, function () {
        if (hls !== instance) return;
        addResolutionLevels(instance.levels);
        showLive();
        startMutedPlayback();
      });
      instance.on(Hls.Events.ERROR, function (_event, data) {
        if (hls === instance && data.fatal) handleStreamOffline();
      });
      instance.loadSource(playbackUrl);
      instance.attachMedia(video);
    }

    function startStream() {
      clearTimeout(retryTimer);
      stopHls();
      resetResolutionMenu();
      showWaiting();
      if (window.Hls && Hls.isSupported()) {
        usingNativeHls = false;
        startHlsJsPlayback();
      } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
        usingNativeHls = true;
        video.src = playbackUrl;
        video.load();
      } else {
        streamStatus.textContent = 'HLS playback is not supported by this browser.';
      }
    }

    playButton.addEventListener('click', togglePlayback);
    muteButton.addEventListener('click', toggleMute);
    fullscreenButton.addEventListener('click', toggleFullscreen);
    video.addEventListener('click', function () {
      // Clicking the video only enables sound; it never pauses the live stream.
      if (video.muted) {
        video.muted = false;
        video.play().catch(function () {});
        syncMuteButton();
      }
    });
    video.addEventListener('play', function () {
      syncPlayButton();
      showControls();
    });
    video.addEventListener('pause', function () {
      syncPlayButton();
      showControls();
    });
    video.addEventListener('timeupdate', updateTime);
    video.addEventListener('progress', updateTime);
    video.addEventListener('loadedmetadata', function () {
      if (!usingNativeHls) return;
      showLive();
      startMutedPlayback();
    });
    video.addEventListener('error', function () {
      if (usingNativeHls) handleStreamOffline();
    });
    resolution.addEventListener('change', function () {
      if (hls) hls.currentLevel = Number(resolution.value); // -1 restores adaptive Auto quality.
    });
    document.addEventListener('fullscreenchange', syncFullscreenButton);

    syncMuteButton();
    syncFullscreenButton();
    showControls();
    startStream();
  </script>
</body>
</html>
```

Apply the Caddy and Compose changes from Phase 4, then start or recreate Caddy so it receives the static-file mount:

```bash
cd ~/ome
sudo docker compose config --quiet
sudo docker compose up -d
sudo docker compose logs --tail=50 caddy
```

Set `showPlayButton`, `showLiveStatus`, or `showTime` to `false` to hide those controls; change `brandingText` to replace the branding text or set it to `''` to hide it. The player also blocks the context menu and common source/devtools shortcuts as a deterrent. This cannot securely hide client-delivered HTML from a determined user; keep secrets and access control server-side. The video itself remains clickable for play/pause when the button is hidden. The time display is deliberately a live-edge offset rather than a duration because a live stream has no fixed end time.

When no live manifest is available, a full-frame `WAIT FOR THE LIVE STREAM` message is displayed and the player retries every 10 seconds. As soon as OME publishes the manifest, the overlay is removed and playback starts automatically. Browsers permit that automatic start only when muted; the first viewer click on the video enables audio, and later clicks toggle play/pause.

The current OME configuration is a single 720p bypass output, so it exposes only one HLS rendition. The resolution menu therefore remains hidden with this guide's default stream. A UI cannot create 480p/720p/1080p choices: configure a multi-variant HLS playlist from multiple encoder outputs or a transcoding workflow first. When that playlist has two or more variants, hls.js populates the menu and `Auto` retains adaptive-bitrate selection; a forced resolution can cause a short rebuffer while the player switches levels.

## 11. Run the concurrent-viewer counter on another server

Deploy the counter on a separate server at `count.cockxing.online`. Keep Cloudflare in DNS-only mode, keep port 3000 private, and do not route the player through the streaming origin.

### 11.1 Configure the counter hostname and server

**Location: Cloudflare dashboard**

1. Open the `cockxing.online` zone and go to **DNS -> Records**.
2. Add an `A` record with **Name** `count` and the public IPv4 address of the counter server.
3. Set **Proxy status** to **DNS only** (grey cloud). Do not enable orange-cloud.
4. Verify:

```bash
dig +short count.cockxing.online
```

Expected result: the command prints only the counter server public IP.

**Location: Counter server SSH terminal**

1. Install the required packages:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs docker.io docker-compose-v2 caddy
sudo systemctl enable --now docker
```

2. Create the app directory and service file:

```bash
sudo install -d -m 755 /opt/viewer-count
sudo nano /opt/viewer-count/server.js
```

3. Paste the following file exactly:

```js
const http = require('node:http');

const port = 3000;
const heartbeatIntervalMs = 30_000;
const viewerTtlMs = 90_000;
const maxBodyBytes = 2_048;
const viewers = new Map();
const allowedOrigins = new Set(['https://player01.cockxing.online']);
const uuid = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-8][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

function pruneViewers() {
  const cutoff = Date.now() - viewerTtlMs;
  for (const [viewerId, viewer] of viewers) {
    if (viewer.lastSeen < cutoff) viewers.delete(viewerId);
  }
}

function normaliseEmbedUrl(value) {
  if (typeof value !== 'string' || !value) return 'Direct player or referrer unavailable';
  try {
    const url = new URL(value);
    if (url.protocol !== 'https:' && url.protocol !== 'http:') return 'Direct player or referrer unavailable';
    url.search = '';
    url.hash = '';
    return url.href;
  } catch {
    return 'Direct player or referrer unavailable';
  }
}

function setCors(response, origin) {
  if (allowedOrigins.has(origin)) {
    response.setHeader('Access-Control-Allow-Origin', origin);
    response.setHeader('Vary', 'Origin');
  }
  response.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
  response.setHeader('Access-Control-Allow-Headers', 'Content-Type');
}

function sendJson(response, status, body, origin) {
  setCors(response, origin);
  response.writeHead(status, {
    'Cache-Control': 'no-store',
    'Content-Type': 'application/json; charset=utf-8'
  });
  response.end(status === 204 ? undefined : JSON.stringify(body));
}

const server = http.createServer((request, response) => {
  const url = new URL(request.url, 'http://counter');
  const origin = request.headers.origin || '';

  if (request.method === 'OPTIONS' && url.pathname === '/heartbeat') {
    return sendJson(response, 204, {}, origin);
  }

  if (request.method === 'GET' && url.pathname === '/healthz') {
    return sendJson(response, 200, { ok: true }, origin);
  }

  if (request.method === 'GET' && url.pathname === '/count') {
    pruneViewers();
    const embedUrls = new Map();
    for (const viewer of viewers.values()) {
      embedUrls.set(viewer.embedUrl, (embedUrls.get(viewer.embedUrl) || 0) + 1);
    }
    return sendJson(response, 200, {
      concurrentViewers: viewers.size,
      embedUrls: [...embedUrls]
        .map(([url, views]) => ({ url, views }))
        .sort((a, b) => b.views - a.views || a.url.localeCompare(b.url)),
      heartbeatIntervalSeconds: heartbeatIntervalMs / 1000,
      activeWindowSeconds: viewerTtlMs / 1000
    }, origin);
  }

  if (request.method !== 'POST' || url.pathname !== '/heartbeat') {
    return sendJson(response, 404, { error: 'Not found' }, origin);
  }

  let body = '';
  request.setEncoding('utf8');
  request.on('data', (chunk) => {
    body += chunk;
    if (Buffer.byteLength(body) > maxBodyBytes) request.destroy();
  });
  request.on('end', () => {
    try {
      const { viewerId, embedUrl } = JSON.parse(body);
      if (typeof viewerId !== 'string' || !uuid.test(viewerId)) {
        return sendJson(response, 400, { error: 'A valid viewerId is required' }, origin);
      }
      pruneViewers();
      viewers.set(viewerId, { lastSeen: Date.now(), embedUrl: normaliseEmbedUrl(embedUrl) });
      return sendJson(response, 204, {}, origin);
    } catch {
      return sendJson(response, 400, { error: 'Invalid JSON' }, origin);
    }
  });
});

setInterval(pruneViewers, heartbeatIntervalMs).unref();
server.listen(port, '127.0.0.1', () => console.log(`viewer-count listening on ${port}`));
```

4. Create the Docker build file:

```bash
sudo nano /opt/viewer-count/Dockerfile
```

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY server.js ./
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

5. Create the Compose file:

```bash
sudo nano /opt/viewer-count/docker-compose.yml
```

```yaml
services:
  viewer-count:
    build: .
    restart: unless-stopped
    network_mode: host
```

6. Start the service and verify it is bound only to localhost:

```bash
cd /opt/viewer-count
sudo docker compose up -d --build
for i in $(seq 1 20); do
  curl -fsS http://127.0.0.1:3000/healthz >/dev/null && break
  sleep 1
done
curl -i http://127.0.0.1:3000/healthz
sudo ss -ltnp | grep ':3000'
```

Verify: the health request returns `200` and the socket is bound to `127.0.0.1:3000` only.

### 11.2 Configure Caddy and the dashboard

**Location: Counter server SSH terminal**

1. Create the Caddy config file:

```bash
sudo nano /etc/caddy/Caddyfile
```

2. Generate the bcrypt hash and paste it into the file below:

```bash
sudo caddy hash-password --algorithm bcrypt
```

```caddyfile
count.cockxing.online {
    handle_path /dashboard/* {
        basic_auth {
            operator PASTE_BCRYPT_HASH
        }
        root * /srv/viewer-dashboard
        file_server
    }

    handle /count {
        basic_auth {
            operator PASTE_BCRYPT_HASH
        }
        reverse_proxy 127.0.0.1:3000
    }

    handle /heartbeat {
        reverse_proxy 127.0.0.1:3000
    }

    handle /healthz {
        reverse_proxy 127.0.0.1:3000
    }

    respond 404
}
```

3. Create the dashboard file and reload Caddy:

```bash
sudo install -d -m 755 /srv/viewer-dashboard
sudo nano /srv/viewer-dashboard/index.html
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

```html
<!doctype html>
<html lang="en">
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Live stream operations</title>
<body>
  <h1>Live stream operations</h1>
  <p id="count">Loading...</p>
  <pre id="details"></pre>
  <script>
    async function refresh() {
      const count = document.querySelector('#count');
      const details = document.querySelector('#details');
      try {
        const response = await fetch('/count', { cache: 'no-store' });
        if (!response.ok) throw new Error('Request failed: ' + response.status);
        const data = await response.json();
        count.textContent = data.concurrentViewers + ' active player sessions';
        details.textContent = JSON.stringify(data.embedUrls, null, 2);
      } catch (error) {
        count.textContent = 'Viewer count unavailable';
        details.textContent = error.message;
      }
    }
    refresh();
    setInterval(refresh, 30_000);
  </script>
</body>
</html>
```

4. Validate the public endpoints:

```bash
curl -sS -D - -o /dev/null https://count.cockxing.online/healthz
curl -I -X OPTIONS https://count.cockxing.online/heartbeat \
  -H 'Origin: https://player01.cockxing.online' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type'
```

Expected results: the health `GET` returns `200`; the preflight returns `204` and the `Access-Control-Allow-Origin` header matches `https://player01.cockxing.online`. Open `https://count.cockxing.online/dashboard/` and confirm the browser prompts for the `operator` username and password.

### 11.3 Point the player at the separate server

**Location: Origin SSH terminal. File: `~/ome/player/index.html`**

1. Immediately after the existing `const retryDelayMs = 10000;` line, add:

```js
    const viewerIdKey = 'omeViewerId';
    const viewerId = sessionStorage.getItem(viewerIdKey) || crypto.randomUUID();
    const viewerHeartbeatUrl = 'https://count.cockxing.online/heartbeat';
    const viewerHeartbeatMs = 30_000;
    let viewerTimer;
```

2. Before `function startStream()`, add:

```js
    function sendViewerHeartbeat() {
      fetch(viewerHeartbeatUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ viewerId, embedUrl: document.referrer }),
        keepalive: true
      }).catch(function () {});
    }

    function startViewerTracking() {
      if (viewerTimer) return;
      sessionStorage.setItem(viewerIdKey, viewerId);
      sendViewerHeartbeat();
      viewerTimer = setInterval(sendViewerHeartbeat, viewerHeartbeatMs);
    }

    function stopViewerTracking() {
      clearInterval(viewerTimer);
      viewerTimer = undefined;
    }
```

3. Add `startViewerTracking();` as the final statement in `showLive()`. Add `stopViewerTracking();` as the first statement in `handleStreamOffline()`. Do not change the HLS URL. Only the heartbeat goes to `count.cockxing.online`.

4. Validate from an external machine:

```bash
curl -sS -D - -o /dev/null https://count.cockxing.online/healthz
curl -I -X OPTIONS https://count.cockxing.online/heartbeat \
  -H 'Origin: https://player01.cockxing.online' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type'
```

Verify: the health `GET` returns `200` and the preflight returns `204`. If the player does not send the heartbeat, confirm `startViewerTracking()` was added and that the player origin is allowlisted in the counter service.

## 12. Phase 7 - Secure a stream embedded on another website

Signed URLs are the access control. CORS only permits browser JavaScript to read cross-origin media responses; it does not stop someone from copying an otherwise public URL.

### 12.1 Enable viewer authentication

**Location: Bunny dashboard**

1. Open **CDN -> Pull Zones -> `player01` -> Security**.
2. Enable **Advanced Token Authentication** and copy the URL Token Authentication Key.
3. Put that key only in the authorizing backend's secret manager or server-side environment variables.
4. Never put the key in HTML, browser JavaScript, an editor-visible CMS setting, or a partner site.
5. Keep the existing query-string policy; the signer controls the playback URL parameters.

### 12.2 Issue a short-lived directory token

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

3. Return the signed playback URL only to the authorized viewer. Its token and expiry parameters must remain on the playlist URL so the CDN can authorize the playlist and its relative `.m4s` requests.
4. Log the entitlement decision and expiry, but never log the complete signed URL or the signing key.
5. Do not enable IP locking by default. It can interrupt viewers roaming between mobile/VPN networks. If it is needed, sign with the viewer's IPv4 and test thoroughly.

The static page in section 10.6 does not know how to obtain this URL by itself. For a private player, expose an authenticated backend endpoint or render the page through the authenticated application, then pass the returned signed URL to the player before calling `startStream()`. Keep the Bunny signing key exclusively in that backend; never proxy it or expose it to browser code.

### 12.3 Allow the exact website origins with CORS

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

### 12.4 Optional hotlink protection

**Location: Bunny dashboard**

1. Under Pull Zone **Security**, add allowed referrers without a scheme, for example `partner.example` and `www.partner.example`.
2. Include every legitimate embed domain. `*.partner.example` does not include `partner.example`, so add both if required.
3. Enable **Block Direct URL File Access** only after testing mobile apps, privacy browsers, email links, and casting. These can legitimately omit `Referer`.
4. Keep signed tokens enabled: referrers may be missing or spoofed.

## 13. Optional policy - Block a country

Country blocking is enforced by Bunny using the viewer's IP geolocation. It applies to the whole Pull Zone, which blocks both playlists and media fragments.

**Location: Bunny dashboard**

1. Open **CDN -> Pull Zones -> `player01`** and find **Blocked Countries** under Security or geographic restrictions.
2. Add the target ISO 3166-1 alpha-2 country code. `XX` is only a placeholder; replace it with a real code.
3. Save and enable it. Do not disable the matching global delivery region: disabling a region may reroute rather than block the viewer.
4. Test from a reputable exit node in the blocked country and an allowed country. The blocked case should receive `403` for both `llhls.m3u8` and `.m4s` requests.
5. Record the business/legal reason, approver, date, and review date. IP geolocation can be bypassed by VPNs and is not legal advice.

If country access must vary per viewer, do not use a Pull-Zone-wide block. Instead, have the server-side directory-token signer use `token_countries` or `token_countries_blocked` for that viewer.

## 14. Final acceptance checklist

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
- [ ] If used, the concurrent-viewer counter runs on a separate HTTPS server; the streaming origin has no counter service or counter routes.
- [ ] If embedded, every actual website origin is in `<CrossDomains>` and partner playback has no CORS error.
- [ ] Tests from meaningful audience regions record startup time, rebuffer count, and live latency.

## 15. Operations, incidents, and routine checks

**Location: Origin SSH terminal for commands; Bunny dashboard for CDN metrics**

Run these commands after deployment and when investigating an incident:

```bash
cd ~/ome
sudo docker compose ps
sudo docker compose logs --tail=200 ome
sudo docker compose logs --tail=200 caddy
sudo docker compose logs --tail=200 viewer-count
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
- [hls.js embedding and browser-support guidance](https://www.npmjs.com/package/hls.js)
- [hls.js quality-switch API](https://github.com/video-dev/hls.js/blob/master/docs/API.md)
- [Bunny CDN pricing](https://docs.bunny.net/cdn/pricing)
- [Bunny advanced token authentication](https://docs.bunny.net/cdn/security/token-authentication/advanced)
- [Bunny hotlink protection](https://docs.bunny.net/cdn/security/hotlink-protection)
- [Bunny Origin Shield](https://docs.bunny.net/cdn/performance/origin-shield)
