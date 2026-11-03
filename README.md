# Product Requirements Document (PRD) & Technical Architecture Blueprint
## Hybrid-Architecture Homelab Cloud & Distributed Compute Cluster

- **Document Version:** 1.0.0-PROD
- **Target Consumer:** Autonomous Infrastructure & Coding Agent (`Claude Code`)
- **Primary Control Plane:** Raspberry Pi 5 (8GB RAM, `arm64`, 2TB NVMe SSD)
- **Compute Worker Pool:** Heterogeneous Repurposed Laptops/Desktops (`x86_64` / `arm64`, Discrete/Integrated GPUs)
- **Overlay Network Fabric:** Tailscale WireGuard Mesh (`tailscale0`, MagicDNS `*.homelab.ts.net`)
- **Container Orchestrator:** K3s Lightweight Kubernetes (Single-Server SQLite/etcd on NVMe + Multi-Arch Agents)

## Summary

**homelab-cluster is a private cloud that runs at home.** It uses one Raspberry Pi 5 and old laptops and desktops. Together, these machines do work that you usually pay cloud services to do. Your files, your code and your AI stay on hardware that you own.

### What it does

| You want to… | Usual paid service | In this system |
| :--- | :--- | :--- |
| Store and share files | Dropbox, Google Drive | Nextcloud, Filebrowser |
| Save and search photos | Google Photos, iCloud Photos | Immich |
| Store code and run automatic tests | GitHub, GitHub Actions | Forgejo, act_runner |
| Use AI models for chat and code help | ChatGPT, OpenAI API | Private AI gateway with open models |

- **Private access from anywhere.** All machines connect through Tailscale, a private and encrypted network. Only your approved devices can connect.
- **Works with your AI tools.** The AI gateway uses the same format as the OpenAI API. To connect most AI apps and code tools, change the server address in their settings.

### How it works

- **The Raspberry Pi 5 is the main computer.** It is always on. It keeps all data on a 2 TB SSD and sends work to the other machines.
- **Old laptops and desktops are the workers.** They do the heavy work: AI answers, photo search and code builds. If a worker stops, no data is lost.
- **Workers wake when work arrives.** If the AI machines are asleep, the system wakes them. The first answer can then take up to one minute.

### What comes next

The first version gives you storage, code tools and AI. Later, the system will become one home for all of my projects:

- **Run all my projects.** Websites, apps, APIs and databases run on the cluster, not on paid hosts.
- **Cloud tests, sized to the hardware.** Each code change can start automatic tests and a preview copy of the project. The size of these tests depends on the power of the cluster at that time.
- **Larger AI models.** Several machines can share one large model. This makes bigger models possible, but they are slower than paid AI services.
- **More hardware over time.** Any spare computer can join as a worker. Each new machine adds power and memory.
- **Public access when necessary.** At first, only approved devices can connect. Later, you can make selected projects public.

### Limits

- The system is only as powerful as the hardware in it. It is not as fast as a large paid cloud.
- The memory of the workers limits the size of the AI models.
- A power failure at home stops the system. An internet failure stops access from outside the home.

### Where to read next

