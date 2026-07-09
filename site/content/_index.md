---
title: "ITScape — KVM/arm64 guest-to-host escape tracking"
description: "Linux kernel KVM/arm64 vGIC-ITS double-put race (CVE-2026-46316, ITScape) — guest-to-host escape — distro patch status tracker"
layout: "single"
date: 2026-07-08
lastmod: 2026-07-09
cover:
  image: "itscape-tracker.png"
  alt: "ITScape — Linux KVM/arm64 vGIC-ITS guest-to-host escape tracker"
  hiddenInSingle: true
---

## Summary

| Field | Detail |
|---|---|
| CVE ID | CVE-2026-46316 |
| Alias | `ITScape` (the name its [PoC][poc] uses) |
| Component | Kernel: KVM/arm64 vGIC-ITS emulation — `vgic_its_invalidate_cache()` (`arch/arm64/kvm/vgic/vgic-its.c`) |
| Type | Guest-to-host escape — translation-cache reference double-put → `vgic_irq` use-after-free |
| Impact | A guest with **root/EL1** can execute code with **host-kernel privilege** — an in-kernel KVM escape, independent of QEMU |
| Upstream fix | [`13031fb6b835`][fix] (*KVM: arm64: vgic-its: Drop the translation cache reference only for the erased entry*); first in **v7.1** |
| Introduced | [`8201d1028caa`][intro] in **v6.10** (2024-04-25) |
| Affected window | Kernels **6.10 through 7.0** (reachable); ≥ the per-branch fix is patched; **< 6.10 not affected** |
| Scope | **arm64 only** — the first known guest-to-host escape for KVM/arm64; x86 hosts are not affected |
| Discoverer | Hyunwoo Kim ([`@v4bel`][poc]) |
| Public disclosure | 2026-07 (post-embargo, reported to linux-distros) |
| Public PoC | [V4bel/ITScape][poc] (built on the in-tree KVM selftest; drops `/ITScape` on the host) |
| CVSS | 3.1 **8.8** — `AV:L/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H` (kernel CNA) |
| Related | [Januscape (CVE-2026-53359)][januscape] — the x86 sibling by the same researcher (KVM/x86 shadow-MMU escape). ITScape is arm64-only; Januscape is x86-only |

## How the exploitation chain works

ITScape is a use-after-free in KVM/arm64's **vGIC-ITS** (Interrupt
Translation Service) emulation. `vgic_its_invalidate_cache()` walks the
per-ITS translation cache with `xa_for_each()` and drops the cache's
reference on each entry with `vgic_put_irq()` — but it puts the *iterated*
pointer rather than the value returned by `xa_erase()`.

The function runs from contexts that do **not** exclude one another: the
ITS command handlers hold `its_lock`, the `GITS_CTLR` write path holds
`cmd_lock`, and the path that clears `EnableLPIs` in a redistributor's
`GICR_CTLR` holds neither. Two or more can drain the same cache
concurrently; if each observes the same entry, erases it, and then puts it,
the single reference the cache holds is dropped **more than once**. The
`vgic_irq` can then be freed while an ITE still maps it — a
use-after-free.

A guest with EL1 (guest-kernel) privilege triggers this by driving the
relevant interrupt-related ITS operations to race the invalidation paths.
Because the bug is in **in-kernel KVM** rather than QEMU userspace, a
successful exploit runs with **host-kernel** privilege — not the privilege
of a userspace VMM process. The PoC, built on the kernel's own KVM
selftest, escapes the guest and creates `/ITScape` on the host.

> :information_source: ITScape is **arm64-only** — it lives in the GICv3/4
> ITS emulation that x86 KVM does not have. The x86 KVM escape by the same
> researcher is a different bug, [Januscape][januscape]. Triggering ITScape
> needs guest **root/EL1**; there is no clean runtime mitigation knob, so
> **the kernel backport is the only thing that flips a verdict here**.

## Vulnerable commit range

| Commit | Role | Description |
|---|---|---|
| [`8201d1028caa`][intro] | Introduced | vGIC-ITS translation-cache rework (v6.10) — began putting the iterated pointer rather than the erased entry. |
| [`13031fb6b835`][fix] | Fixed | *KVM: arm64: vgic-its: Drop the translation cache reference only for the erased entry* — uses `xa_erase()`'s atomic return so each entry is put exactly once; first released in **v7.1**. |

The reachable lifetime is therefore **v6.10 through v7.0**; kernels older
than 6.10 never had the buggy rework and are **not affected**. This is a
much narrower window than the x86 [Januscape][januscape] sibling, whose bug
dates to 2010.

## Upstream fixed versions

