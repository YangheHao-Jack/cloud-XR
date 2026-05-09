# CloudXR + Isaac Sim + Quest 3 — Install & Run Guide

Reproduces the working setup in the exact order it was built. Target:

- **Server** (Linux, RTX 5090, Ubuntu 24.04, driver ≥ 573.42, Isaac Sim 5.1.0.0 + Isaac Lab 2.3.2.post1 already installed in `~/isaac_env`)
- **CloudXR Runtime 6.0.5** (Linux SDK tarball — server side, provides OpenXR runtime)
- **CloudXR.js 6.1.0** (browser client on the Quest)
- **wss-proxy** (HAProxy in Docker — HTTPS ↔ runtime WS bridge)
- **Meta Quest 3** as the XR client, browser-based (no sideloaded APK)
- **OpenXR Intrinsics API Layer** (Part 9 — captures per-eye FoV + K matrix from the live XR session, since Isaac Sim exposes none of this through any Python / USD / settings interface)
- **Wireless ADB** (Part 10 — for cable-free Android Studio deploys of companion APKs to the Quest)

---

## Part 0 — Network setup (IPs, connectivity, link quality)

The Linux box and the Quest must be on the **same Wi-Fi network** (same subnet).
You'll reuse these addresses in Parts 4, 5, and everywhere a URL appears below.

### 0.1 Linux server IP

```bash
# Simplest — takes the first non-loopback IPv4
hostname -I | awk '{print $1}'

# More detailed (shows which interface)
ip -4 addr show | grep -E "inet " | grep -v 127.0.0.1
# Look for something like 192.168.1.154 on wlp*/enp*/eth0
```

If you see multiple IPs, pick the one on the same subnet as the Quest (usually `192.168.x.x` or `10.0.x.x`). Throughout this guide that address appears as `<LINUX_IP>` — in the author's setup it was `192.168.1.154`.

### 0.2 Quest IP

On the Quest headset:

1. **Settings** (gear icon in the universal menu)
2. **Wi-Fi** → tap the **connected network** → scroll down → note **IP address**

This appears as `<QUEST_IP>` below. Only needed for firewall verification from Linux; the Quest browser itself reaches the Linux box by `<LINUX_IP>`, not the other way around.

### 0.3 Verify connectivity both directions

```bash
# From Linux, can you reach the Quest?
ping -c 3 <QUEST_IP>

# Check firewall isn't blocking the Quest-facing ports yet (they won't work until §1.4 opens them):
ss -tlnp | grep -E "8080|48010|48322"   # should be empty before setup
```

From the Quest, quickly verify you can reach the Linux IP: open Meta Browser and visit `http://<LINUX_IP>` — you'll get a "can't connect" page, which is fine; we just want to confirm the address resolves (no DNS error, just a connection-refused page).

If `ping` fails, the devices are on different networks or AP isolation is enabled on your router. CloudXR will not work until that's fixed.

### 0.4 Network quality requirements (CRITICAL for CloudXR)

CloudXR streams two stereo eye buffers as compressed video at 60–90 Hz from
Linux to the Quest. This is roughly 30–100 Mbps with sub-frame timing
constraints. The link between them must be:

| Metric             | Healthy             | Bad (will corrupt) |
|--------------------|---------------------|--------------------|
| Average ping       | <10 ms              | >20 ms             |
| Max ping           | <20 ms              | >50 ms             |
| Ping variance (mdev) | <5 ms             | >10 ms             |
| Packet loss        | 0%                  | any                |
| Both devices on    | 5 GHz Wi-Fi         | 2.4 GHz            |
| 5 GHz channel      | 36 / 40 / 44 / 48   | 60+ (DFS)          |

**Symptoms of bad network during CloudXR session** — tearing, macroblocking
(rainbow squares), freezing, ghosting, flickering — especially correlated
with head motion or scene motion. Encoded video frame size spikes when
content changes; if Wi-Fi can't deliver the burst before the next frame
deadline, you get all five symptoms at once.

#### 0.4.1 Measure your link

```bash
# Continuous-motion test — run while moving the headset / scene actively for ~10 seconds
ping -c 60 -i 0.2 <QUEST_IP> > /tmp/ping.log &
# (move/turn during ping)
wait
tail -5 /tmp/ping.log
```

A summary line appears at the end like
`rtt min/avg/max/mdev = 1.234/2.567/3.890/0.456 ms`. If `max` is more than
20 ms or `mdev` is more than 5 ms, the link cannot reliably stream CloudXR
under load.

#### 0.4.2 Check what band each device is on

```bash
# Linux band
iwconfig 2>/dev/null | grep Frequency
# Frequency 2.4xx GHz → 2.4 GHz (BAD)
# Frequency 5.xx GHz  → 5 GHz   (good)

# Surrounding networks (channel congestion check)
nmcli -f IN-USE,SSID,CHAN,SIGNAL,RATE dev wifi | head -20
# CHAN  1, 6, 11             → 2.4 GHz
# CHAN 36-48                 → 5 GHz, non-DFS, preferred
# CHAN 52-144                → 5 GHz, DFS (radar shared, can drop)
# CHAN 149-165               → 5 GHz, non-DFS, preferred
```

Quest band: **Settings → Wi-Fi → tap connected network → Frequency/Band**.
Confirm 5 GHz. If it shows 2.4 GHz, either the router pushed it there for
range reasons, or you're connected to a 2.4-only SSID.

#### 0.4.3 Fixes (best-to-worst)

**1. Plug Linux into Ethernet** (single biggest win, ~£5 cable, zero config)

When Linux is wired and Quest is wireless, they don't compete for airtime.
The Linux→router leg is 0.5 ms wired; only the router→Quest leg uses
Wi-Fi. Total path becomes consistently <10 ms.

```bash
# Verify after plugging cable in (Linux may need to reconnect)
ip -4 addr show | grep "inet "
# A new IP will appear on eth0/enp*/similar
```

If the Ethernet IP differs from the previous Wi-Fi IP, tell the Quest the
new server IP (`<LINUX_IP>` field in the CloudXR.js form on `:8080`).

**2. Switch the router's 5 GHz channel to 36, 40, 44, or 48**

These are non-DFS — never affected by radar interruptions. Login to the
router admin (usually `http://192.168.1.1`, password on a sticker on the
router) → Wireless → 5 GHz channel → set to one of the above. Reconnect
both devices, retest §0.4.1.

If your router doesn't expose channel selection (some ISP boxes lock it),
move on to fix 3.

**3. Move the headset and Linux box closer to the router**

Every wall and metre adds latency under congestion. A cable trace of
"server in one room, router in another, headset in a third" multiplies
contention symptoms.

**4. Powerline adapter (Ethernet-over-mains)**

If a physical Ethernet run isn't possible, powerline adapters (~£30 a
pair) give around 50% of true Ethernet performance and require no new
cabling between rooms. Plug one near the router, one near the Linux box,
patch each to its device with a short cable.

**5. Last resort: cap CloudXR's peak bitrate**

If the network genuinely is what it is, you can tell CloudXR to stay
within what the link can sustain. This adds two `nv_cxr_service_set_*`
calls in `cxr_service.cpp`. Trade-off: softer picture but stable. Only
worth it if 1–4 are all blocked. See `cxr_service.cpp` and the CloudXR
SDK property reference for `bitrate-mbps` and similar keys.

#### 0.4.4 What "good" looks like once fixed

```bash
ping -c 30 -i 0.2 <QUEST_IP>
# rtt min/avg/max/mdev = 1.123/2.456/3.789/0.234 ms
# 0% packet loss
```

CloudXR session: rock-steady picture during head turns and continuous
scene motion. No tearing, no macroblocks, no flicker.

---

## Part 1 — Prerequisites on the Linux workstation

### 1.1 NVIDIA driver

```bash
nvidia-smi | head -5      # driver version top-right; must be ≥ 573.42
```

### 1.2 Docker + NVIDIA Container Toolkit

Docker is only needed for the **wss-proxy** container. The CloudXR runtime itself runs bare-metal.

```bash
# Docker Engine
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Let your user run docker without sudo (log out/in after this, or `newgrp docker`)
sudo usermod -aG docker $USER
newgrp docker
docker run --rm hello-world         # smoke test

# NVIDIA Container Toolkit (optional — only needed if you later want Isaac Lab in Docker too)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 1.3 Node 20 for CloudXR.js build

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version                     # v20.x
npm --version                      # 10.x
```

### 1.4 Firewall

```bash
sudo ufw allow 47998:48000,48005,48008,48012/udp     # CloudXR media
sudo ufw allow 48010/tcp                             # CloudXR signaling
sudo ufw allow 48322/tcp                             # wss-proxy (browser ↔ runtime)
sudo ufw allow 8080/tcp                              # CloudXR.js HTTPS dev server
```

---

## Part 2 — CloudXR Runtime 6.0.5

### 2.1 Obtain the tarball