- **Business owners:** This summary is all you need.
- **Developers:** Read [Section 6](#6-core-workload-modules-phased-rollout) for the services, [Section 1](#1-repository--directory-structure-for-claude-code) for the file layout and [Section 7](#7-implementation-roadmap--task-breakdown-for-claude-code) for the build steps.

## 1. Repository & Directory Structure for Claude Code

Claude Code MUST initialize and populate the following exact monorepo file tree at the repository root (`homelab-cluster/`). Every file path listed below is mandatory and must contain deterministic, production-ready code with zero placeholder stubs.

```text
homelab-cluster/
├── .env.example                                  # Template environment variables (Tailscale keys, UUIDs, MACs, IPs)
├── Makefile                                      # Unified CLI entrypoint for bootstrapping, linting, and verification
├── README.md                                     # Operator runbook and architecture overview
├── ansible/
│   ├── ansible.cfg                               # Ansible configuration (pipelining, ssh args, inventory path)
│   ├── inventory/
│   │   ├── hosts.yml                             # Declarative inventory: control_plane, workers_x86, workers_arm64, gpu_nodes
│   │   └── group_vars/
│   │       ├── all.yml                           # Global variables: tailnet_domain, k3s_version, nfs_cidr
│   │       ├── control_plane.yml                 # Pi 5 specific vars: nvme_uuid, k3s_data_dir, ingress_ip
│   │       └── workers.yml                       # Laptop worker vars: battery_start_thresh, battery_stop_thresh, wol_interface
│   ├── playbooks/
│   │   ├── 01-pi-base-os.yml                     # Pi 5 cgroups, NVMe fstab mount, swap, log2ram, sysctl tuning
│   │   ├── 02-storage-layer.yml                  # NFSv4 server, Samba shares, and MinIO S3 systemd service on Pi 5
│   │   ├── 03-laptop-clamshell-prep.yml          # Lid switch ignore, sleep masking, TLP battery cap, headless GPU/Xorg
│   │   ├── 04-tailscale-mesh.yml                 # Tailscale installation, authkey join, ACL tag advertisement, MTU tuning
│   │   ├── 05-k3s-control-plane.yml              # K3s server bootstrap bound to tailscale0 and /mnt/nvme/k3s
│   │   ├── 06-k3s-workers.yml                    # K3s agent join, node labels, taints, NVIDIA container toolkit
│   │   └── 99-cluster-verify.yml                 # End-to-end automated health assertions across all nodes
│   └── roles/
│       ├── pi_nvme/
│       ├── nfs_samba_minio/
│       ├── laptop_blade/
│       ├── gpu_headless/
│       ├── tailscale_node/
│       ├── k3s_server/
│       └── k3s_agent/
├── scripts/
│   ├── hardware-prep/
│   │   ├── pi5-bootstrap.sh                      # Standalone bash script for Day-1 Raspberry Pi 5 OS & NVMe setup
│   │   ├── laptop-clamshell-prep.sh              # Idempotent bash script to convert Ubuntu/Debian laptop into server blade
│   │   ├── gpu-headless-init.sh                  # Configures dummy Xorg / DRM KMS / nvidia-persistenced for clamshell GPUs
│   │   └── verify-battery-cap.sh                 # Reads /sys/class/power_supply/BAT*/charge_control_* and asserts 50-80% cap
│   └── network/
│       ├── tailscale-join.sh                     # Wrapper around tailscale up with retry logic and DERP latency check
│       └── wol-dispatch.sh                       # Sends magic packets (etherwake/wakeonlan) over LAN/subnet router
├── tailscale/
│   ├── policy.hujson                             # Complete Tailscale ACL policy, tagOwners, autoApprovers, and nodeAttrs
│   └── serve-config.json                         # Tailscale Serve / HTTPS cert automation config for *.homelab.ts.net
├── k3s/
│   ├── bootstrap/
│   │   ├── k3s-server-config.yaml                # /etc/rancher/k3s/config.yaml for Raspberry Pi 5 control plane
│   │   ├── k3s-agent-config.yaml.j2              # Jinja2 template for /etc/rancher/k3s/config.yaml on laptop workers
│   │   └── registries.yaml                       # Local container registry mirror config
│   └── manifests/
│       ├── 00-namespaces.yaml                    # Namespaces: storage, cicd, inference, ingress-system, monitoring
│       ├── 01-storage-classes.yaml               # nfs-client StorageClass (RWX) and local-nvme StorageClass (RWO on Pi)
│       ├── 02-priority-classes.yaml              # PriorityClasses: core-control-plane (100000), batch-ephemeral (1000)
│       ├── 03-ingress-traefik.yaml               # Traefik HelmChartConfig & TLSStore bound to Pi Tailscale IP
│       ├── 04-nvidia-device-plugin.yaml          # NVIDIA k8s-device-plugin DaemonSet pinned to nvidia.com/gpu=true nodes
│       └── 05-coredns-custom.yaml                # CoreDNS ConfigMap rewrite rules for *.homelab.ts.net hair-pinning
├── services/
│   ├── storage/
│   │   ├── filebrowser/
│   │   │   └── deployment.yaml                   # Lightweight file portal on Pi 5 backed by /mnt/nvme/cloud
│   │   ├── nextcloud/
│   │   │   └── helm-values.yaml                  # Nextcloud configuration bound to Pi NVMe and PostgreSQL
│   │   └── immich/
│   │       ├── immich-core-pi.yaml               # Immich Server, Web, Postgres (pgvector), Redis pinned to arm64 Pi 5
│   │       └── immich-ml-worker.yaml             # Immich Machine Learning Deployment pinned to x86_64/GPU worker nodes
│   ├── cicd/
│   │   ├── forgejo/
│   │   │   └── forgejo-pi.yaml                   # Forgejo Git server + SSH ingress hosted on Pi 5 (/mnt/nvme/git)
│   │   └── act-runner/
│   │       ├── runner-secret.yaml.example        # Registration token secret template
│   │       └── act-runner-workers.yaml           # Multi-arch act_runner Deployment targeting ephemeral laptop workers
│   └── inference/
│       ├── gateway/
│       │   ├── Dockerfile                        # FastAPI Wake-on-LAN & Redis Queue proxy + LiteLLM router image
│       │   ├── main.py                           # Async FastAPI middleware: intercepts /v1/*, checks worker readiness, sends WoL
│       │   ├── requirements.txt                  # fastapi, uvicorn, httpx, redis, wakeonlan, kubernetes, litellm
│       │   ├── litellm-config.yaml               # Model routing table mapping OpenAI model names to worker Ollama/vLLM/Exo endpoints
│       │   └── deployment-pi.yaml                # Gateway + Redis Deployment pinned to Pi 5 control plane
│       ├── workers/
│       │   ├── ollama-daemonset.yaml             # Ollama DaemonSet for GPU/CPU laptop workers with model cache on NFS/local SSD
│       │   ├── vllm-deployment.yaml              # High-throughput vLLM Deployment for NVIDIA >=8GB VRAM workers
│       │   └── llama-cpp-rpc-daemonset.yaml      # llama.cpp rpc-server DaemonSet for cross-node tensor/layer sharding
│       └── exo/
│           ├── Dockerfile.exo                    # Containerized Exo distributed inference node
│           └── exo-statefulset.yaml              # Peer-to-peer Exo cluster manifest across heterogeneous laptops
└── docs/
    └── hardware/
        ├── bom-and-thermals.md                   # Complete Bill of Materials, power budget, and vertical rack thermal design
        └── bios-wol-matrix.md                    # Vendor-specific BIOS/UEFI settings for Dell, Lenovo, HP, and Apple hardware
```

## 2. Phase 1: Physical Hardware & Base OS Provisioning (The Control Plane)

### 2.1 Hardware Target Specifications
| Component | Specification | Technical Role & Constraint |
| :--- | :--- | :--- |
| **Compute Node** | Raspberry Pi 5 (8GB LPDDR4X-4267 SDRAM) | Always-on (`24/7/365`) Control Plane, Ingress Gateway, NAS Host |
| **Power Supply** | Official Raspberry Pi 27W USB-C PD (`5.1V @ 5A`) | Prevents under-voltage throttling during peak NVMe I/O bursts |
| **Primary Storage** | 2TB M.2 2280 NVMe SSD (PCIe Gen 3 x1 via M.2 HAT+) | Hosts K3s state (`/mnt/nvme/k3s`), NFS exports, S3 buckets, Git repos |
| **Boot Media** | 32GB Industrial A2 microSD (or direct NVMe EEPROM boot) | Read-mostly `/boot/firmware` and OS root; write-heavy paths offloaded to NVMe |
| **Cooling** | Official Raspberry Pi 5 Active Cooler (PWM Blower + Heatsink) | Maintains SoC `< 65°C` under continuous ingress/storage load |
| **Network** | Onboard Gigabit Ethernet (`eth0`, IEEE 802.3ab) | Connected directly to primary LAN router/switch for deterministic WoL broadcast |

### 2.2 Base OS & Kernel Configuration (`arm64`)
* **Target OS:** Debian 12 (Bookworm) 64-bit Lite or Ubuntu 24.04 LTS Server (`aarch64`, Kernel `>= 6.6.x`).
* **PCIe Gen 3 & Fan Curve Enablement (`/boot/firmware/config.txt`):**
  Append the following exact lines to `/boot/firmware/config.txt`:

  ```ini
  dtparam=pciex1
  dtparam=pciex1_gen=3
  dtparam=cooling_fan=on
  dtparam=fan_temp0=50000,fan_temp0_hyst=5000,fan_temp0_speed=75
  dtparam=fan_temp1=60000,fan_temp1_hyst=5000,fan_temp1_speed=150
  dtparam=fan_temp2=67500,fan_temp2_hyst=5000,fan_temp2_speed=255
  dtoverlay=disable-wifi
  dtoverlay=disable-bt
  ```

* **Kernel Cgroups & Memory Accounting (`/boot/firmware/cmdline.txt`):**
  Ensure `/boot/firmware/cmdline.txt` is a single line containing:

  ```text
  console=serial0,115200 console=tty1 root=PARTUUID=<ROOT_PARTUUID> rootfstype=ext4 fsck.repair=yes rootwait quiet cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1
  ```

* **Deterministic NVMe Partitioning & `/etc/fstab` Mount:**
  Format `/dev/nvme0n1p1` as `ext4` (`mkfs.ext4 -L HOMELAB_NVME -m 1 -E lazy_itable_init=0,lazy_journal_init=0 /dev/nvme0n1p1`), retrieve its UUID via `blkid -s UUID -o value /dev/nvme0n1p1`, create `/mnt/nvme`, and append to `/etc/fstab`:

  ```text
  UUID=<NVME_UUID>  /mnt/nvme  ext4  defaults,noatime,nodiratime,nofail,commit=60,errors=remount-ro  0  2
  ```

* **NVMe Swapfile & Kernel `sysctl` Tuning:**
  Disable microSD swap and allocate an 8GB swapfile on `/mnt/nvme/swapfile` (`sw,pri=10` in `/etc/fstab`). Populate `/etc/sysctl.d/99-homelab-k3s.conf` with `vm.swappiness=10`, `vm.vfs_cache_pressure=50`, `vm.overcommit_memory=1`, `net.ipv4.ip_forward=1`, `net.ipv6.conf.all.forwarding=1`, `net.bridge.bridge-nf-call-iptables=1`, `fs.inotify.max_user_instances=8192`, and `fs.inotify.max_user_watches=524288`.

### 2.3 Shared Storage Layer on the 2TB NVMe SSD
Create `/mnt/nvme/{k3s,nfs/shared,nfs/models,nfs/k8s-pvs,samba/cloud,minio/data,minio/config,git,immich/library,postgres}`:
* **NFSv4 (`/etc/exports`):** Export `/mnt/nvme/nfs` (`fsid=0`), `/mnt/nvme/nfs/shared`, `/mnt/nvme/nfs/models`, and `/mnt/nvme/nfs/k8s-pvs` restricted strictly to the Tailscale CGNAT subnet `100.64.0.0/10(rw,sync,no_subtree_check,no_root_squash)`.
* **Samba (`/etc/samba/smb.conf`):** Bind strictly to `interfaces = lo tailscale0` (`bind interfaces only = yes`, `server min protocol = SMB3`, `smb encrypt = required`), exporting `[cloud]` at `/mnt/nvme/samba/cloud`.
* **MinIO S3 (`/etc/systemd/system/minio.service`):** Bind `--address "${PI_TAILSCALE_IP}:9000"` and `--console-address "${PI_TAILSCALE_IP}:9001"` backed by `/mnt/nvme/minio/data`.

## 3. Headless "Clamshell" Worker Node Prep (Repurposed Laptops/Desktops)

### 3.1 Lid Switch & Systemd Sleep Disabling
* Populate `/etc/systemd/logind.conf.d/00-homelab-headless.conf` with `HandlePowerKey=ignore`, `HandleSuspendKey=ignore`, `HandleHibernateKey=ignore`, `HandleLidSwitch=ignore`, `HandleLidSwitchExternalPower=ignore`, `HandleLidSwitchDocked=ignore`, `LidSwitchIgnoreInhibited=no`, and `IdleAction=ignore`.
* Execute `systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target && systemctl restart systemd-logind`.
* Zero panel backlight brightness via `/sys/class/backlight/*/brightness` and `consoleblank=60`.

### 3.2 Display/GPU Activation
* Enable `nvidia-persistenced.service` and `nvidia-smi -pm 1`.
* Install `xserver-xorg-video-dummy` and write `/etc/X11/xorg.conf.d/10-headless-gpu.conf` defining `DummyGPU0` (`256000` VideoRam) and `1920x1080_60.00` modeline, or append `video=HDMI-A-1:1920x1080@60D` to `GRUB_CMDLINE_LINUX_DEFAULT` (paired with a 4K60 HDMI EDID dummy plug on hardware-muxed laptops).

### 3.3 Battery Protection ("Spicy Pillow" Prevention)
* **TLP (`/etc/tlp.d/00-homelab-battery.conf`):** Set `START_CHARGE_THRESH_BAT0=50`, `STOP_CHARGE_THRESH_BAT0=80`, `START_CHARGE_THRESH_BAT1=50`, `STOP_CHARGE_THRESH_BAT1=80`, `CPU_ENERGY_PERF_POLICY_ON_AC=balance_performance`, `USB_AUTOSUSPEND=0`.
* **Kernel `sysfs` Udev Rule (`/etc/udev/rules.d/99-battery-threshold.rules`):**

  ```text
  SUBSYSTEM=="power_supply", KERNEL=="BAT[0-9]", ATTR{charge_control_start_threshold}=="*", ATTR{charge_control_start_threshold}="50"
  SUBSYSTEM=="power_supply", KERNEL=="BAT[0-9]", ATTR{charge_control_end_threshold}=="*", ATTR{charge_control_end_threshold}="80"
  ```

### 3.4 Physical Enclosure, Thermal Specs & BOM
* Mount laptops vertically in an anodized aluminum stand with a 15mm silicone lid spacer (hinges facing upward for natural convection exhaust) cooled by dual 120mm 5V USB PWM intake fans across the bottom/keyboard vents.
* Configure BIOS/UEFI `Wake on LAN from S4/S5 = Enabled`, `ErP / Deep Sleep = Disabled`, `Restore on AC Power Loss = Power On`, and enable `wol-enable@.service` (`ethtool -s %i wol g`).

## 4. Network Mesh Fabric (Tailscale Layer)

* **Unattended Join (`scripts/network/tailscale-join.sh`):** Configures `rx-udp-gro-forwarding on` on the egress NIC and runs `tailscale up --authkey="${TS_AUTHKEY}" --hostname="${TS_HOSTNAME}" --advertise-tags="${TS_TAGS}" --accept-dns=true --accept-routes=true --reset`.
* **ACL Tag Strategy (`tailscale/policy.hujson`):**
  * `tag:control-plane` (`pi-control`, `100.64.0.10`): Exposed to `autogroup:member` exclusively on ports `22, 80, 443, 445, 2049, 2222, 6443, 9000, 9001`.
  * `tag:worker` & `tag:gpu-node` (`blade-x86-01`, `blade-x86-02`, `blade-arm-01`): Isolated from regular client traffic; full bidirectional mesh access (`*`) with `tag:control-plane` and peer workers.

## 5. Orchestration & The "Single-Interface Illusion" (K3s over Tailscale)

* **Control Plane Bootstrap (`/etc/rancher/k3s/config.yaml` on Pi 5):**
  `write-kubeconfig-mode: "0644"`, `data-dir: "/mnt/nvme/k3s"`, `node-ip: "100.64.0.10"`, `advertise-address: "100.64.0.10"`, `bind-address: "100.64.0.10"`, `flannel-iface: "tailscale0"`, `flannel-backend: "vxlan"`, `node-monitor-period=5s`, `node-monitor-grace-period=20s`, `default-not-ready-toleration-seconds=30`, `default-unreachable-toleration-seconds=30`.
* **Worker Node Join (`/etc/rancher/k3s/config.yaml` on Laptops):**
  `server: "https://100.64.0.10:6443"`, `node-ip: "{{ worker_tailscale_ip }}"`, `flannel-iface: "tailscale0"`, labels `homelab.io/node-role=worker` and `homelab.io/power-profile=ephemeral-laptop`, taints `homelab.io/ephemeral=true:NoSchedule` and (if GPU present) `nvidia.com/gpu=present:NoSchedule`.
* **Unified Traefik Ingress (`k3s/manifests/03-ingress-traefik.yaml`):**
  Pinned to `homelab.io/node-role: control-plane` with `externalIPs: ["100.64.0.10"]` exposing ports `80`, `443`, and `2222`. All requests to `*.homelab.internal` / `pi-control.homelab.ts.net` terminate at the Pi and reverse-proxy over `tailscale0` to worker pods.

## 6. Core Workload Modules (Phased Rollout)

1. **Personal Cloud Storage (`services/storage/`):**
   * Filebrowser / Nextcloud + Immich Server, Web, PostgreSQL (`pgvecto-rs`), and Redis pinned to `pi-control` (`arm64`) backed by `/mnt/nvme`.
   * `immich-machine-learning` deployed to `homelab.io/node-role: worker` with `homelab.io/ephemeral=true:NoSchedule` and `nvidia.com/gpu=present:NoSchedule` tolerations, mounting `/mnt/nvme/nfs/models/immich-ml`.
2. **Self-Hosted Dev & CI/CD Hub (`services/cicd/`):**
   * Forgejo (`codeberg.org/forgejo/forgejo:9`) pinned to `pi-control` (`/mnt/nvme/git/forgejo`).
   * `act_runner` + rootless `buildkit` Deployment pinned to `kubernetes.io/arch: amd64` worker blades (`homelab.io/node-role: worker`).
3. **Distributed AI Inference Engine (`services/inference/`):**
   * **Gateway (`services/inference/gateway/main.py`):** Async FastAPI middleware + Redis job queue (`homelab:inference:queue`) + `wakeonlan` dispatcher running on `pi-control:8000` in front of LiteLLM (`:4000`). If all GPU workers are asleep when `/v1/chat/completions` arrives, the Pi broadcasts a UDP Magic Packet on `eth0`, polls worker `/api/tags` every `1.5s` (up to `60s`), and streams the response once awake.
   * **Backend Workers (`services/inference/workers/` & `exo/`):** Ollama and vLLM for single-node inference (`7B–32B`), plus `llama.cpp` `rpc-server` (`--rpc 100.64.0.22:50052`) as the primary deterministic sharding engine for `70B+` GGUF models and `Exo` (`--discovery-module manual`) for mixed Apple Silicon + CUDA rings.

## 7. Implementation Roadmap & Task Breakdown for Claude Code

| Task ID | Input | Action | Verification Command & Expected Stdout |
| :--- | :--- | :--- | :--- |
| **Task 1.1** | Section 1 Tree | Generate monorepo directories, `.env.example`, `Makefile`, `ansible.cfg` | `find . -maxdepth 3 -not -path '*/.*' \| sort` -> Matches all 8 modules |
| **Task 1.2** | `.env.example` | Generate `ansible/inventory/hosts.yml` and `group_vars/*.yml` | `ansible-inventory -i ansible/inventory/hosts.yml --list -y` -> Valid YAML |
| **Task 2.1** | Section 2.2 | Write & run `pi5-bootstrap.sh` and `01-pi-base-os.yml` | `ssh root@100.64.0.10 "stat -fc %T /sys/fs/cgroup && findmnt -n -o SOURCE /mnt/nvme"` -> `cgroup2fs` & `/dev/nvme0n1p1` |
| **Task 2.2** | Section 2.3 | Write & run `02-storage-layer.yml` (NFSv4, Samba, MinIO) | `curl -s -o /dev/null -w '%{http_code}\n' http://100.64.0.10:9000/minio/health/live` -> `200` |
| **Task 3.1** | Section 3.1–3.3 | Write & run `03-laptop-clamshell-prep.yml` & `verify-battery-cap.sh` | `ansible workers -m shell -a "/usr/local/bin/verify-battery-cap.sh"` -> `BATTERY_CAP_OK: start=50 stop=80` |
| **Task 3.2** | Section 3.2, 3.4 | Enable `wol-enable@.service`, dummy Xorg, `nvidia-persistenced` | `nvidia-smi --query-gpu=persistence_mode --format=csv,noheader` -> `Enabled` |
| **Task 4.1** | Section 4 | Join Tailscale mesh (`04-tailscale-mesh.yml`) & apply `policy.hujson` | `tailscale ping -c 1 100.64.0.21` -> `pong ... via <LAN_IP>:<PORT>` |
| **Task 4.2** | Section 5.1–5.3 | Deploy K3s Server (`05-k3s-control-plane.yml`) & Agents (`06-k3s-workers.yml`) | `kubectl get nodes` -> `pi-control` (`arm64`) and `blade-x86-*` (`amd64`) `Ready` |
| **Task 5.1** | `k3s/manifests/` | Apply namespaces, storage classes, priority classes, Traefik ingress | `kubectl -n kube-system get svc traefik -o jsonpath='{.spec.externalIPs[0]}'` -> `100.64.0.10` |
| **Task 5.2** | `services/storage/` | Deploy Filebrowser, Immich Core (Pi), and Immich ML (Worker) | `curl -ks https://photos.homelab.internal/api/server-info/ping` -> `{"res":"pong"}` |
| **Task 5.3** | `services/cicd/` | Deploy Forgejo (Pi) and `act_runner` (x86_64 Workers) | `curl -ks https://git.homelab.internal/api/v1/version` -> Valid JSON version |
| **Task 5.4** | `services/inference/` | Deploy FastAPI WoL Gateway, LiteLLM, Ollama/vLLM, `llama.cpp` RPC | `curl -ks https://ai.homelab.internal/v1/models` -> Lists `qwen2.5-coder-14b`, `llama-3.1-8b-fast`, `llama-3.3-70b-sharded` |

## 8. Non-Functional Requirements, Edge Cases & Failure Modes

* **Split-Brain & Tailscale DERP Fallback Safeguards:** K3s runs a single authoritative control plane on `pi-control` backed by local NVMe (`/mnt/nvme/k3s`), eliminating multi-node etcd quorum loss when laptop workers disconnect. Flannel VXLAN MTU is clamped to `1230` bytes (inside `tailscale0`'s `1280` MTU) with iptables `TCPMSS --clamp-mss-to-pmtu`, and `tailscale-lan-watchdog.timer` checks `.CurAddr` every 60s to recover direct LAN UDP paths if a node falls back to a DERP relay.
* **Zero microSD Wear on Raspberry Pi 5:** K3s `data-dir` is bound to `/mnt/nvme/k3s` (relocating SQLite, containerd, and kubelet volumes to NVMe), `systemd-journald` is configured with `Storage=volatile` (`RuntimeMaxUse=64M`), and `/tmp` + `/var/tmp` use `tmpfs`. Verification via `iostat -d -k 1 5 mmcblk0` confirms `0.00 kB_wrtn/s` steady-state microSD write load.
* **Graceful Worker Eviction (`<= 50s` Total Window):** With `node-status-update-frequency=5s`, `node-monitor-grace-period=20s`, and `default-unreachable-toleration-seconds=30`, an unplugged laptop is marked `NotReady` and removed from `EndpointSlices` in 20 seconds, and stateless `Deployment` pods (`immich-machine-learning`, `act_runner`) reschedule within 50 seconds total while DaemonSets (`ollama`, `llama-cpp-rpc`) remain bound and resume on Wake-on-LAN.