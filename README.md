# ITScape — Linux KVM/arm64 guest-to-host escape tracking site

Patch-status tracker for **ITScape** (**CVE-2026-46316**), a KVM/arm64
guest-to-host escape in the Linux kernel.  A translation-cache double-put in
`vgic_its_invalidate_cache()` — it drops a `vgic_irq`'s cache reference on
the iterated pointer rather than the value returned by `xa_erase()`, and the
function runs from mutually-non-exclusive contexts — frees the `vgic_irq`
while an ITE still maps it under concurrent invalidation: a use-after-free a
guest with **root/EL1** can drive to **host-kernel** code execution.
Because the bug is in in-kernel KVM rather than QEMU userspace, the escape
runs with host-kernel privilege.  This is the first known guest-to-host
escape for KVM/arm64.  Discovered by Hyunwoo Kim (`@v4bel`) and
[disclosed in 2026-07](https://thehackernews.com/2026/07/16-year-old-linux-kvm-flaw-lets-guest.html).
Public PoC: <https://github.com/V4bel/ITScape>.

The bug was introduced by `8201d1028caa` in **v6.10** (2024-04-25) and fixed
in v7.1 by
[`13031fb6b835`](https://github.com/torvalds/linux/commit/13031fb6b8357fbbcded2a7f4cba73e4781ee594)
(*KVM: arm64: vgic-its: Drop the translation cache reference only for the
erased entry*).  The practical exploitable window is arm64 kernels **6.10
through 7.0**; kernels older than 6.10 are **not affected**.  Distro
adoption of the backport is tracked below.

**CVE-2026-46316** is assigned (CVSS 3.1 **8.8**); the kernel CNA backported
the fix to 6.12.93, 6.18.35, and 7.0.12.  ITScape is **arm64-only**; its
**x86 sibling** by the same researcher is **Januscape (CVE-2026-53359)**, a
KVM/x86 shadow-MMU escape tracked at <https://kimmo.cloud/januscape/>.

The rendered site is published at **<https://kimmo.cloud/itscape/>**.
Deployment plan and current setup state live in [`WEBSITE.md`](WEBSITE.md).

## Source of truth

The tracker is a single Hugo page: [`site/content/_index.md`](site/content/_index.md).
Edit that file; everything else is build infrastructure.

## Local development

Requires Hugo extended (≥ 0.146.0) and Go (for Hugo Modules to fetch the
PaperMod theme).

### With Nix (recommended)

```sh
nix develop          # dev shell: hugo, go, git, resvg
cd site
hugo server          # local preview at http://localhost:1313/itscape/
```

If you use [direnv](https://direnv.net/), `direnv allow` once and the dev
shell auto-activates whenever you `cd` into the repo.

### Without Nix

Install Hugo extended ≥ 0.146.0 and Go ≥ 1.24 yourself, then:

```sh
cd site
hugo server          # http://localhost:1313/itscape/
```

## Build and publish

```sh
make build       # local build into site/public/
make dist        # build, then rsync to haig:/itscape/
make banner      # re-rasterise the social banner SVG → PNG (needs resvg + Roboto)
```

`make dist` runs `make build` first.  `make banner` is only needed after
editing `site/assets/itscape-tracker.svg`; the rendered PNG is committed.

## Repo layout

```
.
├── flake.nix              # Nix dev environment (hugo, go, git, resvg + RPM tools)
├── .envrc                 # direnv hook → `use flake`
├── .gitignore
├── Makefile               # `make build`, `make dist`, `make banner`
├── LICENSE                # CC BY 4.0
├── README.md              # this file
├── CLAUDE.md              # project instructions for Claude Code
├── WEBSITE.md             # publication plan / decisions log
├── scripts/               # auto-update agent: prompt + driver
├── systemd/               # user-level timer + service units
└── site/                  # Hugo project
    ├── hugo.toml
    ├── content/
    │   └── _index.md      # the tracker (single page)
    ├── assets/css/extended/custom.css # PaperMod CSS overrides
    ├── assets/itscape-tracker.svg     # social-banner source (→ make banner)
    ├── static/itscape-tracker.png     # rendered OpenGraph banner (committed)
    ├── layouts/partials/  # PaperMod overrides (post_meta, extend_footer)
    ├── go.mod, go.sum     # Hugo Modules — pulls PaperMod theme
    └── …                  # standard Hugo skeleton
```

## License

[CC BY 4.0](LICENSE) — share and adapt with attribution.