Apply for access: https://developer.nvidia.com/cloudxr-sdk-early-access-program/join (mention Isaac Sim use cases). After approval, NGC shows the SDK download. Grab `CloudXR-6.0.5-Linux-sdk.tar.gz` into `~/Downloads/`.

### 2.2 Extract to `~/cloudxr-runtime/`

**Important:** create the directory first, `cd` into it, *then* extract — the tarball has no top-level folder, so running `tar -xzf` from `~/` scatters files across your home directory.

```bash
mkdir -p ~/cloudxr-runtime
cd ~/cloudxr-runtime
tar -xzf ~/Downloads/CloudXR-6.0.5-Linux-sdk.tar.gz

# Verify
cat VERSION                        # 6.0.5
cat openxr_cloudxr.json            # {"file_format_version":"1.0.0","runtime":{"name":"NVIDIA™ CloudXR™ Runtime (based on Monado™)","library_path":"./libopenxr_cloudxr.so"}}
ls                                 # libcloudxr.so, libNvStreamServer.so, libopenxr_cloudxr.so,
                                   # libcudart.so.12*, libssl*, libPoco.so, plugins/, include/, etc.
```

### 2.3 Write & compile the service launcher

Critical: the 6.0.5 SDK **ships no pre-built service binary** (`find ~/cloudxr-runtime -type f -executable` returns nothing). `XR_RUNTIME_JSON` alone is not enough — we verified with `strace` that Isaac Sim's OpenXR client calls `connect()` on `/run/user/1000/ipc_cloudxr` expecting the socket to already exist, and the library never spawns a service process itself.

Per Option 3 of the CloudXR Runtime docs, build a tiny launcher that calls `nv_cxr_service_create` → `nv_cxr_service_start` → `nv_cxr_service_join`. Save as `~/cloudxr-runtime/cxr_service.cpp`:

```cpp
// cxr_service.cpp
//
// Minimal CloudXR service launcher for Linux.
// Implements Option 3 from:
//   https://docs.nvidia.com/cloudxr-sdk/latest/usr_guide/cloudxr_runtime/getting_started.html
//
// Creates, starts, and joins a CloudXR service so that OpenXR apps
// (e.g. Isaac Sim with XR_RUNTIME_JSON=.../openxr_cloudxr.json) can
// connect to the CloudXR IPC socket at /run/user/$UID/ipc_cloudxr.
//
// Build:
//   cd ~/cloudxr-runtime
//   g++ -std=c++17 -Iinclude cxr_service.cpp -L. -lcloudxr \
//       -Wl,-rpath,'$ORIGIN' -o cxr-service

#include <cstdio>
#include <cstdlib>
#include <csignal>
#include <atomic>
#include <thread>
#include <chrono>

#include <cxrServiceAPI.h>

static std::atomic<bool>    g_stop_requested{false};
static struct nv_cxr_service* g_service = nullptr;

static const char* result_name(nv_cxr_result_t r) {
    switch (r) {
        case NV_CXR_SUCCESS:                        return "SUCCESS";
        case NV_CXR_INTERNAL_SERVICE_ERROR:         return "INTERNAL_SERVICE_ERROR";
        case NV_CXR_STARTUP_FAILED:                 return "STARTUP_FAILED";
        case NV_CXR_NULL_OBJECT:                    return "NULL_OBJECT";
        case NV_CXR_NULL_PTR:                       return "NULL_PTR";
        case NV_CXR_SERVICE_ALREADY_STARTED:        return "SERVICE_ALREADY_STARTED";
        case NV_CXR_SERVICE_NOT_STARTED:            return "SERVICE_NOT_STARTED";
        case NV_CXR_PROPERTY_NAME_MALFORMED:        return "PROPERTY_NAME_MALFORMED";
        case NV_CXR_PROPERTY_NAME_INVALID:          return "PROPERTY_NAME_INVALID";
        case NV_CXR_ERROR_PROPERTY_VALUE_INVALID:   return "PROPERTY_VALUE_INVALID";
        case NV_CXR_ERROR_BUFFER_SIZE_INSUFFICIENT: return "BUFFER_SIZE_INSUFFICIENT";
        case NV_CXR_ERROR_PROPERTY_READ_ONLY:       return "PROPERTY_READ_ONLY";
        case NV_CXR_PORT_UNAVAILABLE:               return "PORT_UNAVAILABLE";
        case NV_CXR_ERROR_PROPERTY_WRITE_ONLY:      return "PROPERTY_WRITE_ONLY";
    }
    return "(unknown)";
}

static void handle_signal(int sig) {
    fprintf(stderr, "\n[cxr-service] signal %d received, stopping...\n", sig);
    g_stop_requested = true;
    if (g_service) nv_cxr_service_stop(g_service);
}

static void event_loop() {
    while (!g_stop_requested.load()) {
        nv_cxr_event_t event{};
        nv_cxr_result_t r = nv_cxr_service_poll_event(g_service, &event);
        if (r == NV_CXR_SUCCESS) {
            switch (event.type) {
                case NV_CXR_EVENT_CLOUDXR_CLIENT_CONNECTED:
                    printf("[event] CloudXR client CONNECTED\n"); break;
                case NV_CXR_EVENT_CLOUDXR_CLIENT_DISCONNECTED:
                    printf("[event] CloudXR client DISCONNECTED\n"); break;
                case NV_CXR_EVENT_OPENXR_APP_CONNECTED:
                    printf("[event] OpenXR app CONNECTED (e.g. Isaac Sim)\n"); break;
                case NV_CXR_EVENT_OPENXR_APP_DISCONNECTED:
                    printf("[event] OpenXR app DISCONNECTED\n"); break;
                case NV_CXR_EVENT_NONE: default: break;
            }
            fflush(stdout);
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }
}

int main() {
    signal(SIGINT,  handle_signal);
    signal(SIGTERM, handle_signal);

    uint32_t la_maj=0, la_min=0, la_patch=0;
    uint32_t rt_maj=0, rt_min=0, rt_patch=0;
    nv_cxr_get_library_api_version(&la_maj, &la_min, &la_patch);
    nv_cxr_get_runtime_version(&rt_maj, &rt_min, &rt_patch);
    printf("[cxr-service] Library API:    %u.%u.%u\n", la_maj, la_min, la_patch);
    printf("[cxr-service] Runtime version: %u.%u.%u\n", rt_maj, rt_min, rt_patch);

    printf("[cxr-service] Creating service...\n");
    nv_cxr_result_t r = nv_cxr_service_create(&g_service);
    if (r != NV_CXR_SUCCESS) {
        fprintf(stderr, "[cxr-service] nv_cxr_service_create FAILED: %s\n", result_name(r));
        return 1;
    }

    // Device profile "auto-webrtc" enables Quest 3 via browser (CloudXR.js).
    {
        const char prop[] = "device-profile";
        const char val[]  = "auto-webrtc";
        r = nv_cxr_service_set_string_property(g_service,
                                               prop, sizeof(prop) - 1,
                                               val,  sizeof(val)  - 1);
        if (r != NV_CXR_SUCCESS) {
            fprintf(stderr, "[cxr-service] set device-profile FAILED: %s\n", result_name(r));
            nv_cxr_service_destroy(g_service);
            return 3;
        }
        printf("[cxr-service] Device profile set to: %s\n", val);
    }

    printf("[cxr-service] Starting service (opens IPC socket at /run/user/$UID/ipc_cloudxr)...\n");
    r = nv_cxr_service_start(g_service);
    if (r != NV_CXR_SUCCESS) {
        fprintf(stderr, "[cxr-service] nv_cxr_service_start FAILED: %s\n", result_name(r));
        nv_cxr_service_destroy(g_service);
        return 2;
    }
    printf("[cxr-service] Service started. Ready for OpenXR apps. Press Ctrl+C to stop.\n");
    fflush(stdout);

    std::thread events(event_loop);

    r = nv_cxr_service_join(g_service);
    if (r != NV_CXR_SUCCESS && r != NV_CXR_SERVICE_NOT_STARTED)
        fprintf(stderr, "[cxr-service] nv_cxr_service_join returned: %s\n", result_name(r));

    g_stop_requested = true;
    if (events.joinable()) events.join();

    printf("[cxr-service] Destroying service...\n");
    nv_cxr_service_destroy(g_service);
    g_service = nullptr;
    printf("[cxr-service] Shutdown complete.\n");
    return 0;
}
```

Compile:

```bash
cd ~/cloudxr-runtime
g++ -std=c++17 -Iinclude cxr_service.cpp -L. -lcloudxr \
    -Wl,-rpath,'$ORIGIN' -o cxr-service

ls -la cxr-service        # ~35 KB executable
ldd cxr-service | head    # libcloudxr.so → /home/jack/cloudxr-runtime/libcloudxr.so
```

Flag notes: `-Iinclude` finds `cxrServiceAPI.h`; `-L. -lcloudxr` links `libcloudxr.so`; `-Wl,-rpath,'$ORIGIN'` embeds a relative library path so the binary finds its `.so` siblings without needing `LD_LIBRARY_PATH`.

