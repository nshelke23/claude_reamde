# claude_reamde

Running context for Claude sessions working on **NetPortal**. If you're a Claude instance picking this up on a new host/instance: read this whole file before touching anything. It exists so you don't have to re-derive things the hard way — several of the notes below cost real debugging time to learn.

## What NetPortal is

A browser-based network lab virtualization platform: SvelteKit (Svelte 5) frontend, FastAPI backend (async SQLAlchemy, Alembic), PostgreSQL, Caddy reverse proxy, QEMU/KVM + Docker + Dynamips for node virtualization. Runs on Ubuntu 24.04.

## The repos

| Repo | What it is |
|---|---|
| [`net2.0`](https://github.com/nshelke23/net2.0) | **The only source of truth.** Backend, frontend, `deploy.sh` (the installer), build scripts. Every other repo below is a *build output* of this one — never hand-edit a deployment artifact directly, the fix dies on the next rebuild. |
| [`netportal-bare`](https://github.com/nshelke23/netportal-bare) | The `netportal-X.Y.run` installer, for a server that already has Ubuntu 24.04 — run it directly. |
| [`netportal-qcow2`](https://github.com/nshelke23/netportal-qcow2) | Ready-made KVM/Proxmox/VMware(via `qemu-img convert`) VM disk. Boots and auto-starts, no install step. |
| [`netportal-vmware`](https://github.com/nshelke23/netportal-vmware) | Unattended installer ISO for Hyper-V/VMware/any hypervisor — installs onto the VM's own disk, like a Windows/Ubuntu install ISO. Split into 2 parts (GitHub's 2GB asset cap). |
| `claude_reamde` (this repo) | This document. Update it when you finish meaningful work so the next session doesn't start cold. |

### How a change propagates

```
net2.0 (the code)
   │  deploy/scripts/build-run.sh
   ▼
netportal-X.Y.run ──────────────────► netportal-bare (published as-is)
   │                         │
   │ build-qcow2.sh          │ build-vmware-iso.sh
   ▼                         ▼
netportal-qcow2          netportal-vmware
```

Fine-tune features in `net2.0` only. To ship a new release: bump the version banner in `deploy.sh`, `build-run.sh`, then feed the resulting `.run` through `deploy/scripts/build-qcow2.sh` and `deploy/scripts/build-vmware-iso.sh` (both live in `net2.0/deploy/scripts/`, both take `--run <path>`).

## Architecture facts worth knowing

- **Filesystem**: app code at `/opt/netportal/`, secrets/config at `/etc/netportal/backend.env`, all runtime state (labs, images, templates) at `/var/lib/netportal/`, logs at `/var/log/netportal/`.
- **Services**: `netportal-backend` (uvicorn, 127.0.0.1:8000), `caddy` (:80, reverse-proxies `/api/*` and `/ws/*` to the backend, serves the static SPA), `postgresql`, `docker`. Plus `netportal-postboot`/`netportal-bridge-cloud-learning` (bridge-cloud network re-provisioning after reboot) and `netportal-mem-tuning` (KSM/swappiness).
- **Install**: `deploy.sh` is an 11-stage, resumable, idempotent installer. `sudo ./netportal-X.Y.run -- -y` for fully unattended.
- **Default NetPortal login**: `admin` / `netportal` (as of v5.8 — was `root` / `netportal` before that; both the install template and the live production instance were changed).
- **Console architecture**: no Guacamole — built-in WebSocket console proxy (xterm.js/noVNC), including an admin-only real host-shell tab (added this session, server-side-enforced role gate, audit-logged).
- **Docker images used by lab nodes**: tagged `netportal-lab/*`. The "Universal VM" node type (`netportal-pktgen:latest`) is a general-purpose test client — Firefox-via-VNC, hping3/nmap/curl/iperf3/tcpdump, and `wpa_supplicant` for 802.1X testing.

## Landmines already found (don't rediscover these)

1. **`TopologyCanvas.svelte` contains 2 legitimate embedded NUL bytes** (a map-key delimiter). Plain `grep` on this file silently misbehaves — always use `grep -a`.
2. **`virt-customize --firstboot` runs *before* `multi-user.target`.** If the firstboot script invokes something that does `systemctl enable --now <unit ordered After=multi-user.target>` (deploy.sh's density-tuning stage does exactly this), it deadlocks — the unit waits for multi-user.target, which waits for the firstboot service to finish, which is waiting for the unit. `deploy.sh` already tolerates this gracefully (its own warning + continue); it's not something to "fix," just something to know about when building images this way.
3. **The opposite problem on the ISO installer**: `late-commands` in a Subiquity autoinstall run inside a `curtin in-target` chroot — an *unbooted* filesystem, not a live system. Real services (postgres, docker) can't actually start there. The real NetPortal install has to happen via a systemd unit on the target's genuine first boot, and that unit must be ordered *after* `multi-user.target` (opposite of #2).
4. **Never boot the exact file you're about to ship, then checksum it.** Booting a qcow2 writes to it (journal, machine-id). The checksum you compute before boils down to being wrong by the time you upload. Boot a throwaway copy for verification, checksum the untouched original.
5. **GitHub Releases caps a single asset at 2GB.** The ISO (~3.5GB) needs `split -b 1900M -d --numeric-suffixes=1`, uploaded as separate assets with a `cat`-to-reassemble README section and checksums for both the parts and the whole.
6. **Removing `linux-headers-*`/`linux-tools-*`/`llvm-18` from the QCOW2 build looks safe (they're genuinely unused) but isn't**: purging them makes apt try to pull a *newer kernel* as a side effect of resolving the `linux-generic`/`linux-virtual` meta-package family. That needs network mid-build and silently changes what kernel ships. Left alone on purpose — not worth ~150MB for that risk.
7. **`apt-get clean` does NOT clear `/var/lib/apt/lists`** — that's the package-index cache (250MB+), a completely separate thing from the downloaded-`.deb` cache. Both need clearing for a lean image.
8. **`pgrep -f <pattern>` inside a Bash/Monitor tool's own command string can match its own process** if the pattern text happens to also appear in the wrapper's command line (e.g. checking `pgrep -f "curl.*live-server"` while your own script's text literally contains that string). Causes an infinite "still running" false positive. Prefer checking an exact PID (`kill -0 $PID`) over a name/pattern match when scripting a wait loop.
9. **`xorriso -extract_boot_images`** gives you real, generic boot-image files (`systemarea.img`, `gpt_part2_efi.img`) instead of hand-computed byte offsets into the source ISO — use this, not manual `dd` math, so the ISO-remastering script survives future Ubuntu point releases with a different on-disk layout.
10. **`command -v a b c` in a shell script only reliably checks the first argument** in some contexts — don't trust a multi-arg existence check without verifying each one individually first.
11. **Docker Engine 29's `containerd-snapshotter=true`** stores image layers under `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs`, *not* the classic `/var/lib/docker/overlay2`. If `/var/lib/docker` looks suspiciously empty after a `docker load`, check there before assuming data loss.
12. **A `RUN ... <<'EOF' cat > file` heredoc in a classic (non-BuildKit) Dockerfile silently produces a 0-byte file.** The heredoc body doesn't make it into the `RUN` instruction's shell invocation. Use `COPY` with a separate file instead.
13. **`curl --data-binary @bigfile` can fail with "out of memory" on large uploads** (hit this on a 3.67GB ISO) — it reads the whole file into memory before sending. Use `-T`/`--upload-file` instead, which streams from disk.
14. **Renaming a NetPortal user's username isn't a plain `UPDATE`.** `users.username` is the primary key, and `labs.owner`/`pods.username` are foreign keys to it with `NO ACTION` (not `CASCADE`) — check with `SELECT confupdtype FROM pg_constraint WHERE confrelid = 'users'::regclass` before assuming otherwise. A direct rename violates the FK the moment any row references the old value. Safe pattern: insert a new row with the new username (copy every column, clear `session_token`/`session_expires` so a fresh login is required), repoint every dependent table's FK column, *then* delete the old row — all in one transaction. Renaming the account you're currently logged in as also invalidates that session no matter what, since the client's cookie still names the old username.

## Session log (what's been built)

Chronological, most recent first. Each entry is a `net2.0` commit unless noted otherwise.

- **v5.8 release** — Copy Node + Snapshot as Image + v1.1 branding + admin/netportal default, all rolled into one `.run`/GitHub release. Verified the actual release artifact (extracted the `.run`, grepped the real files) rather than just trusting the build log, since it's cheap insurance against a stale/wrong upload.
- **Default admin credentials changed to `admin`/`netportal`** (was `root`/`netportal`) — two separate things, both done: (1) `deploy/env/backend.env.example` template fix, so future fresh installs default to `admin`; (2) the *live production instance's* actual database account renamed too, via the FK-safe transaction pattern (see landmine #14). Also fixed a real pre-existing bug: the installer's non-quiet summary hardcoded `"Username: root"` literally instead of reading the `admin_user` variable the quiet-mode summary a few lines above already computed correctly.
- **Branding: "NetPortal v1.1"** on `/login` and `/labs` — they'd drifted apart (v4.0 vs v2.0) before this made them consistent. Purely cosmetic UI text, unrelated to the actual `deploy.sh` release version numbering.
- **"Snapshot as Image"** — right-click a *stopped* QEMU node → flattens its current disk (config, IPs, installed packages, everything on it, not just its starting state) into a new standalone base image under `images/qemu/`, via `qemu-img convert` merging the node's COW overlay with its backing chain. Node must be stopped (enforced server-side — converting a live-written qcow2 risks a half-flushed disk); QEMU-only for now (Docker/IOL/dynamips images work completely differently); admin-gated (adds to the shared image library). Runs off the event loop (`asyncio.to_thread`) since conversion can take a while on a large disk. Verified live: snapshotted a FortiGate node with real prior state, confirmed the output has zero backing-file reference, confirmed a new node could actually be created from it.
- **"Copy Node"** — right-click a node → creates an independent new node with the same image/cpu/ram/ethernet/console config. Not a clone: `uuid`/`firstmac` are dropped from the copied `extras` so the backend mints a fresh identity, same as any newly-created node (new id, new QEMU uuid, new MAC, starts stopped). Reuses the existing single-node-create endpoint — no backend changes needed.
- **v5.7 release** — labs-running indicator + lab download/import + the two build scripts, rolled into one `.run`/release.
- **Lab download/import** — `GET /api/labs/{path}/download` serves the raw lab JSON; `POST /api/labs/import` (multipart upload) creates a new lab from it, reassigning a fresh id and auto-suffixing on a filename collision. Lets a lab move between NetPortal instances. UI: a download button next to Rename/Delete on each lab card, an "Import" toolbar button.
- **Labs-running indicator** — the folder-listing endpoint now flags each lab `running: bool` (cross-referencing `NodeRuntimeService`'s live node registry against each lab's own `id`). Labs with active nodes render with a red border/icon/badge in the `/labs` list.
- **`deploy/scripts/build-qcow2.sh` and `build-vmware-iso.sh`** — turn the manual, hand-debugged build sessions that produced the qcow2/ISO into one-command, verified pipelines. `build-qcow2.sh` is proven (ran fully unattended end-to-end). `build-vmware-iso.sh` encodes the same commands already proven manually but hasn't been run start-to-finish as a script yet.
- **Three deployment repos created**: `netportal-qcow2` (897MB→797MB after optimization passes — toolchain/docs/snapd/apt-lists-cache stripped, zero feature loss; demo docker images intentionally dropped to hit that size, a real trade-off, not a bug), `netportal-vmware` (autoinstall ISO, full install→reboot→firstboot cycle verified), `netportal-bare` (the plain `.run`, republished standalone).
- **Admin host-shell console** — a real root shell on the NetPortal host, reachable from the browser console bar (docked/tabbed/floating), gated to `role == admin` server-side (not just UI-hidden), every open audit-logged at WARNING.
- **Bridge-cloud link color** — changed from orange to light gray.
- **Network tile redesign** — networks render as a square tile matching node cards, not an oversized icon with a text pill; fixed link/port attachment to the tile perimeter.
- **Network icon picker** — "Change Icon…" on the canvas right-click menu.
- **v5.6 release** — rolled up the above into one `.run`/release.
- **"Universal VM" platform feature** — promoted an ad-hoc EAP-TTLS Docker test client (`netportal-pktgen:latest`) into a first-class node type, selectable from Add Element like any node/network. Fixed a real bug found along the way: `build_node_catalog()` picked the *first* marked docker image alphabetically instead of the template's own declared image when multiple docker templates existed.
- **802.1X/EAP-TTLS root-cause + durable fix** — a lab bridge's `group_fwd_mask` sysfs value was blocking EAPOL frames (IEEE 802.1D reserved multicast range). Fixed live, then made it self-healing: a `bridge-tune` verb in the privileged helper + a startup reconciliation pass that retunes any bridge found with a stale mask (proven by deliberately breaking one and confirming the next backend restart fixed it automatically).
- **v5.5 release, Bridge-Cloud UX** — smaller/editable icon on bridge-cloud networks, orange link coloring (later changed to gray, see above).

## Pending

`net2.0` is fully pushed and released as of **v5.8** — nothing sitting local-only there right now (check `git log origin/main..HEAD` in a fresh clone to be sure this hasn't drifted since).

What's genuinely pending: **`netportal-qcow2`, `netportal-vmware`, and `netportal-bare` are stale** — all three were last built against v5.6 content and have not been rebuilt against v5.7 or v5.8. Concretely, none of them have Copy Node, Snapshot as Image, the v1.1 branding, or the `admin`/`netportal` default credentials yet. To bring them current: build a `.run` from the latest `net2.0` (`deploy/scripts/build-run.sh`), then run `deploy/scripts/build-qcow2.sh --run <path>` and `deploy/scripts/build-vmware-iso.sh --run <path>`, then publish all three the same way the v5.6 ones were done (see the repo table above).

Push/release cadence in this project has been "commit locally, ask before pushing" for individual features, but the user has been asking for pushes/releases fairly readily once a batch of work is done — don't assume silence means "don't push," but don't push without being asked either.

## How to actually work here

1. Clone `net2.0`, make the change, test it against the **live running instance** (deploy the changed files directly to `/opt/netportal/` + `/var/lib/netportal/www/`, restart `netportal-backend`, rebuild+rsync the frontend) — this codebase's convention is real verification against a live host, not just "it builds."
2. `python3 -m py_compile` the backend files and `npm run check` the frontend before calling anything done.
3. Never restart `netportal-backend` carelessly — it uses `KillMode=process` specifically so a restart doesn't kill in-flight QEMU/docker/dynamips lab nodes; a plain `systemctl restart` is safe, but always verify `ps aux | grep qemu-system` afterward to confirm nothing was disrupted.
4. Commit locally with a clear message explaining *why*, not just *what*. Don't push/release without being asked.
5. Update this file when you finish something worth a future session knowing about.
