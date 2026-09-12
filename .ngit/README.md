# Nostr CI for this repo (ngit-ci)

`ngit-ci` runs GitHub-Actions-format workflows through `act` and publishes signed
results to Nostr. It executes **only** the files in `.ngit/act/workflows/` —
`.github/workflows/` is detected but never executed by it. So Nostr CI and
GitHub CI coexist: nothing here changes the GitHub runs.

## What this workflow was verified against

| | |
| --- | --- |
| GitHub source | `https://github.com/FreedomTechFeed/packages` |
| Branch | `master` (the default branch; the coordinator mirrors this one) |
| Verified commit | `63c6224177eef99c28f5f219e7151ef52544dd08` (`Merge pull request #3 from FreedomTechFeed/feat/add-missing-arches`) |
| Content under test | `net/tollgate-wrt/{Makefile,test.sh,test-version.sh,test-feed-ci.sh}`, plus the repo's own `.github/workflows/` that `test-feed-ci.sh` inspects — in a tree of 5032 tracked files |

An earlier revision of this file was authored against a **different**
repository's content (a fork whose `master` branch holds no
`net/tollgate-wrt` at all), which is why the coordinator's run of it reported
`FAIL: no *.sh test harness in net/tollgate-wrt`: the checks were proven against
one tree and executed against another. Every step below was re-executed against
**this** branch before being committed; the raw results are in "Local
verification" and the omissions in "Not covered".

## What runs

`.ngit/act/workflows/packages-test.yml` — one job (`package-check`),
`timeout-minutes: 15`:

| Step | Why |
| --- | --- |
| Install `shellcheck` | The act job containers are lighter than GitHub runner VMs and do not guarantee shellcheck. The step is a no-op when a usable shellcheck is already on `PATH`, which is what keeps it runnable verbatim outside CI. |
| The package's own test harnesses | The **exact set** `test.sh`, `test-version.sh`, `test-feed-ci.sh` must be present (`test.sh`/`test-version.sh` are the harnesses the openwrt/packages buildbot runs after installing the package; `test-feed-ci.sh` validates this repo's CI config), executable — the buildbot runs them from the checkout — must parse under `sh -n`, and must pass `shellcheck --severity=error`. Pinning the set makes a lost harness a loud failure instead of a smaller green run. |
| Run `test-feed-ci.sh` | This one is a full run, not just a lint: it asserts the repo's own `.github/workflows/` still build `tollgate-wrt` for the bench arch `aarch64_cortex-a53`/`mediatek-filogic`, publish per-arch release assets with deterministic `tollgate-wrt_<version>_<arch>` names, preserve the existing arches, and keep the known-broken runtime smoke test non-blocking. |
| Makefile lint | The upstream-submission shape: all required fields present (incl. `PKG_SOURCE_VERSION`, `PKG_TARBALL_DIR`), `BuildPackage` evaluated, `golang-package.mk` included as `../../lang/golang/golang-package.mk`, **no** `./files/` install path and **no** vendored `files/` directory (this branch installs its runtime files from the tarball, so a vendored copy here would be a second drifting source of truth), install lines read from `$(PKG_TARBALL_DIR)/packaging/files/`, no `REPLACES`, no `luci` dependency, `PKG_LICENSE:=GPL-3.0-only` with `PKG_LICENSE_FILES:=LICENSE`, `PKG_VERSION` apk-legal (no hyphen — `apk mkpkg` rejects it on v24.10+) while `PKG_SOURCE` / `PKG_SOURCE_URL` / `PKG_BUILD_DIR` are all tag-named from `PKG_SOURCE_VERSION`, real 64-hex `PKG_HASH`, and `test-version.sh`'s hardcoded `PKG_VERSION` equal to the Makefile's `PKG_SOURCE_VERSION`. |
| `PKG_HASH` + tarball layout + `packaging/files` | Re-downloads the release tarball named by `PKG_SOURCE_VERSION`, compares sha256, asserts it extracts to `tollgate-module-basic-go-$(PKG_SOURCE_VERSION)/` with `src/go.mod`, and that **every** `$(PKG_TARBALL_DIR)/packaging/files/...` path the install recipe references exists inside the tarball (glob-aware, so `man/man8/*.8` and the recursive site copy are handled). Since the runtime files are not vendored here, this is the only cheap way to catch a Makefile/tarball mismatch. |

Two details that are easy to get wrong and are pinned deliberately:

- The `REPLACES` / `DEPENDS` policy greps use `^[[:space:]]*`. Both live **inside**
  the `define Package/tollgate-wrt` block and are tab-indented there, so an
  anchored `^DEPENDS:=` regex can never match — a check that cannot fail.