### 2.4 Register as the system OpenXR runtime (on-demand, not always-on)

Isaac Sim's "System OpenXR Runtime" dropdown reads `XR_RUNTIME_JSON`.

**Don't export this globally** — pointing your whole system's OpenXR loader
at CloudXR breaks unrelated things (other XR apps, system updates that
poke OpenXR, etc.). Instead, define an alias that activates the CloudXR
env *only in the terminal that launches Isaac Sim*:

```bash
cat >> ~/.bashrc <<'EOF'

# CloudXR env — activate only in the terminal launching Isaac Sim with XR
alias use-cloudxr='export XR_RUNTIME_JSON=$HOME/cloudxr-runtime/openxr_cloudxr.json; export LD_LIBRARY_PATH=$HOME/cloudxr-runtime:$LD_LIBRARY_PATH'
EOF
source ~/.bashrc
```

Then in *that one terminal* before launching Isaac Sim:

```bash
use-cloudxr
echo $XR_RUNTIME_JSON         # /home/jack/cloudxr-runtime/openxr_cloudxr.json
cat  $XR_RUNTIME_JSON         # JSON manifest with library_path "./libopenxr_cloudxr.so"
```

All other terminals (system updates, unrelated work) stay clean. The
alias is idempotent — running `use-cloudxr` twice is harmless.

### 2.5 Start the runtime (Terminal 1)

```bash
cd ~/cloudxr-runtime
./cxr-service
```

Expected output:

```
[cxr-service] Library API:    1.0.6
[cxr-service] Runtime version: 6.0.5
[cxr-service] Creating service...
[cxr-service] Device profile set to: auto-webrtc
[cxr-service] Starting service (opens IPC socket at /run/user/$UID/ipc_cloudxr)...
[cxr-service] Service started. Ready for OpenXR apps. Press Ctrl+C to stop.
```

Leave this terminal open. When Isaac Sim later connects, you'll see `[event] OpenXR app CONNECTED`. When the Quest browser connects, you'll see `[event] CloudXR client CONNECTED`.

---

## Part 3 — CloudXR.js 6.1.0 (browser client for Quest)

### 3.1 Get the package

From NGC (same Early Access approval), download `nvidia-cloudxr-6.1.0.tgz` (the npm package) and the `cloudxr-js-samples` archive.

### 3.2 Layout at `~/cloudxr-js/`

```
~/cloudxr-js/
├── nvidia-cloudxr-6.1.0.tgz
└── cloudxr-js-samples/
    ├── simple/                 ← the WebGL sample we use
    ├── react/                  ← React Three Fiber variant
    └── proxy/                  ← wss-proxy Docker build
```

The sample's `package.json` references the local tgz via `"@nvidia/cloudxr": "file:../../nvidia-cloudxr-6.1.0.tgz"`, so `npm install` picks it up automatically.

### 3.3 Install dependencies

```bash
cd ~/cloudxr-js/cloudxr-js-samples/simple
npm install
```

### 3.4 Run the HTTPS dev server (Terminal 2)

The Quest browser requires HTTPS for WebXR permissions (camera, local network, immersive session).

```bash
cd ~/cloudxr-js/cloudxr-js-samples/simple
npm run dev-server:https
# Listens on https://0.0.0.0:8080/ with a self-signed cert
```

Leave running.

---

## Part 4 — wss-proxy (browser HTTPS ↔ runtime WS bridge)

The CloudXR runtime speaks plain `ws://` on port 49100. The Quest browser pages over HTTPS can't connect to `ws://` due to mixed-content block. The shipped HAProxy wrapper solves this.

### 4.1 Build the image

```bash
cd ~/cloudxr-js/cloudxr-js-samples/proxy
docker build -t cloudxr-wss-proxy .
```

### 4.2 Run the proxy (Terminal 3, or detached)

Use a persistent Docker volume for the self-signed cert so the Quest doesn't have to re-trust it on every restart:

```bash
docker run -d --name wss-proxy \
  --network host \
  -v cloudxr-proxy-certs:/usr/local/etc/haproxy/certs \
  -e BACKEND_HOST=localhost \
  -e BACKEND_PORT=49100 \
  -e PROXY_PORT=48322 \
  cloudxr-wss-proxy

docker logs -f wss-proxy
# First run: "🔐 Generating self-signed SSL certificate..."
# Subsequent runs: "✅ Using existing SSL certificate..."
```

Stop / start later:

```bash
docker stop wss-proxy
docker start wss-proxy
# Full cert reset:
docker rm wss-proxy
docker volume rm cloudxr-proxy-certs
```

---

## Part 5 — End-to-end launch sequence

Five things must be running in this order:

```bash
# Terminal 1 — CloudXR runtime (§2.5)
cd ~/cloudxr-runtime && ./cxr-service

# Terminal 2 — wss-proxy (one-time setup from §4, then just `docker start wss-proxy`)
docker start wss-proxy && docker logs -f wss-proxy

# Terminal 3 — CloudXR.js HTTPS dev server (§3.4)
cd ~/cloudxr-js/cloudxr-js-samples/simple && npm run dev-server:https

# Terminal 4 — Your Isaac Sim script
use-cloudxr                                         # activate CloudXR env in this terminal only
source ~/isaac_env/bin/activate                     # if venv not auto-activated
python3 ~/test_xr_controller.py --xr --device cuda:0 \
    --or-environment "/media/jack/Seagate Basic/i4h-assets/132c82d/Props/shared_OR_without_Mark/main.usd"
```

In the Isaac Sim UI:

1. **AR panel** → Output Plugin: `OpenXR` → OpenXR Runtime: `System OpenXR Runtime` → **Start AR**
2. Viewport shows two eyes rendered, status "AR profile is active"
3. Terminal 1 prints `[event] OpenXR app CONNECTED`

On the Quest:

1. Open Meta Browser → `https://<LINUX_IP>:8080/`
2. First time: accept the self-signed cert warning (Advanced → Proceed)
3. **First time only** (per Quest + Linux IP combo): also visit
   `https://<LINUX_IP>:48322/` once and accept that cert too. The CloudXR.js
   client opens a WSS to 48322 internally; without trusting that cert,
   `CONNECT` will silently fail. Trust persists across sessions.
4. Back on the `:8080` page, enter:
   - Server IP: `<LINUX_IP>`
   - Port: `48010`
   - Proxy URL: `https://<LINUX_IP>:48322`
5. Click **CONNECT**
6. First connection only: grant camera / local-network / WebXR permissions
7. Scene appears in the headset → head tracking updates `/_xr/stage/xrCamera` transform → thumbsticks route through `OpenXRDevice`
8. Terminal 1 prints `[event] CloudXR client CONNECTED`

---

## Part 6 — Clean shutdown

```bash
# Quest: disconnect from CloudXR.js page
# Isaac Sim: Stop AR, close Isaac Sim
# Terminal 3: Ctrl+C npm
# Terminal 1: Ctrl+C cxr-service  (prints Destroying service... Shutdown complete.)
# Terminal 2 (wss-proxy): leave running, or `docker stop wss-proxy`
```

---

## Part 7 — Troubleshooting

**Isaac Sim AR panel shows no OpenXR runtime / Start AR greyed out**
→ `XR_RUNTIME_JSON` not set. `echo $XR_RUNTIME_JSON` should print `/home/jack/cloudxr-runtime/openxr_cloudxr.json`. If empty, run `use-cloudxr` in this terminal (defined in §2.4) before relaunching Isaac Sim.

**`./cxr-service` exits with "could not load libcloudxr.so"**
→ The rpath flag wasn't used at compile time. Recompile with `-Wl,-rpath,'$ORIGIN'` per §2.3. Alternatively set `LD_LIBRARY_PATH=~/cloudxr-runtime` in that terminal.

**Isaac Sim errors "Failed to connect to socket /run/user/1000/ipc_cloudxr"**
→ `cxr-service` isn't running. Terminal 1 must be alive and showing "Service started" before Isaac Sim starts AR.

**Quest browser: CONNECT button does nothing**
→ One of: (a) firewall missing 48322/tcp or the 47998-48012 range, (b) wss-proxy not running (`docker ps` should show `wss-proxy`), (c) Quest didn't trust the cert at 48322 — visit `https://<LINUX_IP>:48322/` directly once to accept.

**Quest browser loads the page but CONNECT times out**
→ `nc -vz <LINUX_IP> 48010` from another host on the same network should print "succeeded". If not, firewall or routing issue on your LAN.

**Isaac Sim viewport renders one eye, not two, after Start AR**
→ CloudXR service isn't accepting the session. Check Terminal 1 for an error; it may have exited. Restart `./cxr-service`.

**Quest connects, scene visible, but head motion doesn't update xrCamera pose in Python**
→ Read it via `UsdGeom.Xformable(prim).ComputeLocalToWorldTransform(Usd.TimeCode.Default())` **after** Start AR; the prim only exists while AR is active.