The fix landed in **v7.1** and the kernel CNA (CVE-2026-46316) backported it
to every maintained in-window stable line: **6.12.93**, **6.18.35**, and
**7.0.12**. The pre-6.10 longterm lines (6.6.y, 6.1.y, 5.15.y, 5.10.y)
predate the introducing commit and are not affected.

| Branch | Status | Current | Notes |
|---|---|---|---|
| Linus mainline | :white_check_mark: Carries `13031fb6b835` | v7.2-rc2 | first fixed release v7.1 |
| 7.1.x | :white_check_mark: Carries the fix | 7.1.3 | fix shipped in the 7.1 release |
| 7.0.x | :white_check_mark: Carries the backport | 7.0.14 | in window; first fixed 7.0.12 |
| 6.18.x | :white_check_mark: Carries the backport | 6.18.38 | LTS; in window; first fixed 6.18.35 |
| 6.12.x | :white_check_mark: Carries the backport | 6.12.95 | LTS; in window; first fixed 6.12.93 |
| 6.6.x | :heavy_minus_sign: Not affected | 6.6.144 | LTS; predates the trigger (< 6.10) |
| 6.1.x | :heavy_minus_sign: Not affected | 6.1.177 | LTS; predates the trigger (< 6.10) |
| 5.15.x | :heavy_minus_sign: Not affected | 5.15.211 | predates the trigger |
| 5.10.x | :heavy_minus_sign: Not affected | 5.10.260 | predates the trigger |

The short-lived 6.10.y / 6.11.y stable branches were in-window but reached
end of life; when verifying a tree directly, the fixed function is
`vgic_its_invalidate_cache()` in `arch/arm64/kvm/vgic/vgic-its.c`.

## Distribution status

The deciding fact per release is whether the **arm64 kernel** is in the
6.10–7.0 window *and* lacks the [`13031fb6b835`][fix] backport. A kernel
older than 6.10, or carrying the fix, is not exploitable. *Fixed since*
records the date the kernel fix first lands in that release.

The rows below track the arm64 builds of a focused set of distributions.
**Proxmox VE is x86-only** and therefore does not appear here — its KVM
exposure is covered by the [Januscape][januscape] tracker. Other systems
named in the disclosures appear only in prose where relevant.

| Distribution | Release | Kernel | Fixed since | Status |
|---|---|---|---|---|
| Debian | sid (unstable) | 7.1.3-1 | 2026-07-08 | :white_check_mark: Fixed — ships 7.1.3 (carries the fix) |
| Debian | forky (testing) | 7.0.13-1 | 2026-07-08 | :white_check_mark: Fixed — 7.0.13 ≥ 7.0.12 (carries the backport) |
| Debian | 13 (trixie) | 6.12.95-1 | 2026-07-08 | :white_check_mark: Fixed — 6.12.95-1 via trixie-security (DSA-6355-1; ≥ 6.12.93) |
| Debian | 12 (bookworm) | 6.1.170-3 | — | :white_check_mark: Not affected — predates the trigger (< 6.10) |
| Debian | 11 (bullseye, LTS) | 5.10.223-1 | — | :white_check_mark: Not affected — predates the trigger |
| NixOS | Unstable | 6.18.38 | 2026-07-08 | :white_check_mark: Fixed — ships 6.18.38 (carries the backport) |
| NixOS | 26.05 | 6.18.38 | 2026-07-08 | :white_check_mark: Fixed — ships 6.18.38 (carries the backport) |
| Rocky Linux | 10 | 6.12.0-211.28.1.el10_2 | — | :x: Vulnerable — RHEL fixed (RHSA-2026:34911, 211.30.1); Rocky rebuild pending |
| Rocky Linux | 9 | 5.14.0-687.17.1.el9_8 | — | :x: Vulnerable — RHEL 9 affected via vGIC backport; RHSA-2026:36018 fix (687.22.1) not yet in Rocky |
| Rocky Linux | 8 | 4.18.0-553.el8_10 | — | :white_check_mark: Not affected — predates the trigger |
| Amazon Linux | 2023 | 6.1.x (amzn2023) | — | :white_check_mark: Not affected — default stream < 6.10 |
| Amazon Linux | 2 | 4.14.x (amzn2) | — | :white_check_mark: Not affected — predates the trigger |
{.distros}

### Debian

Debian's arm64 `linux` is affected only from 6.10 on. **sid** (`7.1.3-1`,
carries the fix) and **forky** (testing, `7.0.13-1` ≥ the first-fixed
7.0.12) are fixed. **trixie** stable received `6.12.95-1` via
`trixie-security` (DSA-6355-1); since 6.12.95 ≥ 6.12.93 it carries the
backport — trixie is now fixed. **bookworm** (6.1) and **bullseye** (5.10)
predate the v6.10 trigger and are not affected.

