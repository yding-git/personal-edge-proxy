# AGENTS.md — AI maintainer / deployer guide

Read this file before editing or deploying the repository.

## 1. Core model

Keep the architecture separated into two independent layers:

```text
Inbound  = how the client reaches the VPS
Outbound = how the VPS reaches a destination
```

Do not confuse inbound redundancy with outbound identity/reputation.

Typical inbound roles:

```text
Hysteria2                    primary daily entry
VLESS + REALITY + Vision     optional TCP backup
VLESS + WS + CF Tunnel       optional emergency entry
```

Typical outbound roles:

```text
VPS direct       ordinary traffic
WARP             preferred selected-AI egress
fixed SOCKS5     optional stable egress for selected services
block            explicit deny / fail-closed
```

---

## 2. Deployment profiles — recommended interpretation

Profiles are not strictly cumulative. Use the profile that matches the user's real goal.

### Profile A — minimum viable

```text
Client -> HY2 -> VPS -> Direct
```

Use when the user only wants a simple working personal node.

Trade-off: AI services see the VPS data-center egress directly. On low-reputation or heavily reused data-center ranges, users may encounter more availability challenges, CAPTCHAs, regional mismatches, or account-security checks.

Do **not** claim that this guarantees account suspension or that data-center IPs are universally unusable.

### Profile B — inbound-resilient direct egress

```text
HY2 primary
+
REALITY backup
+
VPS Direct egress
```

This reduces **inbound protocol failure risk** when UDP is poor or unavailable.

It does **not** materially improve the final egress identity versus Profile A, because destinations still see the VPS Direct IP.

### Profile C — WARP-selected AI egress

```text
HY2 -> VPS
        |- ordinary traffic -> Direct
        `- selected AI      -> WARP Local Proxy
```

Use when the user wants to reduce dependence on the VPS's raw data-center egress for AI/SaaS traffic.

This is the preferred starting profile for an AI-heavy use case.

REALITY is optional here; do not force it if HY2 is stable.

### Profile D — recommended practical AI profile

```text
HY2 -> VPS
        |- ordinary traffic          -> Direct
        |- OpenAI / ChatGPT / Codex  -> WARP
        |- Gemini / Google AI        -> WARP
        `- Claude / Anthropic        -> trusted fixed SOCKS5 (optional by policy)
```

This is the repository maintainer's **preferred practical profile** when the user primarily uses AI services and also wants a deliberately stable Claude/Anthropic egress.

Reasons:

- HY2 keeps the client side simple and fast when UDP is healthy.
- WARP decouples selected AI traffic from the VPS's raw data-center IP.
- A fixed SOCKS5 can keep selected services on a stable final egress when the user explicitly wants that property.
- Direct remains available for normal traffic and as a clean network baseline.

Do not claim:

- that WARP is residential;
- that Claude requires residential IPs;
- that this profile guarantees avoiding bans or risk checks;
- that any service must use these exact routes.

### Profile E — extra inbound resilience / icing on the cake

Start from Profile D, then optionally add:

```text
VLESS + REALITY + Vision
Cloudflare Tunnel / VLESS WS
```

Use only when the user actually wants more inbound fallback paths.

This is an **availability enhancement**, not an egress-reputation enhancement.

### Default deployment rule

If the user says only "set this up for AI use" and provides no contrary preference:

1. prefer **Profile C** as the default balanced target;
2. upgrade to **Profile D** if the user has/provides a trusted fixed egress and wants Claude/Anthropic pinned to it;
3. add REALITY only when the user wants TCP fallback or the network has real UDP problems;
4. add Cloudflare Tunnel only as an extra emergency entry.

Do not overbuild simply because example files exist.

---

## 3. VPS procurement guidance — guide the human before deployment

If the user has **not bought a VPS yet**, do not jump directly to installation commands.

First read and follow:

- [`docs/vps-selection.md`](./docs/vps-selection.md)

The agent's job is to help the human make a procurement decision, not merely repeat VPS marketing specs.

### 3.1 Use the project's “impossible triangle” as the selection model

Treat VPS selection as a balance between:

```text
                 route / speed
        latency · loss · peak hour
          UDP · stable throughput
                       ▲
                      / \
                     /   \
                    /     \
             config ─────── price
       CPU/RAM/disk/traffic  real long-term cost
```

This is not a literal mathematical impossibility. It is a decision framework:

> **Understand the user's real need first, then set minimum acceptable thresholds for all three corners and search for the best balance inside the budget.**

Do not assume that a larger CPU/RAM package is better for a personal relay. Once the workload has crossed its practical minimum, better routing and stability may be more valuable than unused compute.

### 3.2 Ask high-leverage questions first

Do **not** start with an open-ended question such as:

```text
“你对 VPS 有什么需求？”
```

That often produces a useless answer like “要快、要稳、要便宜”.

Instead, ask about the inputs that actually change the search path. Prefer one variable per question, 2–4 understandable options, and briefly explain why the question matters.

Do not waste early questions on baseline assumptions that this project can safely infer for a personal HY2 relay. Unless the user says otherwise, the default screening baseline may assume:

```text
Linux VPS
root / sudo
public IPv4 preferred
UDP required for HY2
reinstall / console recovery preferred
personal authenticated use, not a public open proxy
```

State these assumptions before vendor research so the user can correct them, but do not make them answer technical questions they do not need to understand.

Use information already present in the conversation. Never ask the user to repeat a known ISP, location, budget or use case.

### 3.3 Four-question fast path

If almost nothing is known, start with these four questions. Keep the wording human-facing.

#### 1. Local carrier / ISP — highest leverage for mainland-China routing

Ask:

> **你平时主要用哪家网络连这台 VPS？** 这个会决定我优先看哪一类线路。
>
> - 中国电信
> - 中国联通
> - 中国移动
> - 其他 / 境外网络

For a mainland-China user, use the answer as a **shortlisting direction**, not as proof of route quality:

```text
China Telecom
  focus first on: CN2 / AS4809
  if a vendor claims CN2 GIA, verify the actual route and peak-hour behavior

China Unicom
  focus first on: AS9929 + AS10099 / CUP / China Unicom Premium

China Mobile
  focus first on: CMIN2 / AS58807

Other / overseas
  do not force mainland-China route labels; choose based on that network's actual peers and tests
```

Do not say “China Telecom = must buy CN2 GIA”. `AS4809` or a CN2 label does not by itself prove full-path GIA quality.

If province/city is not already known and route-sensitive comparison requires it, ask it as a follow-up. Do not make province/city a mandatory first question for every user.

#### 2. Main use case — determines the triangle weights

Ask:

> **这台 VPS 主要拿来干什么？** 我会按用途决定“线路 / 价格 / 配置”哪个更重要。
>
> - AI / ChatGPT / Claude / Gemini 为主
> - 日常网页 + 视频 / 流媒体
> - 游戏 / 对延迟很敏感
> - 综合都要
> - 还要跑网站、Docker、数据库或其他服务

Translate the answer internally:

```text
AI / chat
  -> route stability, packet loss, connection success and UDP matter more than oversized CPU/RAM

web + streaming
  -> sustained throughput + peak-hour stability become more important

gaming / latency-sensitive
  -> RTT + jitter + packet loss + UDP receive very high weight

mixed use
  -> balanced route / price / config

hosting additional services
  -> raise CPU/RAM/disk/config weight before recommending 1C1G
```

If the user's AI architecture will use WARP or a fixed egress, still prioritize the user's local -> VPS ingress quality; separately verify VPS -> WARP / upstream quality later.

#### 3. Budget — use long-term effective price

Ask in the user's own currency when possible:

> **你每个月大概愿意花多少钱？** 我会按长期续费价算，不会拿首月促销价冒充长期价格。
>
> - 低预算
> - 中低预算
> - 中等预算
> - 预算比较宽松
> - 或者直接告诉我一个上限

For mainland-China consumer guidance, a simple example range is acceptable if useful:

```text
<= ¥30
¥30–60
¥60–100
> ¥100
```

Do not treat these ranges as universal. Convert them to the user's currency/context.

Always distinguish:

```text
first-purchase price
renewal price
recurring discount
one-time coupon
mandatory IPv4/location/setup/tax fees
```

#### 4. Region preference — optional, never force expertise the user does not have