**CloudXR.js version mismatch concern**
→ CloudXR.js 6.1.0 is compatible with CloudXR Runtime 6.x (per NVIDIA release notes). 6.0.5 runtime + 6.1.0 client is a supported pairing.

**Picture corrupts during head motion or scene motion (tearing, macroblocking, ghosting, freezing, flickering)**
→ Network bandwidth/latency issue, not a code bug. Run §0.4.1 ping test. If `max` >50 ms or `mdev` >10 ms, fix the network (§0.4.3) — Ethernet on Linux is the single biggest win. CloudXR cannot stream cleanly over a contested Wi-Fi link no matter what the software does.

**Was working, now Quest CONNECT just hangs at "Waiting for connection"; `docker logs wss-proxy` shows `SSL handshake failure ... certificate unknown`**
→ The wss-proxy's self-signed cert expired (typical lifetime ~1 year) or the Quest's trust was wiped. Re-visit `https://<LINUX_IP>:48322/` on the Quest browser, Advanced → Proceed, accept the cert. The CloudXR.js port settings in the form (48010) and the wss-proxy backend (49100) don't change — only the cert trust.

---

## Part 8 — File locations reference

| What                       | Where                                                                    |
|----------------------------|--------------------------------------------------------------------------|
| CloudXR runtime            | `~/cloudxr-runtime/`                                                     |
| OpenXR manifest            | `~/cloudxr-runtime/openxr_cloudxr.json`                                  |
| Service launcher source    | `~/cloudxr-runtime/cxr_service.cpp`                                      |
| Service launcher binary    | `~/cloudxr-runtime/cxr-service`                                          |
| Service API header         | `~/cloudxr-runtime/include/cxrServiceAPI.h`                              |
| IPC socket (runtime-managed) | `/run/user/$UID/ipc_cloudxr`                                           |
| CloudXR.js sample entry    | `~/cloudxr-js/cloudxr-js-samples/simple/src/main.ts`                     |
| wss-proxy Dockerfile       | `~/cloudxr-js/cloudxr-js-samples/proxy/Dockerfile`                       |
| XR camera USD prim         | `/_xr/stage/xrCamera` (exists only while AR is active)                   |
| Intrinsics API layer dir   | `~/xr_intrinsics_layer/`                                                 |
| Intrinsics layer .so       | `~/xr_intrinsics_layer/build/libxr_intrinsics_layer.so`                  |
| Intrinsics layer manifest  | `~/xr_intrinsics_layer/build/xr_intrinsics_layer.json`                   |
| Intrinsics IPC socket      | `/tmp/xr_intrinsics.sock`                                                |

---

## Part 9 — OpenXR Intrinsics API Layer (per-eye FoV + K extraction)

Isaac Sim 5.1 + CloudXR 6.0.5 does not expose per-eye display intrinsics
through any Python, USD, Fabric, or `carb` settings surface. The values shown
in the UI "Fisheye Lens" fields are property-panel widget defaults, not real
runtime values. The only reliable way to get the actual XrFovf angles the
Quest is rendering with is to intercept them at the OpenXR level.

This is done with a small OpenXR API layer that sits between Isaac Sim and
`libopenxr_cloudxr.so`:

```
Isaac Sim  →  libopenxr_loader.so.1  →  intrinsics_layer  →  libopenxr_cloudxr.so
                                               │
                                               ▼  /tmp/xr_intrinsics.sock (UDS)
                                          Python consumer
                                               │
                                               ▼  (fx, fy, cx, cy) per eye, per frame
```

The layer hooks two OpenXR calls:

- **`xrEnumerateViewConfigurationViews`** — called once per session, returns per-eye recommended render resolution
- **`xrLocateViews`** — called every frame, returns `XrFovf` (angleLeft/Right/Up/Down in radians) and view pose per eye

It publishes one JSON line per frame to the Unix socket.

### 9.1 Layer source files

All five files live in `~/xr_intrinsics_layer/`. Create the directory and
each file with the contents below.

```bash
mkdir -p ~/xr_intrinsics_layer
cd ~/xr_intrinsics_layer
```

#### 9.1.1 `xr_intrinsics_layer.cpp` — the layer itself

