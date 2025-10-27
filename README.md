# My Custom OS Image

A custom Fedora Atomic Desktop built with [BlueBuild](https://blue-build.org/), based on Universal Blue's Silverblue.

## What This Is

This repository automatically builds a customized, immutable Fedora Linux desktop with:
- **Base:** Fedora Silverblue 42 (GNOME desktop)
- **Custom packages:** micro, starship, htop, vim
- **Flatpak apps:** Firefox, GNOME apps
- **Custom configurations:** Enhanced bash prompt, helpful aliases
- **Bootable ISOs:** Automatically generated for easy installation

## Features

- 🔄 **Automated Daily Builds:** Image rebuilds daily with latest updates
- 📦 **Immutable & Atomic:** Uses rpm-ostree for reliable updates and rollbacks
- 🔐 **Signed Images:** Cryptographically signed for verification
- 💿 **ISO Generation:** Create bootable installation media on-demand
- 🎨 **Fully Customizable:** Edit `recipes/myimage.yml` to add/remove software

---

## Quick Start

### 1. Fork This Repository

Click the **Fork** button at the top right of this page to create your own copy.

### 2. Enable GitHub Actions

1. Go to your forked repository on GitHub
2. Click **Actions** tab
3. Click **"I understand my workflows, go ahead and enable them"**

### 3. Set Up Signing Keys

Image signing is required for secure updates. Generate keys locally:

```bash
# Generate a cosign key pair (you'll be prompted for a password)
cosign generate-key-pair

# This creates:
# - cosign.key (private key - keep this SECRET!)
# - cosign.pub (public key - can be shared)
```

**Add the private key to GitHub Secrets:**
1. Copy the contents of `cosign.key` (including the `-----BEGIN` and `-----END` lines)
2. Go to your repo: **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Name: `SIGNING_SECRET`
5. Value: Paste the entire contents of `cosign.key`
6. Click **Add secret**

> ⚠️ **Important:** Never commit `cosign.key` to your repository! Keep it secure offline.

### 4. Customize Your Image (Optional)

Edit `recipes/myimage.yml` to customize your OS:

```yaml
modules:
  - type: dnf
    install:
      packages:
        - micro
        - neovim        # Add more packages
        - fish          # Try a different shell
```

See [available modules](https://blue-build.org/reference/modules/) for all customization options.

### 5. Push Changes to Build

```bash
git add .
git commit -m "Initial setup"
git push origin main
```

GitHub Actions will automatically build your image and push it to `ghcr.io`.

---

## Using Your Custom Image

### Option A: Rebase an Existing System

If you already have Fedora Silverblue/Kinoite/Universal Blue installed:

```bash
# Get your image URL (replace USERNAME with your GitHub username)
IMAGE="ghcr.io/USERNAME/myimage:latest"

# Rebase to your custom image
rpm-ostree rebase ostree-unverified-registry:$IMAGE

# Reboot to apply
systemctl reboot
```

### Option B: Fresh Installation with ISO

#### Generate an ISO

**Method 1: Manual Trigger (Fastest)**
1. Go to **Actions** → **Generate ISO**
2. Click **Run workflow** → **Run workflow**
3. Wait ~20-30 minutes for the build
4. Download the ISO from the **Artifacts** section

**Method 2: Create a Release Tag**
```bash
git tag v1.0
git push origin v1.0
```
The ISO will be automatically generated and attached to the GitHub Release.

#### Write ISO to USB

**Linux:**
```bash
sudo dd if=myimage-kinoite-20250127.iso of=/dev/sdX bs=4M status=progress
sync
```

**Windows/macOS:**
- Use [Balena Etcher](https://www.balena.io/etcher/)
- Or [Ventoy](https://www.ventoy.net/)

#### Install from USB
1. Boot from the USB drive
2. Follow the Fedora installer (Anaconda)
3. Your custom image will be installed with all your customizations

---

## How It Works

### Build Process

```
recipes/myimage.yml (your config)
         ↓
BlueBuild CLI validates & generates Containerfile
         ↓
Modules execute (dnf, files, flatpaks, etc.)
         ↓
Container image built with Podman/Docker
         ↓
Image signed with cosign
         ↓
Pushed to ghcr.io/USERNAME/myimage:latest
         ↓
[Optional] ISO generated from image
```

### GitHub Actions Workflows

1. **`build.yml`** - Runs daily at 06:00 UTC and on every push
   - Validates your recipe
   - Builds the container image
   - Signs it with your cosign key
   - Pushes to GitHub Container Registry

2. **`release-iso.yml`** - Runs on tags or manual trigger
   - Pulls your built image from ghcr.io
   - Wraps it in a Fedora Anaconda installer
   - Uploads bootable ISO to GitHub Releases or Artifacts

### Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── build.yml           # Daily image builds
│       └── release-iso.yml     # ISO generation
├── recipes/
│   └── myimage.yml             # Your OS configuration
├── files/
│   └── system/                 # Custom files to include
│       └── etc/
│           └── skel/
│               └── .bashrc_custom
└── README.md
```

---

## Customization Guide

### Adding Software

**RPM Packages (via DNF):**
```yaml
- type: dnf
  install:
    packages:
      - neovim
      - fish
      - kubectl
```

**Flatpak Apps:**
```yaml
- type: default-flatpaks
  configurations:
    - scope: system
      install:
        - com.visualstudio.code
        - org.gimp.GIMP
```

**COPR Repositories:**
```yaml
- type: dnf
  repos:
    copr:
      - atim/starship
      - user/repo-name
  install:
    packages:
      - starship
```

### Adding Custom Files

Place files in `files/system/` - they'll be copied to `/` in the image:

```
files/system/
├── etc/
│   ├── skel/.bashrc_custom      # User home directory template
│   └── profile.d/custom.sh      # System-wide environment
└── usr/
    └── local/
        └── bin/
            └── myscript.sh       # Custom scripts
```

### Changing Base Image

Edit `recipes/myimage.yml`:

```yaml
base-image: ghcr.io/ublue-os/kinoite-main  # KDE Plasma desktop
# or
base-image: ghcr.io/ublue-os/bazzite       # Gaming-focused
# or
base-image: ghcr.io/ublue-os/bluefin       # Developer-focused
```

See [Universal Blue images](https://universal-blue.org/images/) for all options.

### Multiple Images

Create multiple recipes and add them to `.github/workflows/build.yml`:

```yaml
matrix:
  recipe:
    - myimage.yml
    - gaming.yml
    - workstation.yml
```

---

## Maintenance & Updates

### Your Image Updates
- **Automatic:** GitHub Actions rebuilds daily at 06:00 UTC
- **Manual:** Go to **Actions** → **Build Custom OS Image** → **Run workflow**

### System Updates
On your installed system:

```bash
# Check for updates
rpm-ostree update --check

# Update to latest build
rpm-ostree update

# Reboot to apply
systemctl reboot

# Rollback if needed
rpm-ostree rollback
```

### Monitoring Builds
- Go to **Actions** tab to see build status
- Failed builds will show errors in the logs
- Image size and build time visible in each run

---

## Troubleshooting

### Build Failures

**"Recipe validation failed"**
- Check YAML syntax in `recipes/myimage.yml`
- Ensure indentation is correct (use spaces, not tabs)
- Validate at https://blue-build.org/

**"Package not found"**
- Verify package name with `dnf search <package>`
- Check if it requires a COPR repo
- Some packages only exist as Flatpaks

**"Permission denied" or "Out of space"**
- Enable `maximize_build_space: true` in `build.yml` (already set)
- Remove unnecessary packages to reduce image size

### Rebase Issues

**"Signature verification failed"**
```bash
# Use unverified registry temporarily
rpm-ostree rebase ostree-unverified-registry:ghcr.io/USERNAME/myimage:latest
```

**"Error pulling image"**
- Check if the image exists: https://github.com/USERNAME?tab=packages
- Ensure the package is set to **Public** (Settings → Package settings → Visibility)

### ISO Generation Issues

**"ISO workflow failed"**
- Ensure the image build completed successfully first
- Check that your image is public on ghcr.io
- Review ISO generation logs in Actions tab

**"ISO is too large"**
- Remove unnecessary packages from your recipe
- Consider using fewer/smaller Flatpaks

---

## Local Development

### Build Locally

Install BlueBuild CLI:
```bash
# Download from https://github.com/blue-build/cli/releases
# Or use Docker:
docker run --rm -it \
  -v $(pwd):/build \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  ghcr.io/blue-build/cli:latest \
  build recipes/myimage.yml
```

### Generate ISO Locally

```bash
# From your built image
sudo bluebuild generate-iso recipe recipes/myimage.yml \
  --output-dir ./iso-output \
  --variant kinoite

# From remote image
sudo bluebuild generate-iso image ghcr.io/USERNAME/myimage:latest \
  --output-dir ./iso-output
```

### Test in Virtual Machine

```bash
# Using virt-manager or QEMU
qemu-system-x86_64 \
  -m 4096 \
  -cpu host \
  -enable-kvm \
  -cdrom myimage-kinoite.iso
```

---

## Resources

- **BlueBuild Docs:** https://blue-build.org/
- **Module Reference:** https://blue-build.org/reference/modules/
- **Universal Blue:** https://universal-blue.org/
- **BlueBuild Discord:** https://discord.gg/f8MUghd5PB
- **BlueBuild GitHub:** https://github.com/blue-build

## Contributing

Contributions are welcome! Feel free to:
- Open issues for bugs or feature requests
- Submit pull requests with improvements
- Share your custom modules and configurations

---

## License

This project is licensed under the Apache License 2.0. See the base BlueBuild project for full license details.

---

## Credits

Built with [BlueBuild](https://blue-build.org/) - A declarative build system for custom OS images.

Based on [Universal Blue](https://universal-blue.org/) - A community project creating Fedora Atomic Desktop images.
# Updated Mon Oct 27 03:10:20 PM PDT 2025