Ask:

> **机房地区有偏好吗？** 没概念也没关系，我可以按你的运营商和用途来推荐。
>
> - 日本
> - 香港
> - 韩国 / 新加坡
> - 美国西海岸
> - 没概念，按实测和需求推荐

A region preference is not a route-quality proof. If the preferred region tests poorly, explain the trade-off and offer a better-performing alternative.

### 3.4 Ask expectations in human language, then translate them into metrics

Users should **not** be expected to know terms such as P95 jitter, packet-loss thresholds or sustained Mbps requirements.

Ask about the experience they want. Translate that answer into technical acceptance criteria yourself.

Useful conditional follow-ups include:

> **你最不能接受哪种情况？**
> - 跟 AI 聊着聊着断 / 卡住
> - 看视频老转圈
> - 游戏延迟忽高忽低
> - 下载太慢
> - 晚高峰偶尔抖一下也能接受，只要便宜

Then map the answer:

```text
AI chat continuity
  -> prioritize connection success, terminal loss, timeout rate, stable latency, UDP if HY2

video / streaming
  -> prioritize sustained throughput and peak-hour throughput stability

gaming
  -> prioritize RTT, jitter, terminal loss and UDP

download-heavy
  -> prioritize sustained throughput, monthly traffic and port limits

price-tolerant of occasional jitter
  -> allow lower route score if the price advantage is meaningful
```

Another useful question when the user says “快就行” is:

> **你说的“够快”，更接近哪一种？**
> - AI / 网页顺滑就够
> - 4K 视频要稳定
> - 经常大文件下载
> - 游戏要低延迟

The purpose is not to force the user to name Mbps. The agent should derive a reasonable throughput/latency target from the scenario and explain the assumption.

### 3.5 Only ask follow-ups that can change the recommendation

After the four-question fast path, ask **at most a few conditional follow-ups**. Do not turn procurement into a questionnaire marathon.

High-value follow-ups include:

#### Willingness to test / tinker

> **你愿意先测几家、买一两台月付试机再退掉不合适的吗？还是更想一次选稳一点、少折腾？**

This changes how aggressively to shortlist small vendors, refundable plans and experimental routes.

#### Price-risk preference

> **你更偏向哪种买法？**
> - 月付贵一点也行，先稳妥试
> - 愿意等活动 / 优惠码
> - 可以年付，但必须确认续费价和退款规则

This controls how much weight to give recurring discounts versus lock-in risk.

#### Availability tolerance

> **晚高峰偶尔抖一下你能接受吗，还是这台必须一直稳？**

If the user requires high availability, consider whether a backup node or optional REALITY entry is justified **after** the primary path is proven healthy.

#### Extra workloads

Only ask detailed CPU/RAM/disk questions when the user plans to run websites, databases, Docker, builds or other services. For a relay-only user, do not make them choose CPU models they do not understand.

### 3.6 Infer the triangle instead of forcing the user to score it

Do not require the user to answer:

```text
“线路、价格、配置分别权重多少？”
```

Most users cannot meaningfully assign percentages before they understand VPS networking.

Infer an initial triangle from the four core answers and the optional expectation follow-up, then show it back in plain language for correction.

Example:

```text
我先按这个方向筛：
线路 > 价格 > 配置。
原因：你主要用 AI，预算 ¥60/月，机器只做个人中转，所以配置先跨过 1C1G 的够用线，把更多预算留给线路。
如果这个取舍不对，你告诉我，我再改。
```

This is preferable to asking a novice to invent percentages.

### 3.7 Produce a short procurement brief before searching vendors

Before naming providers, summarize what was learned in a compact brief. Keep it readable to both the human and another AI.

Example:

```text
Use case: personal HY2 relay, mainly AI
Local network: China Telecom home broadband
Budget: <= ¥60/month, compare by renewal price
Triangle: route > price > config
Baseline: Linux, root, public IPv4, UDP, 1C1G enough unless tests show otherwise
Experience target: AI chat should remain stable; occasional small throughput variation is acceptable
Regions: Tokyo / Osaka first, Seoul acceptable
Purchase preference: monthly first, refundable preferred, recurring discounts welcome
```