```cpp
// xr_intrinsics_layer.cpp
//
// OpenXR API layer that sniffs per-eye FoV + recommended resolution from
// Isaac Sim's XR session and publishes them to Python via a Unix socket.
//
// Architecture:
//   Isaac Sim  →  libopenxr_loader.so.1  →  THIS LAYER  →  libopenxr_cloudxr.so
//
// Intercepts:
//   - xrEnumerateViewConfigurationViews  → per-eye recommendedImageRectWidth/Height
//   - xrLocateViews                      → per-eye XrFovf (angleLeft/Right/Up/Down)
//
// Output: newline-delimited JSON on Unix socket /tmp/xr_intrinsics.sock
// Each frame's XrView values are written as one JSON line. A Python consumer
// connects to the socket, reads lines, converts XrFovf→K, and uses them.
//
// Build (see CMakeLists.txt):
//   cmake -B build -DCMAKE_BUILD_TYPE=Release
//   cmake --build build
//
// Register:
//   export XR_API_LAYER_PATH=$PWD/build
//   export XR_ENABLE_API_LAYERS=XR_APILAYER_INTRINSICS_SNIFF
//
// Then run Isaac Sim normally. The layer auto-creates the socket and
// streams values once xrLocateViews starts firing.

#include <openxr/openxr.h>
#include <openxr/openxr_loader_negotiation.h>

#include <algorithm>
#include <atomic>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <mutex>
#include <string>
#include <unordered_map>
#include <vector>

#include <fcntl.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

// ──────────────────────────────────────────────────────────────────────────────
// Globals: dispatch table for the "next" layer (the runtime in our case).
// ──────────────────────────────────────────────────────────────────────────────
namespace {

constexpr const char* LAYER_NAME = "XR_APILAYER_INTRINSICS_SNIFF";
constexpr const char* SOCKET_PATH = "/tmp/xr_intrinsics.sock";

struct NextDispatch {
    PFN_xrGetInstanceProcAddr                   GetInstanceProcAddr = nullptr;
    PFN_xrEnumerateViewConfigurationViews       EnumerateViewConfigurationViews = nullptr;
    PFN_xrLocateViews                           LocateViews = nullptr;
    PFN_xrDestroyInstance                       DestroyInstance = nullptr;
};

// One dispatch per XrInstance.
std::mutex                                g_instances_mu;
std::unordered_map<XrInstance, NextDispatch> g_instances;

// Socket state
std::mutex        g_sock_mu;
int               g_server_fd = -1;  // listening socket (published side)
int               g_client_fd = -1;  // accepted client (consumer connected)
std::atomic<bool> g_client_connected{false};

// Per-eye cached resolution from xrEnumerateViewConfigurationViews.
struct EyeRes { uint32_t width = 0; uint32_t height = 0; };
std::mutex                                   g_res_mu;
std::vector<EyeRes>                          g_recommended_views;
std::atomic<int64_t>                         g_res_seq{0};

// ──────────────────────────────────────────────────────────────────────────────
// Socket publisher — one Unix domain datagram-style stream.
// Accepts a single client at a time (Python consumer). Newline-delimited JSON.
// ──────────────────────────────────────────────────────────────────────────────

void socket_init() {
    std::lock_guard<std::mutex> lk(g_sock_mu);
    if (g_server_fd >= 0) return;

    unlink(SOCKET_PATH);  // remove stale

    g_server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (g_server_fd < 0) {
        fprintf(stderr, "[xr-layer] socket() failed: %s\n", strerror(errno));
        return;
    }

    sockaddr_un addr{};
    addr.sun_family = AF_UNIX;
    std::strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    if (bind(g_server_fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
        fprintf(stderr, "[xr-layer] bind(%s) failed: %s\n", SOCKET_PATH, strerror(errno));
        close(g_server_fd);
        g_server_fd = -1;
        return;
    }
    if (listen(g_server_fd, 1) < 0) {
        fprintf(stderr, "[xr-layer] listen() failed: %s\n", strerror(errno));
        close(g_server_fd);
        g_server_fd = -1;
        return;
    }

    // Non-blocking so accept() in the hot path doesn't stall rendering
    int flags = fcntl(g_server_fd, F_GETFL, 0);
    fcntl(g_server_fd, F_SETFL, flags | O_NONBLOCK);

    fprintf(stderr, "[xr-layer] listening on %s\n", SOCKET_PATH);
}

void socket_try_accept() {
    std::lock_guard<std::mutex> lk(g_sock_mu);
    if (g_server_fd < 0 || g_client_fd >= 0) return;
    int fd = accept(g_server_fd, nullptr, nullptr);
    if (fd >= 0) {
        g_client_fd = fd;
        g_client_connected = true;
        fprintf(stderr, "[xr-layer] consumer connected\n");
    }
}

void socket_send_line(const std::string& line) {
    std::lock_guard<std::mutex> lk(g_sock_mu);
    if (g_client_fd < 0) return;
    ssize_t n = send(g_client_fd, line.data(), line.size(), MSG_NOSIGNAL);
    if (n < 0) {
        fprintf(stderr, "[xr-layer] consumer disconnected (send errno=%d)\n", errno);
        close(g_client_fd);
        g_client_fd = -1;
        g_client_connected = false;
    }
}

void socket_close_all() {
    std::lock_guard<std::mutex> lk(g_sock_mu);
    if (g_client_fd >= 0) { close(g_client_fd); g_client_fd = -1; }
    if (g_server_fd >= 0) { close(g_server_fd); g_server_fd = -1; }
    unlink(SOCKET_PATH);
}

// ──────────────────────────────────────────────────────────────────────────────
// Intercepted OpenXR calls.
// ──────────────────────────────────────────────────────────────────────────────

XrResult XRAPI_CALL layer_EnumerateViewConfigurationViews(
    XrInstance                  instance,
    XrSystemId                  systemId,
    XrViewConfigurationType     viewConfigurationType,
    uint32_t                    viewCapacityInput,
    uint32_t*                   viewCountOutput,
    XrViewConfigurationView*    views)
{
    NextDispatch nd;
    {
        std::lock_guard<std::mutex> lk(g_instances_mu);
        auto it = g_instances.find(instance);
        if (it == g_instances.end() || !it->second.EnumerateViewConfigurationViews)
            return XR_ERROR_FUNCTION_UNSUPPORTED;
        nd = it->second;
    }

    XrResult r = nd.EnumerateViewConfigurationViews(
        instance, systemId, viewConfigurationType,
        viewCapacityInput, viewCountOutput, views);

    // Capture when the runtime actually fills the array
    if (r == XR_SUCCESS && views && viewCapacityInput > 0 && viewCountOutput) {
        std::lock_guard<std::mutex> lk(g_res_mu);
        g_recommended_views.clear();
        g_recommended_views.reserve(*viewCountOutput);
        for (uint32_t i = 0; i < *viewCountOutput; ++i) {
            EyeRes e;
            e.width  = views[i].recommendedImageRectWidth;
            e.height = views[i].recommendedImageRectHeight;
            g_recommended_views.push_back(e);
        }
        g_res_seq.fetch_add(1);
        fprintf(stderr, "[xr-layer] recommended view sizes captured: %u views\n",
                *viewCountOutput);
        for (uint32_t i = 0; i < *viewCountOutput; ++i) {
            fprintf(stderr, "[xr-layer]   view[%u] %ux%u\n",
                    i, views[i].recommendedImageRectWidth,
                    views[i].recommendedImageRectHeight);
        }
    }
    return r;
}

XrResult XRAPI_CALL layer_LocateViews(
    XrSession                   session,
    const XrViewLocateInfo*     viewLocateInfo,
    XrViewState*                viewState,
    uint32_t                    viewCapacityInput,
    uint32_t*                   viewCountOutput,
    XrView*                     views)
{
    // Figure out which instance owns this session by asking the runtime.
    // Simplest path: walk our instance map (only one XR instance per process
    // in practice). Store the dispatch under the first (and only) instance.
    NextDispatch nd;
    {
        std::lock_guard<std::mutex> lk(g_instances_mu);
        if (g_instances.empty()) return XR_ERROR_HANDLE_INVALID;
        nd = g_instances.begin()->second;
    }
    if (!nd.LocateViews) return XR_ERROR_FUNCTION_UNSUPPORTED;

    XrResult r = nd.LocateViews(session, viewLocateInfo, viewState,
                                viewCapacityInput, viewCountOutput, views);

    if (r != XR_SUCCESS || !views || viewCapacityInput == 0 || !viewCountOutput)
        return r;
    uint32_t n = *viewCountOutput;
    if (n == 0) return r;

    // Lazy init socket + opportunistic accept (non-blocking; doesn't stall).
    socket_init();
    socket_try_accept();
    if (!g_client_connected.load()) return r;

    // Build a single JSON line for this frame.
    std::string out = "{\"seq\":";
    out += std::to_string(g_res_seq.load());
    out += ",\"time\":";
    out += std::to_string(viewLocateInfo ? (long long)viewLocateInfo->displayTime : 0LL);
    out += ",\"views\":[";
    for (uint32_t i = 0; i < n; ++i) {
        if (i) out += ",";
        const XrView& v = views[i];
        const XrFovf&   f = v.fov;
        const XrPosef&  p = v.pose;

        uint32_t w = 0, h = 0;
        {
            std::lock_guard<std::mutex> lk(g_res_mu);
            if (i < g_recommended_views.size()) {
                w = g_recommended_views[i].width;
                h = g_recommended_views[i].height;
            }
        }

        char buf[512];
        std::snprintf(buf, sizeof(buf),
            "{\"eye\":%u,\"w\":%u,\"h\":%u,"
            "\"angleLeft\":%.9f,\"angleRight\":%.9f,"
            "\"angleUp\":%.9f,\"angleDown\":%.9f,"
            "\"px\":%.9f,\"py\":%.9f,\"pz\":%.9f,"
            "\"qx\":%.9f,\"qy\":%.9f,\"qz\":%.9f,\"qw\":%.9f}",
            i, w, h,
            f.angleLeft, f.angleRight, f.angleUp, f.angleDown,
            p.position.x, p.position.y, p.position.z,
            p.orientation.x, p.orientation.y, p.orientation.z, p.orientation.w);
        out += buf;
    }
    out += "]}\n";
    socket_send_line(out);
    return r;
}

XrResult XRAPI_CALL layer_DestroyInstance(XrInstance instance)
{
    PFN_xrDestroyInstance next = nullptr;
    {
        std::lock_guard<std::mutex> lk(g_instances_mu);
        auto it = g_instances.find(instance);
        if (it != g_instances.end()) {
            next = it->second.DestroyInstance;
            g_instances.erase(it);
        }
    }
    XrResult r = next ? next(instance) : XR_SUCCESS;
    if (g_instances.empty()) socket_close_all();
    return r;
}

// ──────────────────────────────────────────────────────────────────────────────
// Dispatcher — returns our wrappers for intercepted calls, otherwise forwards.
// ──────────────────────────────────────────────────────────────────────────────

XrResult XRAPI_CALL layer_GetInstanceProcAddr(
    XrInstance instance, const char* name, PFN_xrVoidFunction* function)
{
    if (!function) return XR_ERROR_VALIDATION_FAILURE;

    // Our overrides
    if (std::strcmp(name, "xrEnumerateViewConfigurationViews") == 0) {
        *function = reinterpret_cast<PFN_xrVoidFunction>(&layer_EnumerateViewConfigurationViews);
        return XR_SUCCESS;
    }
    if (std::strcmp(name, "xrLocateViews") == 0) {
        *function = reinterpret_cast<PFN_xrVoidFunction>(&layer_LocateViews);
        return XR_SUCCESS;
    }
    if (std::strcmp(name, "xrDestroyInstance") == 0) {
        *function = reinterpret_cast<PFN_xrVoidFunction>(&layer_DestroyInstance);
        return XR_SUCCESS;
    }

    // Everything else: forward to runtime
    PFN_xrGetInstanceProcAddr nextGetProc = nullptr;
    {
        std::lock_guard<std::mutex> lk(g_instances_mu);
        auto it = g_instances.find(instance);
        if (it == g_instances.end()) return XR_ERROR_HANDLE_INVALID;
        nextGetProc = it->second.GetInstanceProcAddr;
    }
    if (!nextGetProc) return XR_ERROR_HANDLE_INVALID;
    return nextGetProc(instance, name, function);
}

// ──────────────────────────────────────────────────────────────────────────────
// xrCreateApiLayerInstance: called by the loader when the app creates an
// XrInstance. We forward to the next link and capture its proc-addr table.
// ──────────────────────────────────────────────────────────────────────────────

XrResult XRAPI_CALL layer_CreateApiLayerInstance(
    const XrInstanceCreateInfo*        info,
    const XrApiLayerCreateInfo*        apiLayerInfo,
    XrInstance*                        instance)
{
    if (!apiLayerInfo || !apiLayerInfo->nextInfo) return XR_ERROR_INITIALIZATION_FAILED;

    // Forward CreateInstance through the rest of the chain → runtime.
    PFN_xrGetInstanceProcAddr nextGetProc = apiLayerInfo->nextInfo->nextGetInstanceProcAddr;
    PFN_xrCreateApiLayerInstance nextCreate = apiLayerInfo->nextInfo->nextCreateApiLayerInstance;
    if (!nextGetProc || !nextCreate) return XR_ERROR_INITIALIZATION_FAILED;

    // Advance the chain pointer for the next link.
    XrApiLayerCreateInfo nextLayerInfo = *apiLayerInfo;
    nextLayerInfo.nextInfo = apiLayerInfo->nextInfo->next;

    XrResult r = nextCreate(info, &nextLayerInfo, instance);
    if (r != XR_SUCCESS) return r;

    // Cache the next-link dispatch for this XrInstance.
    NextDispatch nd;
    nd.GetInstanceProcAddr = nextGetProc;

#define LOAD_NEXT(NAME)                                                              \
    do {                                                                             \
        PFN_xrVoidFunction fn = nullptr;                                             \
        if (nextGetProc(*instance, "xr" #NAME, &fn) == XR_SUCCESS) {                 \
            nd.NAME = reinterpret_cast<PFN_xr##NAME>(fn);                            \
        }                                                                            \
    } while (0)

    LOAD_NEXT(EnumerateViewConfigurationViews);
    LOAD_NEXT(LocateViews);
    LOAD_NEXT(DestroyInstance);

#undef LOAD_NEXT

    {
        std::lock_guard<std::mutex> lk(g_instances_mu);
        g_instances[*instance] = nd;
    }
    fprintf(stderr, "[xr-layer] XrInstance created; interceptors armed\n");

    // Init socket early so consumer can connect even before first xrLocateViews.
    socket_init();

    return XR_SUCCESS;
}

} // anon namespace

// ──────────────────────────────────────────────────────────────────────────────
// Loader negotiation entry point — exported with default visibility.
// ──────────────────────────────────────────────────────────────────────────────

extern "C" __attribute__((visibility("default")))
XrResult XRAPI_CALL xrNegotiateLoaderApiLayerInterface(
    const XrNegotiateLoaderInfo* loaderInfo,
    const char*                  layerName,
    XrNegotiateApiLayerRequest*  apiLayerRequest)
{
    if (!loaderInfo || !apiLayerRequest || !layerName) return XR_ERROR_INITIALIZATION_FAILED;
    if (loaderInfo->structType != XR_LOADER_INTERFACE_STRUCT_LOADER_INFO) return XR_ERROR_INITIALIZATION_FAILED;
    if (apiLayerRequest->structType != XR_LOADER_INTERFACE_STRUCT_API_LAYER_REQUEST) return XR_ERROR_INITIALIZATION_FAILED;
    if (std::strcmp(layerName, LAYER_NAME) != 0) return XR_ERROR_INITIALIZATION_FAILED;

    apiLayerRequest->layerInterfaceVersion = XR_CURRENT_LOADER_API_LAYER_VERSION;
    apiLayerRequest->layerApiVersion       = XR_CURRENT_API_VERSION;
    apiLayerRequest->getInstanceProcAddr       = &layer_GetInstanceProcAddr;
    apiLayerRequest->createApiLayerInstance    = &layer_CreateApiLayerInstance;
    return XR_SUCCESS;
}
```