### NixOS

The default `linuxPackages` (`linux_6_18`) on both nixos-unstable and
nixos-26.05 is `6.18.38`, which carries the `13031fb6b835` backport
(6.18.38 ≥ 6.18.35) on aarch64 as on x86; `linuxPackages_latest`
(`linux_7_1`) is `7.1.3`. Both channels are fixed.

### Rocky Linux / RHEL family

On arm64, **EL10** (the 6.12-based el10 kernel) is in the ITScape window —
but so is **EL9**: Red Hat backported the vGIC-ITS code into its el9 5.14
kernel, so RHEL/Rocky 9 carry the bug despite the 5.14 base (this is why a
version-only "predates 6.10" check is wrong for EL — only EL8's 4.18
genuinely predates the code). Red Hat has fixed **RHEL 10** in
RHSA-2026:34911 (`6.12.0-211.30.1.el10_2`) and **RHEL 9** in RHSA-2026:36018
(`5.14.0-687.22.1.el9_8`). Rocky rebuilds RHEL, so its kernels reach those
NVRs when the RLSAs land; as of this writing Rocky 10 is at
`6.12.0-211.28.1.el10_2` and Rocky 9 at `5.14.0-687.17.1.el9_8`, both below
the fixed builds, so both remain `:x:` (Rocky typically trails Red Hat by a
day or two — AlmaLinux 10 has already shipped a fixed kernel). RHEL 8 /
Rocky 8 (4.18) are **not affected**; Oracle Linux and CloudLinux OS track
RHEL.

### Amazon Linux

Amazon has assessed the default streams **not affected**: AL2023's default
`kernel` is the 6.1 stream and AL2's is 4.14 — both predate v6.10. AL2023's
opt-in `kernel6.12` / `kernel6.18` streams were in-window and have been
fixed: `kernel6.12` via ALAS2023-2026-1894 and `kernel6.18` via
ALAS2023-2026-1881 (both released 2026-06-22). The livepatch packages were
also updated in the same window.

## Detection

**Is this an arm64 host?**  ITScape is arm64-only; `aarch64` output means
the arch is in scope (x86 hosts are not affected):

```bash
uname -m
```

**Is the running kernel in the affected window and missing the fix?**  The
bug is reachable on 6.10 through 7.0; compare the running kernel against the
*Upstream fixed versions* table and your distro row above:

```bash
uname -r
```

**Is this a KVM host?**  The bug is in the host's vGIC-ITS emulation, so it
matters only where KVM is actually used to run guests:

```bash
ls -l /dev/kvm
```

**Does the platform expose a GICv3/4 ITS?**  The vulnerable code emulates
the ITS; a host GIC with ITS is the usual case on server-class arm64:

```bash
ls /sys/firmware/devicetree/base 2>/dev/null | grep -i its
```

## Public PoC

The upstream PoC is in [V4bel/ITScape][poc]; it is built on the kernel's
in-tree KVM selftest and, on a successful escape, creates `/ITScape` on the
host. Do **not** run it on a system you are not authorised to test.

## Mitigation

The real fix is a patched kernel (the `13031fb6b835` backport). There is
**no** clean runtime mitigation — the bug is reached through ordinary guest
ITS operations, so it cannot be disabled without breaking interrupt
delivery for guests. Until a patched kernel is installed, the only
risk-reducing postures are operational:

- Do not run **untrusted** arm64 guests on an unpatched host — a guest with
  EL1 is all the attacker needs.
- Where guests are semi-trusted, restricting who can open `/dev/kvm` limits
  which local users can start a hostile guest, but does not help against an
  already-running untrusted guest.

Neither is a fix; the kernel hole remains until patched.

## Risk notes

- **Untrusted-guest arm64 hosts:** this is a guest-to-host-root primitive
  from inside a VM, and the first of its kind demonstrated on KVM/arm64 —
  the headline risk for arm64 multi-tenant hosts.
- **In-kernel, not QEMU:** the escape runs with host-kernel privilege, so
  userspace VMM hardening (seccomp, QEMU sandboxing) does not contain it.
- **Narrow window:** only 6.10-and-later arm64 kernels are affected, so
  many stable EL and Debian releases are simply out of scope — check the
  distribution row before assuming exposure.
- **Backports available (CVE-2026-46316):** the fix has landed in 6.12.93,
  6.18.35, 7.0.12, and 7.1; distro kernels that have not yet adopted one of
  those releases remain vulnerable. Check the distribution row for your
  kernel.

## Verification log

*Last verified 2026-07-09.*

### Upstream