If a missing detail could materially reverse the recommendation, ask before searching. Otherwise state the assumption and proceed.

### 3.8 For users in mainland China, explain route labels instead of blindly ranking them

Use these as **reference labels only**:

```text
China Telecom
  ordinary: ChinaNet / 163 / commonly AS4134
  higher-quality candidate: CN2 / AS4809; marketing may say CN2 GIA

China Unicom
  ordinary: 169 / commonly AS4837
  higher-quality candidate: AS9929 + AS10099 / CUP / China Unicom Premium

China Mobile
  ordinary international: CMI / commonly AS58453
  higher-quality candidate: CMIN2 / AS58807
```

For Japan and other overseas segments, names such as:

```text
SoftBank / AS17676
IIJ / AS2497
NTT / AS2914
```

usually describe an overseas backbone/transit segment. They are **not the same category** as CN2 / 9929 / CMIN2. A single end-to-end path may contain both.

Never tell the user that a label alone proves quality. “CN2”, “9929”, “CMIN2”, “SoftBank”, “IIJ”, “NTT”, “three-network optimized”, “premium network”, “native IP”, and similar phrases are shortlist clues, not evidence.

### 3.9 Search and shortlist like a procurement assistant

Only after the user's need is sufficiently understood should current vendor inventory, prices and promotions be researched.

When current vendor inventory, prices or promotions matter, use fresh public information rather than memory.

For each candidate collect:

1. provider and exact plan;
2. datacenter city / region;
3. actual checkout price;
4. renewal price;
5. IPv4, location, setup, tax and other mandatory fees;
6. CPU / RAM / disk;
7. port bandwidth and monthly traffic;
8. public IPv4 / NAT status;
9. UDP policy;
10. AUP / ToS compatibility with the user's intended personal use;
11. test IP / Looking Glass;
12. reinstall / console / recovery capabilities;
13. refund / cancellation / IP replacement rules;
14. current discount type and whether it recurs.

Prefer a shortlist of roughly 5–10 candidates first, then reduce to 2–3 after hard requirements and route tests.

### 3.10 Treat discounts as a useful price lever, not as proof that a server is worth buying

Compare **effective long-term monthly cost**, not only the largest promotional number:

```text
plan
+ IPv4
+ location surcharge
+ setup fee
+ taxes / mandatory add-ons
------------------------------
actual paid term
```

Explicitly distinguish:

- first-month discount;
- first-term / first-year discount;
- recurring discount;
- coupon code;
- anniversary / seasonal / Black Friday-style promotion;
- normal renewal price.

A good recurring discount can improve the price corner of the triangle without sacrificing route/configuration.

However, for an untested provider or route:

```text
monthly test first
      ↓
validate route + instance
      ↓
then consider annual / long-term discount
```

Do not encourage the user to lock into a long prepaid term merely because the advertised annual price is low.

### 3.11 Guide the human through route testing before purchase

When a Test IP / Looking Glass exists, instruct the user to test from the network they will actually use.

Typical Windows checks:

```powershell
ping TEST_IP -n 50
tracert -d TEST_IP
pathping TEST_IP
```

Typical Linux/macOS checks:

```bash
ping -c 50 TEST_IP
mtr -rwzbc 50 TEST_IP
```

At minimum compare:

```text
daytime
20:00–23:00 local peak hours
```

Evaluate:

- terminal packet loss;
- median / average / P95 latency where available;
- jitter / max spikes;
- route changes;
- repeated timeouts;
- sustained throughput;
- the user's own ISP specifically.

Do not treat an intermediate traceroute hop that ignores/deprioritizes ICMP as automatic end-to-end loss.

Where possible, examine both directions. Internet routing can be asymmetric; a good outbound traceroute from the user's device does not prove the return path is equally good.

### 3.12 HY2 candidates require explicit UDP validation

Do not infer UDP health from TCP/HTTPS success.

For a purchased test instance, validate the actual intended UDP port. If useful, compare common and high ports, for example:

```text
UDP 443
UDP 24443
```

If one reaches the VPS and the other does not, investigate path/port-specific handling before blaming HY2 itself.

QUIC/HY2 does not require UDP 443 specifically. Use the port that is actually reachable and stable for the user's path.

