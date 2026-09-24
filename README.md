# genup

A tool to update the **Portage**(5) tree, all installed packages, and the kernel on Gentoo Linux.

---

## Background and Fork History

This project is a maintained fork of
[sakaki-/genup](https://github.com/sakaki-/genup).

The original upstream project received an
[EOL notice](https://forums.gentoo.org/viewtopic-p-8522963.html#8522963), and over time no longer aligned with my operational needs. While evaluating alternatives, I also reviewed
[phosphorcube/gentoo-update](https://github.com/phosphorcube/gentoo-update), another rewrite fork that is now unmaintained. Some ideas and approaches from that project were incorporated as well.

This fork exists to **preserve the spirit and workflow of the original genup**, while extending it with additional features required for modern Gentoo deployments.

---

## Notable Changes from Upstream

Compared to the original sakaki-/genup, this fork adds or improves support for:

* Email notifications via `sendmail`

  * Relies on a system mailer (for example `nullmailer`) being configured
* Rebuilds of `@live-rebuild` (-9999) packages on every run

  * Disable with `--no-live-rebuild`
* Detection of read-only Portage trees and overlays

  * Useful for NFS-mounted Portage or overlay setups
* Automatic `git pull` of `/etc/portage/patches`

  * For users maintaining local patch repositories
* Automatic mounting and unmounting of `/boot`
* Automatic use of local binary packages via `--usepkg`

  * Enabled when `PKGDIR/Packages` exists; useful for NFS-mounted bindirs
* Kernel build support via `genkernel`

  * Used if installed and `buildkernel` is not available

The script has been deliberately decoupled from a hard dependency on the ebuilds that configure `emtee` and `buildkernel`. If these optional components are present on the system, genup will automatically detect and use them.

---

## Description

**genup** is a utility intended to simplify the process of keeping a Gentoo system fully up to date. When invoked, it performs the following steps in order:

* Updates the Portage tree and active overlays, and syncs **eix**(1)

  * Using `eix-sync`, or `emaint sync --auto` (`emerge-webrsync` in webrsync-gpg mode) if eix is not installed (plus `emaint sync --auto` first when the deprecated `webrsync-gpg` FEATURE is set); skipped if any repository is read-only
* Checks for unread news items

  * Listed for the final report/email; with `--ask`, offers to read them (`eselect news read new`) before anything is updated
* Updates Portage user patches (if `/etc/portage/patches` is a git repo)

  * Using `git -C /etc/portage/patches pull`
* Mounts `/boot` if required, and remounts writable when necessary

  * Using `findmnt`, `mount`, and `umount`
* Removes any prior **emerge**(1) resume state

  * Using `emaint --fix cleanresume`
* On `aarch64`, attempts to apply pending fixups

  * By running `/etc/cron.weekly/fixup` if present (errors are non-fatal)
* Ensures **Portage**(5) itself is up to date

  * Using `emerge --oneshot --update portage` (failures are warnings)
* Ensures **genup** itself is up to date

  * Using `emerge --oneshot app-portage/genup` (restarting if the version changes; failures are warnings)
* Updates selected toolchain packages first (best effort)

  * Using `emerge --oneshot --update` for `sys-libs/glibc`, `sys-devel/binutils`, `dev-build/cmake`, plus `sys-devel/gcc` (gcc mode) or `llvm-core/clang`, `llvm-core/llvm`, `llvm-core/lld` (clang mode); only already-installed packages are updated; see `--compiler`
* Checks and repairs gcc configuration if invalid (best effort)

  * Using `gcc-config`, `env-update`, and re-emerging `libtool`
* Attempts a preliminary `@world` update using **emtee**(1)

  * If `emtee` is installed and not disabled; failure is non-fatal
* Updates all packages in the `@world` set

  * Using `emerge --deep --with-bdeps=y --changed-use --update --backtrack=50 --keep-going @world`; build failures are retried with `-j1` and distcc disabled, dependency/config failures are fatal (unless `--ignore-required-changes`)
* Rebuilds live (-9999) packages (unless `--no-live-rebuild`)

  * Using `emerge --exclude app-portage/genup @live-rebuild`; failure is fatal
* Removes unreferenced packages (first pass)

  * Using `emerge --depclean` (skipped if `webapp-config` reports unused installs)
* Rebuilds packages depending on stale libraries

  * Using `emerge @preserved-rebuild` (run twice, second pass suppressing getbinpkg)
* Updates outdated **perl**(1) modules

  * Using `perl-cleaner --all` (if installed)
* Upgrades the kernel if possible

  * Using `buildkernel --stage-only` (to staging, in `/boot`) when available; otherwise `genkernel all`, followed by `eclean-kernel -n 2` (if installed) and `grub-mkconfig` or `lilo`
* Builds any external modules (such as those for VirtualBox), if a kernel was built this run

  * Using `emerge @module-rebuild --exclude '*-bin'`
* Resolves clashing config file changes

  * Using `dispatch-conf` (in interactive mode, or if forced via `--dispatch-conf` and a tty is available)
* Removes unreferenced packages (second pass)

  * Using `emerge --depclean` (same `webapp-config` check)
* Fixes missing shared library dependencies

  * Using `revdep-rebuild`
* Rebuilds packages depending on stale libraries (second pass)

  * Using `emerge @preserved-rebuild`
* Removes unused distfiles older than two weeks (unless `--keep-old-distfiles`)

  * Using `eclean --deep --time-limit=2w distfiles`; skipped when `DISTDIR` is a mount point or on a network filesystem (e.g. shared over NFS)
* Removes binary packages for versions no longer in the tree, older than four weeks (only with `--purge-old-binpkgs`)

  * Using `eclean --time-limit=4w packages` (not `--deep`, so binpkgs merely not installed on this host are kept); skipped when `PKGDIR` is a mount point or on a network filesystem
* Deploys a staged kernel, if available and requested

  * Using `buildkernel --copy-from-staging` (buildkernel only)
* Updates environment settings

  * Using `env-update`
* Updates **eix** package metadata

  * Using `eix-sync -0`
* Runs any custom updater scripts found in `/etc/genup/updaters.d` (a failing updater is fatal)
* Unmounts `/boot` (or remounts it read-only) if genup changed its state
* Reports final status, including pending config changes, a deprecated Portage profile, `glsa-check` results and unread `eselect news`

genup must be run as root, and refuses to start while another genup run holds `/run/genup.lock`. It can be run in non-interactive mode (the default) or interactive mode using the **--ask** option. Non-interactive mode is suitable for scripted execution, such as nightly **cron**(8) jobs. See `genup --help` or **genup**(8) for all options.

---

## Automation and Email

Example `crontab` and `logrotate` files are included. genup does not write a log file itself, so redirect its output to `/var/log/genup.log` as shown in the example crontab; error emails include the tail of that log.

Email notifications are enabled with `--email a@example.com,b@example.com --email-from host@example.com` (both are required). An email is sent on error, and on success only when there is something to report: a new or outdated kernel, pending config changes, a deprecated profile, GLSAs, unread news, or unused webapp installs.

---

## Installation

On Gentoo systems, **genup** is best installed via its ebuild. A maintained fork of the original sakaki- ebuild is available in the
[oubliette-overlay](https://github.com/nabbi/oubliette-overlay).

Additional background and installation context can be found in
[**Sakaki's EFI Install Guide**](https://wiki.gentoo.org/wiki/Sakaki's_EFI_Install_Guide) on the Gentoo Wiki.