#### 9.1.2 `xr_intrinsics_layer.json` — manifest the OpenXR loader reads

```json
{
    "file_format_version": "1.0.0",
    "api_layer": {
        "name": "XR_APILAYER_INTRINSICS_SNIFF",
        "library_path": "./libxr_intrinsics_layer.so",
        "api_version": "1.0",
        "implementation_version": "1",
        "description": "Sniffs per-eye XrFovf + recommended resolution; publishes to /tmp/xr_intrinsics.sock"
    }
}
```

#### 9.1.3 `CMakeLists.txt` — build configuration

```cmake
cmake_minimum_required(VERSION 3.20)
project(xr_intrinsics_layer CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)   # only xrNegotiateLoaderApiLayerInterface is exported
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

# OpenXR headers are fetched by the build.sh helper into ./openxr_headers/
# If you place them elsewhere, point -DOPENXR_INCLUDE_DIR=/path at config time.
if(NOT DEFINED OPENXR_INCLUDE_DIR)
    set(OPENXR_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/openxr_headers/include"
        CACHE PATH "Path containing openxr/openxr.h and openxr/openxr_loader_negotiation.h")
endif()

if(NOT EXISTS "${OPENXR_INCLUDE_DIR}/openxr/openxr.h")
    message(FATAL_ERROR
        "openxr.h not found at ${OPENXR_INCLUDE_DIR}/openxr/openxr.h\n"
        "Run build.sh (which fetches Khronos headers), or pass -DOPENXR_INCLUDE_DIR=<path>.")
endif()
if(NOT EXISTS "${OPENXR_INCLUDE_DIR}/openxr/openxr_loader_negotiation.h")
    message(FATAL_ERROR
        "openxr_loader_negotiation.h not found at ${OPENXR_INCLUDE_DIR}/openxr/openxr_loader_negotiation.h")
endif()

add_library(xr_intrinsics_layer SHARED xr_intrinsics_layer.cpp)
target_include_directories(xr_intrinsics_layer PRIVATE ${OPENXR_INCLUDE_DIR})
target_compile_options(xr_intrinsics_layer PRIVATE -Wall -Wextra -Wno-unused-parameter -O2)
set_target_properties(xr_intrinsics_layer PROPERTIES OUTPUT_NAME "xr_intrinsics_layer")

# After build, copy the manifest next to the .so so XR_API_LAYER_PATH can point
# at the build directory directly.
add_custom_command(TARGET xr_intrinsics_layer POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy
        "${CMAKE_CURRENT_SOURCE_DIR}/xr_intrinsics_layer.json"
        "$<TARGET_FILE_DIR:xr_intrinsics_layer>/xr_intrinsics_layer.json")
```

#### 9.1.4 `build.sh` — fetches Khronos headers and builds

```bash
#!/usr/bin/env bash
# Fetch Khronos OpenXR headers and build the intrinsics-sniffing API layer.
#
# Usage:
#   bash build.sh                    # default: fetch headers, configure, build
#   bash build.sh clean              # wipe build/ and openxr_headers/
#
# After success:
#   build/libxr_intrinsics_layer.so
#   build/xr_intrinsics_layer.json
# Register (from this directory):
#   export XR_API_LAYER_PATH="$PWD/build"
#   export XR_ENABLE_API_LAYERS=XR_APILAYER_INTRINSICS_SNIFF

set -e

HERE="$(cd "$(dirname "$0")" && pwd)"
cd "$HERE"

if [ "$1" = "clean" ]; then
    rm -rf build openxr_headers
    echo "cleaned."
    exit 0
fi

# ── 1. Fetch headers if missing ──────────────────────────────────────────────
OPENXR_TAG="release-1.1.41"   # stable release; matches loader API version 1.0+
HDR_DIR="$HERE/openxr_headers"

if [ ! -f "$HDR_DIR/include/openxr/openxr.h" ] || \
   [ ! -f "$HDR_DIR/include/openxr/openxr_loader_negotiation.h" ]; then
    echo "[build] Fetching Khronos OpenXR-SDK ($OPENXR_TAG) headers..."
    rm -rf "$HDR_DIR" "$HDR_DIR.tmp"
    mkdir -p "$HDR_DIR.tmp"
    git clone --depth 1 --branch "$OPENXR_TAG" \
        https://github.com/KhronosGroup/OpenXR-SDK.git "$HDR_DIR.tmp/OpenXR-SDK"
    mkdir -p "$HDR_DIR/include/openxr"
    cp "$HDR_DIR.tmp/OpenXR-SDK/include/openxr/openxr.h"                    "$HDR_DIR/include/openxr/"
    cp "$HDR_DIR.tmp/OpenXR-SDK/include/openxr/openxr_platform.h"           "$HDR_DIR/include/openxr/"
    cp "$HDR_DIR.tmp/OpenXR-SDK/include/openxr/openxr_platform_defines.h"   "$HDR_DIR/include/openxr/"
    cp "$HDR_DIR.tmp/OpenXR-SDK/include/openxr/openxr_reflection.h"         "$HDR_DIR/include/openxr/"
    if [ -f "$HDR_DIR.tmp/OpenXR-SDK/include/openxr/openxr_loader_negotiation.h" ]; then
        cp "$HDR_DIR.tmp/OpenXR-SDK/include/openxr/openxr_loader_negotiation.h" "$HDR_DIR/include/openxr/"
    elif [ -f "$HDR_DIR.tmp/OpenXR-SDK/src/common/loader_interfaces.h" ]; then
        cp "$HDR_DIR.tmp/OpenXR-SDK/src/common/loader_interfaces.h" \
           "$HDR_DIR/include/openxr/openxr_loader_negotiation.h"
    else
        echo "[build] ERROR: loader negotiation header not found in checkout"
        ls -la "$HDR_DIR.tmp/OpenXR-SDK/include/openxr/"
        exit 1
    fi
    rm -rf "$HDR_DIR.tmp"
    echo "[build] Headers ready at $HDR_DIR/include/openxr/"
else
    echo "[build] Using existing headers at $HDR_DIR/include/openxr/"
fi

# ── 2. Configure + build ─────────────────────────────────────────────────────
cmake -B build -S . \
    -DCMAKE_BUILD_TYPE=Release \
    -DOPENXR_INCLUDE_DIR="$HDR_DIR/include"
cmake --build build -j"$(nproc)"

echo ""
echo "================================================================"
echo "  Built: $HERE/build/libxr_intrinsics_layer.so"
echo "  Manifest: $HERE/build/xr_intrinsics_layer.json"
echo "================================================================"
```