### 3.13 Score candidates according to the user's triangle, then explain the trade-off

For a daily HY2 / AI relay, a reasonable example is:

```text
route / stability  50%
price              25%
configuration      15%
operations / AUP   10%
```

For a combined application/database server, configuration may deserve equal weight with routing. For a backup node, price may deserve more weight.

There is no universal weight set.

The final recommendation should tell the user:

```text
why candidate A ranks first
what candidate A sacrifices
why candidate B may be better for a different priority
which triangle corner each candidate optimizes
which result is based on marketing and which is based on real measurement
```

The procurement goal is **not** “find the biggest plan”. It is:

> **find the best personal optimum among price, configuration and route quality for this user's actual workload and local network.**

### 3.14 Human purchase boundary

The agent should guide comparison, testing and checkout interpretation, but the human should make the final purchase decision.

Do not ask the user to paste payment-card data, provider passwords or other financial/account secrets into prompts, logs, Issues or the repository.

After purchase, re-test the **real assigned VPS**, because provider Test IP performance does not prove every production instance is identical.

Only after the instance passes procurement acceptance should the deployment flow continue to SSH bootstrap.

---

## 4. Human SSH bootstrap boundary

An AI must not assume it can safely operate a fresh VPS merely because the user knows its IP/password.

Before autonomous or semi-autonomous deployment, require a one-time human bootstrap unless secure key-based access already exists.

Recommended flow:

```text
Human logs in once using provider password / console
        ↓
Human generates SSH key pair locally
        ↓
Only the public key is installed on the VPS
        ↓
Human verifies key-based SSH login
        ↓
Local AI/Agent uses the already-working ssh / SSH agent / SSH config
```

Rules:

1. SSH private keys remain on the user's local device.
2. Only the `.pub` key goes to `authorized_keys`.
3. Never ask the user to paste a root password or private key into chat, GitHub, Issues, logs, or prompts.
4. Prefer invoking the user's existing local `ssh`, SSH Agent, or `~/.ssh/config` over reading/exporting private keys.
5. Do not disable password login until key-based login is proven to work.
6. Preserve provider-console recovery where possible.
7. If secure SSH access cannot be established, stop and ask the human to complete bootstrap.

Windows example:

```powershell
ssh-keygen -t ed25519
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@SERVER_IP "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
ssh root@SERVER_IP
```

The final `ssh` must succeed before unattended changes continue.

---

## 5. Resource contract

Distinguish **required for the selected profile** from **optional enhancements**.

### Required for HY2 baseline

- VPS / Linux host controlled by the user;
- root or sudo access;
- provider policy permitting intended personal VPN/proxy use;
- public server address;
- UDP reachability on selected HY2 port;
- compatible client;
- HY2 password/auth material;
- TLS material accepted by the HY2 client;
- normal VPS outbound Internet access;
- verified SSH key-based administration path before AI-controlled deployment, unless an equally secure path already exists.

Resolve first:

```text
SERVER_IP_OR_HOSTNAME
SSH_ACCESS_METHOD
HY2_UDP_PORT
HY2_TLS_CERTIFICATE_STRATEGY
CLIENT_TYPE
```

### Additional resources for WARP

Only if WARP is selected:

```text
Cloudflare WARP Linux client / account state
local proxy mode support
chosen local proxy port (repo example: 127.0.0.1:40000)
```

Do not make WARP the host-wide default route in this project.

### Additional resources for fixed SOCKS5

Only if a fixed egress is selected:

```text
SOCKS_HOST
SOCKS_PORT
SOCKS_USERNAME (if required)
SOCKS_PASSWORD (if required)
```

The upstream must be trusted and legitimately usable by the user.

### Additional resources for REALITY

Only if REALITY backup inbound is selected:

```text
VLESS UUID
REALITY X25519 private/public key pair
REALITY short ID
serverName/target strategy
reachable TCP port
```

A domain is not inherently required for REALITY itself.

### Additional resources for Cloudflare Tunnel

Only if the emergency Tunnel entry is selected:

```text
Cloudflare account
appropriate domain / tunnel configuration
secret/token handling path
```

### Not universally required

Do not require these merely to run the project:

- residential IP;
- fixed SOCKS5;
- WARP;
- REALITY;
- Cloudflare Tunnel;
- second VPS;
- IPv6.

---

## 6. Routing policy semantics

Repository example policy may use:

```text
ordinary traffic                 -> direct
OpenAI / ChatGPT / Codex         -> WARP
Gemini / Google AI               -> WARP
Claude / Anthropic               -> optional fixed SOCKS5
```

Treat this as a maintainer preference / example policy, not a universal service requirement.

Use wording such as:

> Some services may behave differently across regions or data-center IP ranges. A separate explicitly selected egress can make routing, troubleshooting, and egress stability more predictable.

Avoid unsupported claims such as:

- "this prevents bans";
- "this IP can never be blocked";
- "Claude requires residential IP";
- "WARP is a residential IP".

---

## 7. WARP architecture rule

Prefer **WARP Local Proxy / WarpProxy mode**:

```text
Linux default route -> VPS native network

Xray selected route
   -> 127.0.0.1:40000
   -> warp-svc
   -> Cloudflare WARP
```

Never make global WARP routing the baseline design.

Why:

- keeps SSH/system updates on native route;
- prevents fixed SOCKS upstream connections from accidentally traversing WARP;
- keeps Direct as a clean baseline;
- limits WARP failure impact to the routes that explicitly use it.

Official references:

- <https://developers.cloudflare.com/warp-client/get-started/linux/>
- <https://developers.cloudflare.com/warp-client/warp-modes/>

Cloudflare CLI syntax changes. Always prefer current docs plus local `warp-cli --help` over historical commands.

### Watchdog policy

Two separate concerns:

```text
Xray lifecycle             -> systemd service
WARP real-egress health    -> connectivity watchdog
```

For WARP health, test real requests through the local proxy. A live `warp-svc` process is not sufficient evidence.

Prefer a systemd timer for new/public deployments. Cron may be mentioned only as a historical/simple alternative.

---

## 8. Fixed SOCKS5 boundary

SOCKS5 itself does not provide transport encryption.

When documenting a remote fixed SOCKS5:

- state that SOCKS itself is not an encrypted VPN;
- note HTTPS still protects HTTPS application payloads end-to-end;
- if stronger link confidentiality is required, use a controlled encrypted/private path or provider that supplies one.

Do not equate "fixed/residential IP" with "encrypted" or "safer transport".

For deliberately pinned destinations, prefer fail-closed behavior unless the user explicitly requests fallback.

Do not silently reroute fixed-egress traffic to Direct when the upstream fails.

---

## 9. Audited client reality

Current Windows audit:

```text
v2rayN 7.24.2
HY2 active       -> sing-box 1.13.14
REALITY profile  -> Xray core
TUN + Rule mode
```

v2rayN is a GUI/config manager, not one specific core.

Observed model:

```text
GUI metadata + global preferences
              ↓
        v2rayN generates
              ↓
core-specific runtime config
```

Keep sing-box, Xray and Mihomo field names separate.

---

## 10. Xray source of truth

Prefer current official Xray docs over copied blog configs or remembered production aliases:

- Hysteria inbound: <https://xtls.github.io/config/inbounds/hysteria.html>
- Hysteria transport: <https://xtls.github.io/config/transports/hysteria.html>
- VLESS / Vision: <https://xtls.github.io/config/inbounds/vless.html>
- REALITY: <https://xtls.github.io/config/transports/reality.html>
- RAW: <https://xtls.github.io/config/transports/raw.html>
- SOCKS outbound: <https://xtls.github.io/config/outbounds/socks.html>
- Installer: <https://github.com/XTLS/Xray-install>

Current assumptions:

- current Xray supports Hysteria2;
- Hysteria version is 2;
- HY2 is the primary entry;
- REALITY/Vision is optional backup;
- current docs may use fields that differ from historical production aliases;
- REALITY current server-side docs use `target`; historical configs may show `dest`;
- current SOCKS outbound docs use flat address/port/user/pass fields; old configs may use older shapes.

If upstream schema changes, update every affected example/doc together.

---

## 11. Repository file map

