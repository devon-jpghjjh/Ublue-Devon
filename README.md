# Ublue-Devon

A [BlueBuild](https://blue-build.org/) recipe for a [Bazzite](https://bazzite.gg)-based gaming image with a tiling-WM twist.

## What's in it

Base image: `ghcr.io/ublue-os/bazzite:latest` (Bazzite stable, currently **Fedora 44**).

| App | Source |
|---|---|
| [Noctalia](https://docs.noctalia.dev/) shell | Fedora official repos (`noctalia`, native v5, no quickshell dependency) — installed with the Terra repos disabled in the dnf transaction, because Terra carries an older 5.0 beta and has repo priority |
| [Umbriel](https://docs.noctalia.dev/umbriel/) compositor | Terra repo (`umbriel-nightly` + `xdg-desktop-portal-umbriel`) — Terra ships with Bazzite but its repos are disabled; this image re-enables them permanently |
| [MangoWM](https://github.com/mangowm/mango) compositor | Terra repo (`mangowm`, 0.17.5 = current upstream; no COPR needed) |
| [Brave Origin](https://brave.com/origin/linux/) browser | Official Brave RPM repo (`brave-origin`) |
| [Faugus Launcher](https://github.com/Faugus/faugus-launcher) | Terra repo (`faugus-launcher` 2.4.2 = current upstream; the `umu-launcher` runtime already ships in Bazzite) |
| Ghostty terminal | Terra repo (`ghostty` — Terra has priority over Fedora in Bazzite's repo config) |
| Virtualization stack | Fedora repos: `libvirt`, `qemu-kvm`, `virt-manager`, `virt-install`, `virt-viewer`, `edk2-ovmf`, `swtpm`+`swtpm-tools`, `usbredir` |

After installing, SDDM will offer Plasma, Umbriel and Mango sessions; Noctalia runs inside the session.

## Virtualization

The image comes up as a working QEMU/KVM host:

- `libvirtd.service` is enabled and the default NAT network autostarts, so new VMs have connectivity immediately.
- A polkit rule (`files/system/usr/share/polkit-1/rules.d/50-libvirt.rules`) lets wheel users (Bazzite's default user) connect to `qemu:///system` without password prompts. Alternative: `sudo usermod -aG libvirt $USER` and log out/in.
- **USB passthru**: with the VM's SPICE console open, use virt-manager's *Virtual Machine → Redirect USB device* (via `usbredir`) to hot-plug host USB devices into the guest, or add a persistent *USB Host Device* in the VM's hardware details. No IOMMU needed for USB.
- `edk2-ovmf` (UEFI) and `swtpm` (vTPM) are included, so Windows 11 guests work out of the box.
- PCI/GPU passthru is **not** preconfigured — that would need an `iommu=pt` kernel arg and VFIO setup; ask if you want it later.
- Optional extras not baked in: `guestfs-tools` (virt-customize etc.) if you end up scripting VM images.

## Building

- **GitHub Actions**: `BlueBuild` workflow builds `recipe.yml` on every push to `main` (plus daily) and pushes to `ghcr.io/<owner>/devon-bazzite`. The `Build ISO` workflow turns that image into an Anaconda ISO via [jasonn3/build-container-installer](https://github.com/jasonn3/build-container-installer) and uploads it as a job artifact. Both can be run manually via *Run workflow*.
- **Locally**: `bluebuild build ./recipes/recipe.yml`

### Required setup before the first run

1. Nothing needed for registry auth — the BlueBuild action logs into ghcr.io with the job's `GITHUB_TOKEN`.
2. Optional image signing: generate a key pair (`cosign generate-key-pair`), set a repo secret `SIGNING_SECRET` with the contents of `cosign.key`, commit `cosign.pub` to the repo root, and add `cosign_private_key: ${{ secrets.SIGNING_SECRET }}` to `.github/workflows/bluebuild.yml` ([how-to](https://blue-build.org/how-to/cosign/)). Until then the image ships unsigned and the ISO workflow passes `image_signed: false` accordingly.
3. After the first successful image build, set the new GHCR package to **public**, otherwise the ISO job can't pull it.

## Finding Bazzite's preinstalled apps (for deciding what to exclude)

- **RPMs** are installed inline in the [Containerfile](https://github.com/ublue-os/bazzite/blob/main/Containerfile) (big `dnf5 install` lists), and Bazzite's own removals live in [`build_files/global-remove`](https://github.com/ublue-os/bazzite/blob/main/build_files/global-remove).
- **Flatpaks** are *not* baked into the image — Bazzite's own ISO pipeline installs the set from [`installer/kde_flatpaks/flatpaks`](https://github.com/ublue-os/bazzite/blob/main/installer/kde_flatpaks/flatpaks) (Firefox, Gwenview, Okular, Kcalc, Haruna, Flatseal, protontricks, Warehouse, ProtonPlus, MangoHud/vkBasalt runtimes, …). Our ISO does the same via `iso/flatpak-refs.txt`, which mirrors that list plus the kinoite extras (ProtonUp-Qt, DistroShelf, Gearlever, filelight); `bazzite-flatpak-manager.service` only manages remotes/overrides at first boot, it no longer installs the list.
- Quickest ground truth for the RPM list: `podman run --rm ghcr.io/ublue-os/bazzite:latest rpm -qa | sort`

Once you know what you don't want: RPMs go under a `remove:` list in `recipe.yml`; Flatpaks — just strike their lines in `iso/flatpak-refs.txt` and rebuild the ISO.

## Maintenance notes

- `image-version: latest` tracks Bazzite stable. When Bazzite rebases (e.g. to F45), bump `version` in `build-iso.yml`; Terra tracks `$releasever` automatically and no COPRs are used anymore, so no chroot checks needed. Noctalia keeps coming from Fedora official.
- Noctalia v5 has no Quickshell dependency; if you ever want the git snapshot instead of the Fedora release, COPR `lionheartp/Hyprland` ships `noctalia-git`.