- `PKG_VERSION` and `PKG_SOURCE_VERSION` are *supposed* to differ here
  (`0.6.0_alpha1` vs `0.6.0-alpha1`); the split and its reason are documented in
  the Makefile. `test-version.sh` tracks the **source** version because that is
  what the CLI's `-X` ldflags inject. Do not "fix" the apparent mismatch.

## Local verification

Driven end to end with the whole file, not a hand-copied command list
(`run-workflow-locally.py` parses the YAML, applies job `env`, and runs every
`run:` step with `bash -e` in a clean worktree of this branch):

```
--- RUN Install shellcheck ---            [step exit=0]  (shellcheck 0.11.0 already on PATH)
--- RUN Check the package's own test harnesses ---
OK   net/tollgate-wrt/test-feed-ci.sh
OK   net/tollgate-wrt/test.sh
OK   net/tollgate-wrt/test-version.sh
Test harnesses present, executable, parse, and pass shellcheck (errors only). [step exit=0]
--- RUN Run the repo's CI-config harness (test-feed-ci.sh) ---
OK: matrix override references aarch64_cortex-a53/mediatek-filogic in multi-arch-test-build.yml
OK: matrix override references aarch64_cortex-a53/mediatek-filogic in release-publish.yml
OK: release-triggered asset upload present in multi-arch-test-build.yml
OK: release-triggered asset upload present in release-publish.yml
OK: PKG_VERSION=0.6.0_alpha1
OK: deterministic asset naming pattern present in release-publish.yml
OK: existing arch mipsel_24kc preserved … / mips_24kc … / x86_64 …   (2 workflows each)
OK: PR CI vendored and always builds tollgate-wrt in multi-arch-test-build.yml
OK: runtime-test container build is non-blocking (continue-on-error) …
OK: runtime smoke test is non-blocking (continue-on-error) …
test-feed-ci: PASS                                                    [step exit=0]
--- RUN Lint the package Makefile ---
Makefile lint passed (PKG_VERSION=0.6.0_alpha1, PKG_SOURCE_VERSION=0.6.0-alpha1). [step exit=0]
--- RUN Verify PKG_HASH and packaging/files against the pinned tarball ---
expected=c0ca1b37cbccde8e46de8e42287185d322c648bb10ad317b3e496a136a0e35f7
actual  =c0ca1b37cbccde8e46de8e42287185d322c648bb10ad317b3e496a136a0e35f7
PKG_HASH matches the pinned release.
Tarball layout matches PKG_TARBALL_DIR/PKG_BUILD_DIR.
OK   packaging/files/etc/hotplug.d/iface/95-tollgate-restart
OK   packaging/files/etc/init.d/tollgate-wrt
OK   packaging/files/etc/nftables.d/20-nds-enforce.nft
OK   packaging/files/etc/nftables.d/30-backend-firewall.nft
OK   packaging/files/etc/uci-defaults/90-tollgate-captive-portal-symlink
OK   packaging/files/etc/uci-defaults/99-tollgate-setup
OK   packaging/files/lib/upgrade/keep.d/tollgate
OK   packaging/files/man/man8/*.8
OK   packaging/files/tollgate-captive-portal-site/.
OK   packaging/files/usr/bin/check_package_path
OK   packaging/files/usr/bin/tollgate-apply-ssl
OK   packaging/files/usr/bin/tollgate-remove-ssl
OK   packaging/files/usr/local/bin/first-login-setup
All 13 referenced packaging/files paths exist in the tarball.      [step exit=0]

ALL RUN STEPS PASSED
```

Negative controls (each gate re-run against a deliberately poisoned copy of the
tree under `~/worktrees/.negctl/`, asserting it goes red): 6/6 behaved as
expected — non-upstream license, hyphens in `PKG_VERSION`, `test-version.sh`
pinned to a stale version, a removed harness, a `./files/` install path
reintroduced, and an install reference to a `packaging/files` path the tarball
does not contain. In every case the intended step failed with its own `FAIL:`
message and nothing else did.

shellcheck note: CI installs the distro build via `apt-get` (Ubuntu 24.04 ships
0.9.0) and the local run used 0.11.0. Later shellcheck releases add checks, so a
tree clean at 0.11.0 is clean at 0.9.0 at the same `--severity=error`.

## Not covered (deliberately)

- **The OpenWrt SDK build.** The GitHub side builds through
  `openwrt/gh-action-sdk@v11` (`multi-arch-test-build.yml`) and an 8-arch ×
  2-SDK matrix (`release-publish.yml`). Both need a Docker container and 30-45
  minutes per architecture; ngit-ci refuses `container:`/`services:` blocks and
  caps a job at 30 minutes. **No OpenWrt build is attempted here**, and none was
  attempted locally either. It stays the authoritative gate on GitHub.