After saving, mark it executable: `chmod +x build.sh`.

#### 9.1.5 `xr_intrinsics_consumer.py` — standalone consumer (optional)

Useful as a standalone debugging tool. The cleaned-up `test_cloudxr.py`
contains an embedded consumer (`IntrinsicsOneShot`) so this file is
optional — keep it around for ad-hoc inspection or as a reference for
downstream scripts that need to read the socket.

```python
#!/usr/bin/env python3
"""
xr_intrinsics_consumer.py

Reads per-frame XrView data published by xr_intrinsics_layer.so
on /tmp/xr_intrinsics.sock, converts XrFovf (angleLeft/Right/Up/Down, radians)
into OpenCV-style intrinsics (fx, fy, cx, cy), and prints per-eye K matrices
once per second. Also emits them as JSON lines on stdout for downstream use.

Conversion reference:
    fx =  W / ( tan(angleRight) - tan(angleLeft) )
    fy =  H / ( tan(angleUp)    - tan(angleDown) )
    cx = -fx * tan(angleLeft)
    cy =  fy * tan(angleUp)        # OpenCV y grows downward
"""

from __future__ import annotations

import json
import math
import os
import socket
import sys
import time

SOCKET_PATH = "/tmp/xr_intrinsics.sock"


def fov_to_K(w: int, h: int, aL: float, aR: float, aU: float, aD: float):
    tanL = math.tan(aL); tanR = math.tan(aR)
    tanU = math.tan(aU); tanD = math.tan(aD)
    fx = w / (tanR - tanL)
    fy = h / (tanU - tanD)
    cx = -fx * tanL
    cy = fy * tanU
    return fx, fy, cx, cy


def open_socket_retry(path: str, poll_s: float = 0.5) -> socket.socket:
    print(f"[consumer] waiting for {path} (layer must be loaded by Isaac Sim)...",
          file=sys.stderr, flush=True)
    s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    while True:
        try:
            s.connect(path)
            print(f"[consumer] connected to {path}", file=sys.stderr, flush=True)
            return s
        except (FileNotFoundError, ConnectionRefusedError):
            time.sleep(poll_s)


def main():
    sock = open_socket_retry(SOCKET_PATH)
    f = sock.makefile("r", buffering=1)

    last_print = 0.0
    last_K = {}

    try:
        for line in f:
            line = line.strip()
            if not line: continue
            try:
                msg = json.loads(line)
            except json.JSONDecodeError:
                continue

            for v in msg.get("views", []):
                eye = v["eye"]
                w   = int(v["w"]);  h = int(v["h"])
                aL  = float(v["angleLeft"]);  aR = float(v["angleRight"])
                aU  = float(v["angleUp"]);    aD = float(v["angleDown"])
                if w <= 0 or h <= 0: continue
                fx, fy, cx, cy = fov_to_K(w, h, aL, aR, aU, aD)
                last_K[eye] = {
                    "w": w, "h": h, "fx": fx, "fy": fy, "cx": cx, "cy": cy,
                    "angleLeft": aL, "angleRight": aR,
                    "angleUp":   aU, "angleDown":  aD,
                    "pose": {
                        "p": [v["px"], v["py"], v["pz"]],
                        "q": [v["qx"], v["qy"], v["qz"], v["qw"]],
                    },
                }

            now = time.monotonic()
            if now - last_print > 1.0 and last_K:
                last_print = now
                print("─" * 74)
                for eye_id in sorted(last_K):
                    k = last_K[eye_id]
                    label = {0: "L", 1: "R"}.get(eye_id, f"V{eye_id}")
                    print(
                        f"[eye {label}] {k['w']}×{k['h']}  "
                        f"fx={k['fx']:8.2f} fy={k['fy']:8.2f}  "
                        f"cx={k['cx']:8.2f} cy={k['cy']:8.2f}  "
                        f"FoV_h={math.degrees(k['angleRight']-k['angleLeft']):.2f}° "
                        f"FoV_v={math.degrees(k['angleUp']-k['angleDown']):.2f}°"
                    )
                print(json.dumps({"K": last_K}), flush=True)
    except KeyboardInterrupt:
        print("\n[consumer] interrupted", file=sys.stderr)
    finally:
        sock.close()


if __name__ == "__main__":
    main()
```

After all five files exist in `~/xr_intrinsics_layer/`, you should have:

```
~/xr_intrinsics_layer/
├── xr_intrinsics_layer.cpp       (~390 lines)
├── xr_intrinsics_layer.json      (10 lines)
├── CMakeLists.txt                (37 lines)
├── build.sh                      (~80 lines, +x permission)
└── xr_intrinsics_consumer.py     (~130 lines)
```

### 9.2 Build

```bash
cd ~/xr_intrinsics_layer
bash build.sh
```

On first run this clones Khronos OpenXR-SDK (`release-1.1.41`) into
`openxr_headers/` to get the public `openxr.h` and
`openxr_loader_negotiation.h`. Subsequent builds skip the fetch.

Requires: `g++ ≥ 11`, `cmake ≥ 3.20`, `git`, network access to github.com.

Expected output after success:

```
[100%] Built target xr_intrinsics_layer
================================================================
  Built:    ~/xr_intrinsics_layer/build/libxr_intrinsics_layer.so
  Manifest: ~/xr_intrinsics_layer/build/xr_intrinsics_layer.json
================================================================
```

### 9.3 Register with Isaac Sim

Two env vars must be set in the terminal that launches Isaac Sim:

```bash
export XR_API_LAYER_PATH="$HOME/xr_intrinsics_layer/build"
export XR_ENABLE_API_LAYERS=XR_APILAYER_INTRINSICS_SNIFF
```

These are read by `libopenxr_loader.so.1` at `xrCreateInstance` time. They
must be set **before** Isaac Sim starts — setting them after the process
launches has no effect.

### 9.4 Run — full end-to-end launch sequence

Five things must be running, in this order. Each gets its own terminal.

**Terminal 1 — CloudXR service**

```bash
cd ~/cloudxr-runtime
./cxr-service
```

Wait until you see:

```
[cxr-service] Service started. Ready for OpenXR apps. Press Ctrl+C to stop.
```

Leave running.

**Terminal 2 — wss-proxy (Docker)**

```bash
# First time only — build the image (see §4.1)
# cd ~/cloudxr-js/cloudxr-js-samples/proxy && docker build -t cloudxr-wss-proxy .

# If Docker gives "permission denied while trying to connect":
#   newgrp docker
# (or log out/in; permanent fix was sudo usermod -aG docker $USER in §1.2)

docker start wss-proxy 2>/dev/null || docker run -d --name wss-proxy \
    --network host \
    -v cloudxr-proxy-certs:/usr/local/etc/haproxy/certs \
    -e BACKEND_HOST=localhost \
    -e BACKEND_PORT=49100 \
    -e PROXY_PORT=48322 \
    cloudxr-wss-proxy

docker logs --tail 10 wss-proxy
# Expected: "✅ Using existing SSL certificate..." (or "🔐 Generating..." first time)
```

Leave the container running. `docker ps` should show `wss-proxy` in the list.

**Terminal 3 — CloudXR.js HTTPS dev server**

```bash
cd ~/cloudxr-js/cloudxr-js-samples/simple
npm run dev-server:https
# Expected: "Local: https://0.0.0.0:8080/"
```

Leave running.

**Terminal 4 — Isaac Sim with the intrinsics layer registered**

The CloudXR + intrinsics-layer env vars must be set **in this terminal
before `python3` runs**. The OpenXR loader reads them at `xrCreateInstance`
time, which happens early in Kit startup. Setting them after the process
launches has no effect.

```bash
# 1. Activate CloudXR env in this terminal (defined as alias in §2.4)
use-cloudxr

# 2. Sanity check — both env vars should be populated now
echo "XR_RUNTIME_JSON=$XR_RUNTIME_JSON"
echo "LD_LIBRARY_PATH contains cloudxr? $(echo $LD_LIBRARY_PATH | grep -o cloudxr)"

# 3. Intrinsics layer — set for this terminal
export XR_API_LAYER_PATH="$HOME/xr_intrinsics_layer/build"
export XR_ENABLE_API_LAYERS=XR_APILAYER_INTRINSICS_SNIFF

# 4. Launch Isaac Sim
python3 ~/test_cloudxr.py --xr --device cuda:0
# Or with the OR environment:
# python3 ~/test_cloudxr.py --xr --device cuda:0 \
#     --or-environment "/media/jack/Seagate Basic/i4h-assets/132c82d/Props/shared_OR_without_Mark/main.usd"
```

Wait for `Simulation App Startup Complete` and the UI window to appear.

**In the Isaac Sim UI**