```text
README.md                              Chinese human-facing architecture and profile guidance
AGENTS.md                              AI procurement/deployment/maintenance contract
examples/xray-server.example.jsonc     server-side Xray schema example
examples/v2rayn-hysteria2.example.md   audited v2rayN / sing-box HY2 client example
examples/v2rayn-reality-vision.example.md
                                       v2rayN / Xray REALITY client example
docs/vps-selection.md                  pre-deployment VPS procurement, route and UDP testing guide
docs/warp-outbound.md                  WARP egress behavior and validation
docs/static-socks.md                   fixed SOCKS5 egress behavior
```

Synchronization rules:

- VPS procurement/selection/testing guidance change -> VPS selection doc + relevant README entry points + AGENTS procurement contract.
- Xray schema change -> server example + README references + affected docs + AGENTS.
- HY2 client field change -> HY2 client example + relevant README notes.
- REALITY client field change -> REALITY example + relevant README notes.
- WARP policy/CLI change -> WARP doc + server example if needed + AGENTS.
- SOCKS schema/policy change -> static SOCKS doc + server example + AGENTS if safety boundary changes.

Never leave contradictory assumptions across files.

---

## 12. Production host is not the template

Production evolved through experiments and may contain legacy paths/backups/compatibility fields.

Public examples should follow:

```text
working production behavior
        ↓
read-only audit
        ↓
current upstream documentation
        ↓
remove secrets + legacy/dead config
        ↓
clean public example
```

Prefer clean layouts such as:

```text
/usr/local/bin/xray
/usr/local/etc/xray/config.json
systemd-managed services
explicit firewall rules
minimal configs
```

---

## 13. Secret handling

Never commit or paste live secrets.

Placeholders only:

```text
YOUR_SERVER_IP
YOUR_UUID
YOUR_REALITY_PRIVATE_KEY
YOUR_REALITY_PUBLIC_KEY
YOUR_SHORT_ID
YOUR_HY2_PASSWORD
YOUR_DOMAIN
YOUR_SERVER_NAME
YOUR_SOCKS_HOST
YOUR_SOCKS_USERNAME
YOUR_SOCKS_PASSWORD
```

Never expose:

- SSH private keys;
- root/VPS passwords;
- TLS private keys;
- Cloudflare tokens;
- production proxy credentials;
- subscription URLs;
- full production configs containing secrets.

---

## 14. Firewall / listeners

Open only selected-feature ports.

Example full profile:

```text
TCP 22      SSH
TCP 443     REALITY
UDP 24443   HY2
```

An HY2 + WARP + fixed-SOCKS Profile D does not need REALITY TCP 443 unless REALITY is actually enabled.

Preserve SSH access before firewall changes. Remember provider-side security groups may exist independently of host firewall.

---

## 15. Testing philosophy

Change one variable at a time.

Before buying a VPS, follow [`docs/vps-selection.md`](./docs/vps-selection.md): do not infer route quality, UDP health or peak-hour performance from marketing specs alone.

Prefer checking:

- connection success rate;
- latency / variance;
- timeouts;
- peak-hour behavior;
- sustained transfer stability;
- long-lived connection behavior;
- final egress IP for each route.

Do not call a path healthy from one speed test.

### AI deployment self-check

Before saying deployment is complete, verify the features actually selected:

1. SSH key-based administration still works.
2. Server config validation passes.
3. Expected listeners are present and unnecessary example ports are closed.
4. HY2 client/server credentials match.
5. Direct egress works.
6. If WARP is enabled, a request through the local proxy reaches the intended WARP egress.
7. If fixed SOCKS5 is enabled, selected traffic reaches that fixed egress and does not silently fall back.
8. If REALITY is enabled, client/server UUID/key/short-ID/serverName parameters match.
9. Firewall/provider rules permit only the intended management/proxy ports.
10. No real secrets were written to repository, logs, Issues, or chat output.

Do not claim end-to-end client success unless the client path was actually tested.

---

## 16. Documentation style

`README.md` is for Chinese-speaking humans: explain the practical recommendation first.

`AGENTS.md` is for AI maintainers/deployers: preserve procurement guidance, architecture, profile semantics, SSH bootstrap boundary, resource prerequisites, safety boundaries, source-of-truth links and secret hygiene.

The repository must remain understandable without private production context.