- The fix is `13031fb6b835` (*KVM: arm64: vgic-its: Drop the translation
  cache reference only for the erased entry*), first released in **v7.1**.
  It uses `xa_erase()`'s atomic return value so each cache entry is put
  exactly once even under concurrent invalidation.
- The bug was introduced by `8201d1028caa` in **v6.10** (2024-04-25); older
  kernels never had the buggy rework and are not affected.
- **CVE-2026-46316** assigned by the kernel CNA (confirmed via `vulns.git`
  `origin/master`; record keys on `13031fb6b835`). The `.dyad` gives the
  per-branch fixed versions used in the *Upstream fixed versions* table;
  the CNA scored it CVSS 3.1 **8.8** (`AV:L/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H`).
- **Stable backports landed**: 6.12.93, 6.18.35, and 7.0.12 all carry the
  fix (per the `vulns.git` `.dyad`); every maintained in-window branch is
  therefore already patched upstream, so only distro adoption lags.

### Distributions

- **Debian** (via the dak `madison` API and Debian security tracker): unstable
  `7.1.3-1` and testing `7.0.13-1` carry the fix → fixed. Trixie received
  `6.12.95-1` via `trixie-security` (DSA-6355-1; ≥ 6.12.93, carries the
  backport) → trixie now fixed; status flipped from `:warning:` to
  `:white_check_mark:`. Oldstable `6.1.170-3` and oldoldstable
  `5.10.223-1` predate the v6.10 trigger → not affected.
- **NixOS** (via the local nixpkgs clone at both channel revisions): the
  default `linuxPackages` (`linux_6_18`) is `6.18.38` on both nixos-unstable
  and nixos-26.05 (≥ 6.18.35) → carries the backport → fixed. No change.
- **Rocky / RHEL family** (via the Red Hat security data API, OSV, and
  Rocky BaseOS aarch64 repodata): Red Hat lists RHEL 8 `kernel` **Not
  affected**, but **RHEL 9 and 10 affected and fixed** — RHSA-2026:36018
  (`5.14.0-687.22.1.el9_8`) and RHSA-2026:34911 (`6.12.0-211.30.1.el10_2`);
  RHEL 9's 5.14 kernel carries the vGIC-ITS code by backport despite
  predating upstream 6.10, so the earlier "el9 not affected" reading was
  wrong. Rocky trails: Rocky 9 `5.14.0-687.17.1.el9_8` and Rocky 10
  `6.12.0-211.28.1.el10_2` are both below the fixed builds → both remain
  `:x:` until the RLSA rebuilds land (AlmaLinux 10 already ships
  `6.12.0-211.30.1.el10_2`). OSV shows no Rocky ecosystem entry yet.
- **Amazon Linux**: ALAS CVE page confirmed: AL2023 default `kernel` (6.1)
  and AL2 (4.14) **Not Affected** (< 6.10). The opt-in `kernel6.12` stream
  was fixed via ALAS2023-2026-1894 and `kernel6.18` via ALAS2023-2026-1881
  (both 2026-06-22). Prose updated accordingly.
- **Proxmox VE**: x86-only product — not applicable to this arm64 tracker;
  see the Januscape tracker instead.

## References

| Source | URL |
|---|---|
| Public PoC (V4bel) | <https://github.com/V4bel/ITScape> |
| Companion tracker — Januscape (x86) | <https://kimmo.cloud/januscape/> |
| Kernel fix | <https://github.com/torvalds/linux/commit/13031fb6b8357fbbcded2a7f4cba73e4781ee594> |
| CVE-2026-46316 | <https://www.cve.org/CVERecord?id=CVE-2026-46316> |
| The Hacker News writeup | <https://thehackernews.com/2026/07/16-year-old-linux-kvm-flaw-lets-guest.html> |
| Debian security tracker | <https://security-tracker.debian.org/tracker/CVE-2026-46316> |
| Debian package madison (dak-backed) | <https://api.ftp-master.debian.org/madison?package=linux&s=sid,forky,trixie,bookworm,bullseye&text=on> |
| AlmaLinux errata | <https://errata.almalinux.org/> |
| Amazon Linux ALAS | <https://alas.aws.amazon.com/cve/html/CVE-2026-46316.html> |
| stable point release banner | <https://www.kernel.org/finger_banner> |
{.references}

[poc]: https://github.com/V4bel/ITScape
[januscape]: https://kimmo.cloud/januscape/
[fix]: https://github.com/torvalds/linux/commit/13031fb6b8357fbbcded2a7f4cba73e4781ee594
[intro]: https://github.com/torvalds/linux/commit/8201d1028caa4fae88e222c4e8cf541fdf45b821
