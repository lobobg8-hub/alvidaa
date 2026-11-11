# 50C350LS (Toshiba) - V0000.09.09M.P1124

**Model:** 50C350LS (Toshiba OEM, `brand=tos`)
**Firmware:** V0000.09.09M.P1124
**SoC:** MediaTek MTK9602
**Region:** SA/Brazil (`region=8`)
**VIDAA OS:** U09.09
**Browser:** Chromium 111 "odin"
**`transport_protocol`:** 3290
**Kernel:** Linux 4.19.79+ (Nov 2025)
**Contributed by:** gh:lobobg8-hub (own unit)

> **What this guide includes and leaves out.** This page records what was observed on a single owned unit: which capabilities were reachable, what the evidence shows, and what remains unverified. It does not include the operational payload, the exact command sequence, or a copy-paste recipe. The route family is already documented in the public `09.09O` / `09.09D` guides; this page maps the deltas observed on this build and leaves the operator to adapt the method to their own unit.

## Root - superwatch FaaS family (verified 2026-11-11)

Same method family as the `09.09O` (55A7KAU) and `09.09D` (55E7QTUK) superwatch chains. Three deltas were measured on this build:

1. **Pairing requires `brand=tos`.** The alvidaa `pair.py` hardcodes `his` in the client identity and is refused with `rc=135` on this unit. `pyvidaa`, which discovers the brand via UPnP, pairs successfully. This is a controller-identity issue, not a transport or certificate issue.
2. **The fetcher is a service, not a function.** On this build the fetcher lives under `CHA/services/fetcher/index.js`, not under `CHA/functions/fetch/sw2.js`. The function-bridge shim documented in the `09.09O` guide is absent here and has to be placed through the file bridge before the superwatch route works.
3. **The firmware has no `/etc/rc.local` and no `functions/fetch/sw2.js`.** The `50A51HUV` boot-hook route and the `09.09O` function-bridge route are both closed on this build. The superwatch socket is the measured-open route.

The superwatch install command observed on this generation matches the `09.09O` family, not the `09.60` family.

## Evidence

What was observed on this unit:

- **Direct root proof:** `/usr/local/bin/busybox id` prints `uid=0(root) gid=0(root)`. The firmware has no `id` in `PATH` (no `/bin/id`, `/usr/bin/id`, or `/bin/busybox`), so the proof uses `busybox id` explicitly.
- **Direct root proof via post-install:** a probe IPK's post-install script runs as root. `test.ipk` writing to `/tmp/ipk_proof.txt` returns `EACCES` (not `ENOENT`), i.e. the file exists and is root-owned with restrictive permissions. `proof.ipk` writing to `/APPS/ipk_proof.txt` with an explicit `chmod 644` succeeds.
- **Interactive root shell:** a reverse-shell IPK (`rootsh5.ipk`, three methods: `bash -i` over `/dev/tcp`, FIFO + `toybox nc`, and `exec 5<>` + `bash -i`) lands a `bash-3.2#` prompt on the attacker's listener with `uid=0(root)`. Both `ESTABLISHED` TCP sessions and the shell prompt were observed.
- **Kernel and runtime:** `cat /proc/version` returns Linux 4.19.79+ (built Nov 2025). `bash` is 3.2.57. `toybox_selinux` ships `nc`, `telnetd`, `wget`, `httpd`, `ftpget`. `/dev/tcp` works outbound on four tested ports.

**Evidence grade:** direct. Root execution and identity are both observed directly (`uid=0` from `busybox id`, and the `bash-3.2#` prompt from a reverse shell).

## What does NOT exist on this firmware

- `/etc/rc.local` (the `09.01O` boot-hook route)
- `/var/local/otaTest.sh` (the `50A51HUV` hook)
- `jsservice.system` (the `09.01O` JS service package)
- `/bin/id`, `/usr/bin/id`, `/bin/busybox` (the `id` binary is not in `PATH`; `/usr/local/bin/busybox id` works)
- a native `functions/fetch/sw2.js` (the shim must be placed through the file bridge on this build)

## Limits

- **Persistence across reboot was not tested.** The observation is a session-level root execution. No `serviceList` entry, no boot hook and no re-arm mechanism was installed or verified on this unit. The `serviceList` persistence route (used on `09.09O`) is untested here.
- **The `sw2.js` shim must be placed before the superwatch route is usable** on this build; it is not present in the factory image.
- **`ipk-update -n` returns a non-zero code for non-system packages** after a successful post-install. The post-install runs first; this does not invalidate the root execution.
- **No SSH, Dropbear or persistent firewall rule was deployed.** `iptables` rules were applied during the session (REJECT on the OTA CDN and public DNS) but they are volatile and were lost on reboot. No persistent rule was written.
- **Editing `/APPS/channelA/hisenseUI/static/js/sdk.js` to block ad domains prevents the home screen from launching.** The original file was restored from a backup (`sdk.js.bak`). This is a dead end for ad removal on this build; the router-side DNS block is the working route.

## Reversibility

Every write was placed on a writable partition (`/APPS`); nothing was written to the verified-boot rootfs. The steps to return the unit to its prior state are the standard ones for the method family: remove the placed shim, remove the test packages from `/APPS/`, remove the proof files from `/APPS/`, and reboot (which clears `/tmp/` and any volatile firewall state). No read-only partition was modified during the study.

The `sdk.js` edit was reverted from its backup before this guide was written; the unit's home screen is stock.

## Disclosure

Findings and evidence only. No command sequence, no payload, no copy-paste recipe. The route family is already public in the `09.09O` / `09.09D` guides; this page maps the deltas observed on this build so the next owner of this model knows what to expect and what not to assume.