1. Find the **AR** panel (right side, or Window menu)
2. Output Plugin: **OpenXR**
3. OpenXR Runtime: **System OpenXR Runtime**
4. Click **Start AR**

Within 1–2 seconds Terminal 1 prints:

```
[event] OpenXR app CONNECTED (e.g. Isaac Sim)
```

And Terminal 4 prints the layer-loaded banner:

```
[xr-layer] listening on /tmp/xr_intrinsics.sock
[xr-layer] XrInstance created; interceptors armed
```

**On the Quest**

1. Meta Browser → `https://<LINUX_IP>:8080/` — accept self-signed cert
2. **First time only** (per Quest + Linux IP combo): in a new tab, visit
   `https://<LINUX_IP>:48322/` and accept that cert too. The wss-proxy uses
   a separate self-signed cert, and the WebSocket to it from the `:8080`
   page will silently fail until the browser trusts it. Once accepted, the
   trust persists — skip this step on subsequent sessions unless you reset
   the Quest, regenerate the wss-proxy cert, or change `<LINUX_IP>`.
3. Back on `:8080` — fill in:
   - Server IP: `<LINUX_IP>`
   - Port: `48010`
   - Proxy URL: `https://<LINUX_IP>:48322`
4. Click **CONNECT**

Terminal 1 then prints:

```
[event] CloudXR client CONNECTED
```

Terminal 4 prints:

```
[xr-layer] consumer connected
[xr-layer] recommended view sizes captured: 2 views
[xr-layer]   view[0] 2048x1792
[xr-layer]   view[1] 2048x1792
```

And once per second `test_cloudxr.py` emits:

```
──────────────────────────────────────────────────────────────────────
[eye L] 2048×1792  fx= 845.32 fy= 681.97  cx=1286.87 cy= 711.15
[eye R] 2048×1792  fx= 845.32 fy= 681.97  cx= 761.13 cy= 711.15
[head]   pos=(...) q=(...)
[anchor] pos=(+0.000, +0.000, +0.000) q=(+0.707, +0.707, +0.000, +0.000)
```

Those are the real per-frame Quest display intrinsics. The two eyes' `cx`
values are asymmetric (mirror images about image center, `cx_L + cx_R = W`)
because real HMD optics use off-axis frustums — this is correct, not a bug.

**Alternative consumer**

If you want intrinsics in a terminal without running Isaac Sim's full test
script (e.g. just observing the layer), start this in a separate terminal
(before or after Isaac Sim):

```bash
python3 ~/xr_intrinsics_layer/xr_intrinsics_consumer.py
```

It blocks waiting for `/tmp/xr_intrinsics.sock` to exist, then streams K
matrices once per second plus raw JSON for programmatic consumers.

### 9.5 XrFovf → K conversion

For the canonical OpenXR projection with signed FoV angles (radians):

```
fx =  W / ( tan(angleRight) - tan(angleLeft) )
fy =  H / ( tan(angleUp)    - tan(angleDown) )
cx = -fx * tan(angleLeft)
cy =  fy * tan(angleUp)           # OpenCV convention: y grows downward
```

Same math in Python lives in `IntrinsicsConsumer._fov_to_K` in
`test_cloudxr.py` and in `xr_intrinsics_consumer.py::fov_to_K`.

### 9.6 Troubleshooting the layer

**The layer doesn't load / no `[xr-layer]` lines printed**

```bash
export XR_LOADER_DEBUG=all
python3 ~/test_cloudxr.py --xr --device cuda:0 2>&1 | \
    grep -iE "apilayer|intrinsics|xr_enable|xr_api_layer"
```

Common causes:

- `XR_API_LAYER_PATH` points at the wrong directory (must contain both the
  `.so` and the `.json`)
- `XR_ENABLE_API_LAYERS` mis-spelled — must be exactly
  `XR_APILAYER_INTRINSICS_SNIFF` to match the manifest
- Env vars set *after* Isaac Sim was already started; restart Isaac Sim
  from scratch in a terminal where both vars are set

**Layer loads but consumer doesn't connect**
→ Socket path permission. `ls -la /tmp/xr_intrinsics.sock` should exist and
   be writable. If owned by root (shouldn't happen), remove and relaunch
   Isaac Sim.

**Layer loads, consumer connects, but no data arrives**
→ The XR session never entered the frame loop. Confirm Start AR was clicked
   and the viewport split into two eyes. If the head pose in
   `test_cloudxr.py` output is changing, `xrLocateViews` is firing and
   intrinsics should arrive within 1 second.

**Build fails: `loader negotiation header not found`**
→ The header moved between OpenXR-SDK tags. Edit `build.sh` and change
   `OPENXR_TAG` to `release-1.1.36` or a nearby version, `bash build.sh clean`,
   then `bash build.sh` again.

---

## Part 10 — Wireless ADB (deploy Android Studio APKs without a cable)

Used when you need to push a companion app to the Quest — for example the
Kotlin passthrough-intrinsics extractor used to sanity-check the display
intrinsics from Part 9. After initial setup, Android Studio's **Run** button
deploys over Wi-Fi.

**Prereqs:** Developer Mode enabled on the Quest; `adb` available (shipped
with Android Studio at `~/Android/Sdk/platform-tools/adb`); Quest and Linux
on the same Wi-Fi network (both on the subnet you identified in Part 0).

### 10.1 Put adb on your PATH (one-time)

```bash
echo 'export PATH=$PATH:$HOME/Android/Sdk/platform-tools' >> ~/.bashrc
source ~/.bashrc
adb version     # sanity: prints "Android Debug Bridge version ..."
```

### 10.2 Enable wireless debugging via a USB-C cable (one-time per Quest boot)

1. Plug the Quest into the Linux box with USB-C
2. Put the headset on — an **Allow USB debugging?** dialog appears. Check
   "Always allow" and tap OK
3. Back on Linux:

   ```bash
   adb devices
   # Expected:
   # List of devices attached
   # <serial>   device
   ```

   If it says `unauthorized`, the dialog wasn't accepted; check the headset.

4. Switch the Quest's adb daemon to listen on TCP port 5555:

   ```bash
   adb tcpip 5555
   # Expected: "restarting in TCP mode port: 5555"
   ```

5. **Unplug the USB cable.** This matters — leaving it plugged in keeps adb
   in USB mode and the wireless connect below will fail.

### 10.3 Find the Quest's IP and connect

While the cable is still plugged in (before step 10.2.5), grab the IP:

```bash
adb shell ip route | awk '{print $9}'
# e.g. 192.168.1.103   →  this is <QUEST_IP>
```

After unplugging, connect wirelessly:

```bash
adb connect <QUEST_IP>:5555
# Expected: "connected to <QUEST_IP>:5555"

adb devices
# Expected:
# List of devices attached
# <QUEST_IP>:5555   device
```

### 10.4 Deploy from Android Studio

Open the project. The Quest now appears as `<QUEST_IP>:5555` in the device
dropdown at the top of the IDE. **Run** builds, installs, and launches the
APK over Wi-Fi — no cable involved.

### 10.5 Quick-reconnect aliases

Wireless adb drops on Quest sleep or IP change. Add these so reconnecting is
a single word:

```bash
cat >> ~/.bashrc <<'EOF'

# Wireless ADB to Quest (update IP if your Quest's changes)
export QUEST_IP=192.168.1.103
alias quest-connect='adb connect $QUEST_IP:5555'
alias quest-disconnect='adb disconnect'
alias quest-devices='adb devices'
EOF
source ~/.bashrc

# Day-to-day reconnect:
quest-connect
```

### 10.6 When to cable again

- **Quest reboots** — `adb tcpip 5555` doesn't survive a reboot. Plug in
  cable, `adb tcpip 5555`, unplug, `quest-connect`.
- **Quest's IP changed** (DHCP reshuffle) — plug in, rerun the IP lookup
  from §10.3, update `QUEST_IP` in `~/.bashrc`.
- **`adb devices` shows `unauthorized`** — authorization was revoked
  (happens after factory reset or toggling Developer Mode). Plug in cable,
  accept the debugging dialog again.

### 10.7 Troubleshooting

| Problem                                          | Fix                                                                                                                                   |
|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `adb: command not found`                         | §10.1 not done. `export PATH=$PATH:$HOME/Android/Sdk/platform-tools`                                                                  |
| `failed to connect: No route to host`            | Router AP isolation (separates Wi-Fi clients). Test: `ping <QUEST_IP>` from Linux. If that fails too, disable AP isolation on router. |
| `failed to connect: Connection refused`          | `adb tcpip 5555` wasn't run after the last Quest reboot. Redo §10.2.                                                                  |
| `failed to connect: Connection timed out`        | Quest asleep. Put it on, wait 5s, retry.                                                                                              |
| Pair succeeds but Android Studio doesn't see it  | Restart Android Studio — it caches the device list at startup.                                                                        |
| `unauthorized` state after `connect`             | Debug authorization expired. Plug in USB, accept the dialog, `adb tcpip 5555`, unplug, reconnect.                                     |