- **Executing `test.sh` / `test-version.sh`.** They assert on an *installed*
  package (`/usr/bin/tollgate-wrt`, `/usr/bin/tollgate`) inside an OpenWrt test
  rootfs. That is the buildbot's job; here they are only checked for presence,
  executable bit, parse and shellcheck. `test-feed-ci.sh` **is** executed because
  it needs nothing installed.
- **The other ~5000 files in this openwrt/packages fork.** Repo-wide lint is not
  gated: upstream CI owns that, and a whole-tree check on a fork branch is not
  something this workflow can honestly pass or fail.
- **Package index generation** (`make package/index`) needs the SDK — out of scope.
- **Any arch-specific assertion.** Nothing in this job is arch-dependent; the
  arch matrixes belong to the SDK builds above.
- **`.github/workflows/scripts/ci_helpers.sh`** is not shellchecked here (it is
  clean at default severity, but it is GitHub-workflow plumbing rather than
  package content, and `test-feed-ci.sh` already covers the CI *semantics*).

## Differences from the sibling `feed` repo's workflow

Both repos ship `net/tollgate-wrt/`, but they are different shapes and the two
workflows deliberately check different things:

| | `FreedomTechFeed/feed` (`main`) | `FreedomTechFeed/packages` (`master`) |
| --- | --- | --- |
| Package shape | standalone `src-git` feed | `openwrt/packages` submission |
| Runtime files | **vendored** in `net/tollgate-wrt/files/` (25 files) | install from the tarball's `packaging/files/`; nothing vendored |
| `golang-package.mk` | `$(TOPDIR)/feeds/packages/lang/golang/…` | `../../lang/golang/golang-package.mk` |
| Versioning | one field, `PKG_VERSION:=0.5.0` keys the tag | split: `PKG_VERSION:=0.6.0_alpha1` / `PKG_SOURCE_VERSION:=0.6.0-alpha1` |
| Test harnesses | none | `test.sh`, `test-version.sh`, `test-feed-ci.sh` |
| Gates unique to it | vendored-`files/` existence, orphan-install, `diff -r` vs the tarball's `packaging/files/` | harness set + `test-feed-ci.sh` run, `./files/`-must-not-exist, apk-legal version, harness↔Makefile version agreement, per-path `packaging/files` existence |

## Where this file may live

`.ngit/` sits on the branch `ci/ngit-workflows`, **not** on `master`. That is
deliberate: the GitHub→ngit bridge mirrors with a plain `git push` (no
`--force`), so a file that exists only on the ngit mirror's default branch would
make the bridge's next push non-fast-forward and break the mirror. A dedicated
ref keeps the mirrored content identical to GitHub and leaves zero upstream diff
for the fork's PR flow, and the coordinator runs the workflow when that ref is
pushed or when a manual trigger (kind `9840`) targets it.

## Triggers

`on: push` and `on: pull_request` — no `schedule` (ngit-ci has no timer), no
`workflow_dispatch` dependency, no secrets, no `GITHUB_TOKEN`. Each run is at
most 15 minutes; the coordinator's own ceiling is 30.

## Authorization: this repo runs `request-required`

Ordinary `push`/`pull_request` runs do **not** start on their own. A maintainer
must first publish a **standing Service Request** (kind `9843`) for this repo:

```jsonc
{ "kind": 9843, "content": "",
  "tags": [["a","30617:<maintainer-pubkey>:<repo-id>"],
           ["p","<coordinator-pubkey>"]] }
```

Until that exists the coordinator logs `Skipping push trigger until an authorized
Service Request is observed`. A one-shot manual trigger (kind `9840`) bypasses
the gate and is useful for a first smoke run. The workflow result quotes the
Service Request it ran under in its `q` tag.

## Reading the results

```bash
nak req -k 9841 -a <coordinator-hex> -l 20 wss://relay.ngit.dev   # per-job results
nak req -k 9842 -a <coordinator-hex> -l 5  wss://relay.ngit.dev   # workflow conclusion
nak req -k 39842 -a <coordinator-hex> -l 20 wss://relay.ngit.dev  # progress
```

- **9841** — job result, `content` carries the log tail.
- **9842** — workflow result/conclusion.
- **39842** — workflow progress.

Older `ngit` builds on this fleet have no `ci` subcommand, so read the relays
directly rather than expecting `ngit ci status`.

## If a step fails inside the act container

The act images are lighter than GitHub runner VMs: treat a failure as an
environment gap until proven otherwise (missing tool or incomplete base image),
not automatically as a code failure. This workflow installs `shellcheck` itself
for exactly that reason, and every step after that needs nothing but `curl`,
`tar`, `compgen`, `diff` and the base shell.
