# BlueBuild Architecture & Code Flow

Understanding how this repository builds your custom OS.

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    YOUR GITHUB REPOSITORY                        │
│                                                                  │
│  recipes/myimage.yml  ←  Your OS configuration                  │
│  files/system/        ←  Custom files to include                │
│  .github/workflows/   ←  Automation                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    [Push to GitHub]
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    GITHUB ACTIONS (CI/CD)                        │
│                                                                  │
│  Workflow: build.yml                                            │
│  ├─ Checkout repository                                         │
│  ├─ Install BlueBuild CLI (v0.9)                               │
│  ├─ Setup Podman/Docker                                         │
│  ├─ Build image from recipe                                     │
│  ├─ Sign with cosign                                            │
│  └─ Push to ghcr.io                                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              BLUEBUILD CLI (Build Engine)                        │
│                                                                  │
│  Location: ghcr.io/blue-build/cli:v0.9                         │
│  Source: /home/matt/bluebuild-full/cli/                        │
│                                                                  │
│  Build Process:                                                  │
│  1. Parse recipe (blue_build_recipe crate)                      │
│  2. Validate schema (TypeSpec → JSON Schema)                    │
│  3. Generate Containerfile (blue_build_template)                │
│  4. Execute modules in order                                     │
│  5. Build container image                                        │
│  6. Sign with cosign (if --push)                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    MODULE EXECUTION                              │
│                                                                  │
│  Source: /home/matt/bluebuild-full/modules/                    │
│                                                                  │
│  Your recipe modules run in order:                              │
│  ├─ files     → Copy files/system/* to image                   │
│  ├─ dnf       → Install/remove RPM packages                     │
│  ├─ default-flatpaks → Configure Flatpak repos & apps          │
│  └─ signing   → Setup image signing policies                    │
│                                                                  │
│  Each module:                                                    │
│  • Runs in Nushell or Bash                                      │
│  • Has full root access to image                                │
│  • Can modify any system files                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  GITHUB CONTAINER REGISTRY                       │
│                                                                  │
│  Image: ghcr.io/USERNAME/myimage:latest                        │
│  ├─ Signed with cosign                                          │
│  ├─ Tagged: latest, 42, gts, etc.                              │
│  └─ Public (after you change visibility)                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                     [Manual Trigger or Tag]
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              ISO GENERATION (release-iso.yml)                    │
│                                                                  │
│  Workflow: release-iso.yml                                      │
│  ├─ Pull image from ghcr.io                                    │
│  ├─ Run JasonN3's build-container-installer                    │
│  │   └─ Wraps image in Fedora Anaconda installer              │
│  ├─ Generate ISO file                                           │
│  ├─ Create SHA256 checksum                                      │
│  └─ Upload to GitHub Release or Artifacts                       │
│                                                                  │
│  Tool: ghcr.io/jasonn3/build-container-installer               │
│  Variants: kinoite, silverblue, server                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    BOOTABLE ISO OUTPUT                           │
│                                                                  │
│  myimage-kinoite-20250127.iso                                  │
│  myimage-kinoite-20250127.iso.sha256sum                       │
│                                                                  │
│  Can be:                                                        │
│  • Written to USB drive                                         │
│  • Used for fresh installations                                 │
│  • Booted in VMs                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Flow: Build Process

### 1. Recipe Parsing

**File:** `/home/matt/bluebuild-full/cli/src/commands/build.rs:151`

```
BuildCommand::try_run()
├─ Initialize driver (Podman/Docker/Buildah)
├─ Setup credentials
├─ Find recipe files (./recipes/ or ./config/)
├─ Generate Containerfile for each recipe
└─ Call start() for parallel builds
```

### 2. Containerfile Generation

**File:** `/home/matt/bluebuild-full/cli/src/commands/generate.rs`

```
GenerateCommand::try_run()
├─ Parse YAML recipe
├─ Validate against schema
├─ Load module definitions
├─ Template Containerfile with:
│   ├─ Base image (FROM ghcr.io/ublue-os/silverblue-main:42)
│   ├─ Module COPY and RUN statements
│   └─ Labels and metadata
└─ Write to /tmp/bluebuild-*/Containerfile
```

### 3. Image Building

**File:** `/home/matt/bluebuild-full/cli/src/commands/build.rs:249`

```
BuildCommand::build()
├─ Generate tags (latest, 42, gts, etc.)
├─ Build image with selected driver
│   ├─ podman build
│   ├─ OR docker buildx build
│   ├─ OR buildah bud
│   └─ With layer caching, secrets, squashing
├─ Rechunk (optional - optimize layer sizes)
├─ Push to registry (if --push)
└─ Sign with cosign (if --push && !--no-sign)
```

### 4. Module Execution (During Build)

**Modules Source:** `/home/matt/bluebuild-full/modules/modules/`

Each module runs during the image build:

```dockerfile
# Generated Containerfile snippet for 'dnf' module
COPY --from=ghcr.io/blue-build/modules/dnf:latest /module /tmp/modules/dnf
RUN --mount=type=cache,target=/var/cache/dnf \
    /tmp/modules/dnf/dnf.nu install micro starship htop vim
RUN dnf remove -y firefox firefox-langpacks
```

**Module Script:** `/home/matt/bluebuild-full/modules/modules/dnf/dnf.nu`
- Reads module config
- Adds COPR repos
- Installs packages with DNF
- Removes packages
- Cleans cache

### 5. Signing

**File:** `/home/matt/bluebuild-full/cli/src/commands/build.rs:323`

```
Driver::sign_and_verify()
├─ Get image digest
├─ cosign sign --key cosign.key ghcr.io/USERNAME/myimage@sha256:...
├─ Verify signature
└─ Push signature to registry
```

---

## Code Flow: ISO Generation

### ISO Generation Entry Point

**File:** `/home/matt/bluebuild-full/cli/src/commands/generate_iso.rs:138`

```
GenerateIsoCommand::try_run()
├─ Initialize driver
├─ Create temp directory for build
├─ Determine output directory
├─ IF recipe mode:
│   └─ BuildCommand::build() with --archive
│       └─ Creates OCI tarball in temp dir
├─ ELSE (image mode):
│   └─ Use existing image from registry
└─ Call build_iso()
```

### ISO Builder Invocation

**File:** `/home/matt/bluebuild-full/cli/src/commands/generate_iso.rs:187`

```
GenerateIsoCommand::build_iso()
├─ Prepare environment variables:
│   ├─ VARIANT=kinoite
│   ├─ ISO_NAME=build/myimage.iso
│   ├─ SECURE_BOOT_KEY_URL=...
│   ├─ IMAGE_SRC=oci-archive:/img_src/myimage.oci-archive (recipe mode)
│   └─ OR IMAGE_NAME/IMAGE_REPO/IMAGE_TAG (image mode)
├─ Mount volumes:
│   ├─ output_dir → /build-container-installer/build
│   ├─ dnf-cache → /cache/dnf
│   └─ image_out_dir → /img_src (recipe mode only)
└─ Run container:
    └─ podman run --privileged \
           -v ./iso-output:/build-container-installer/build \
           ghcr.io/jasonn3/build-container-installer
```

### Inside build-container-installer

**Tool:** JasonN3's build-container-installer container

```
1. Load OCI image or pull from registry
2. Extract ostree commit from image
3. Create Anaconda installer environment
4. Configure installer variant (kinoite/silverblue/server)
5. Add secure boot keys
6. Generate ISO with:
   ├─ Kernel and initramfs
   ├─ Anaconda installer
   ├─ Your ostree commit
   └─ Boot menu configuration
7. Write ISO to mounted output directory
```

---

## GitHub Actions Integration

### build.yml Workflow

```yaml
Steps:
1. actions/checkout           # Get your repository
2. free-disk-space           # Clean up disk (optional)
3. docker/setup-buildx       # Setup Docker Buildx
4. cosign-installer          # Install cosign for signing
5. blue-build/github-action  # The main build
   ├─ Installs BlueBuild CLI from ghcr.io/blue-build/cli
   ├─ Runs: bluebuild build --push recipes/myimage.yml
   └─ Uses SIGNING_SECRET from GitHub Secrets
```

### release-iso.yml Workflow

```yaml
Steps:
1. actions/checkout           # Get your repository
2. Install BlueBuild CLI      # From Docker image
3. Generate ISO               # bluebuild generate-iso image ...
4. Generate Checksum          # sha256sum
5. Upload to Release          # If triggered by tag
   OR Upload Artifact         # If manual trigger
```

---

## Key Source Files Reference

### CLI Commands
- **Build:** `/home/matt/bluebuild-full/cli/src/commands/build.rs`
- **ISO:** `/home/matt/bluebuild-full/cli/src/commands/generate_iso.rs`
- **Generate:** `/home/matt/bluebuild-full/cli/src/commands/generate.rs`
- **Main:** `/home/matt/bluebuild-full/cli/src/bin/bluebuild.rs`

### Drivers (Container Runtimes)
- **Podman:** `/home/matt/bluebuild-full/cli/process/drivers/podman_driver.rs`
- **Docker:** `/home/matt/bluebuild-full/cli/process/drivers/docker_driver.rs`
- **Buildah:** `/home/matt/bluebuild-full/cli/process/drivers/buildah_driver.rs`

### Modules
- **All modules:** `/home/matt/bluebuild-full/modules/modules/`
- **DNF:** `/home/matt/bluebuild-full/modules/modules/dnf/dnf.nu`
- **Files:** `/home/matt/bluebuild-full/modules/modules/files/files.sh`
- **Flatpaks:** `/home/matt/bluebuild-full/modules/modules/default-flatpaks/`

### GitHub Action
- **Action:** `/home/matt/bluebuild-full/github-action/action.yml`

### Template (This Repo Based On)
- **Recipe:** `/home/matt/bluebuild-full/template/recipes/recipe.yml`
- **Workflow:** `/home/matt/bluebuild-full/template/.github/workflows/build.yml`

---

## Understanding Your Repository

### Your Recipe Format

```yaml
name: myimage                          # Image name (becomes ghcr.io/user/myimage)
base-image: ghcr.io/ublue-os/silverblue-main  # FROM in Dockerfile
image-version: 42                      # Tag of base image

modules:                               # Executed in order during build
  - type: files                        # Module name
    files:                            # Module config
      - source: system
        destination: /
```

### Module Execution Order

Modules run **sequentially** in the order listed. This matters for:
- Installing dependencies before software that needs them
- Removing packages before installing replacements
- Setting up repos before installing from them

**Example:**
```yaml
modules:
  - type: dnf           # First: Add repos and install packages
    repos:
      copr:
        - user/repo
    install:
      packages:
        - some-package

  - type: files         # Second: Copy config files
    files:
      - source: system
        destination: /

  - type: systemd       # Third: Enable services (needs files to exist)
    system:
      enabled:
        - my-service.service
```

---

## Build Time Expectations

**Image Build:** 10-20 minutes
- Base image pull: 2-5 min
- Module execution: 5-10 min
- Image push: 2-5 min

**ISO Generation:** 20-30 minutes
- Image pull: 2-3 min
- Installer creation: 15-25 min
- ISO upload: 1-2 min

**Factors affecting time:**
- Number of packages
- Whether packages are cached
- Network speed to registries
- GitHub runner availability

---

## Security Model

### Image Signing

```
Build → Sign with cosign.key → Push signature to registry
                                        ↓
User pulls image ← Verify with cosign.pub ← Pull signature
```

**Your Setup:**
- Private key: In `SIGNING_SECRET` (GitHub Secret)
- Public key: `cosign.pub` (in your repo, can be public)
- Signature: Stored in ghcr.io alongside image

### Signature Verification

```bash
# User verifies your image
cosign verify --key cosign.pub ghcr.io/USERNAME/myimage:latest

# If valid, shows signature information
# If invalid or missing, errors
```

---

## Customization Points

### 1. Recipe (recipes/myimage.yml)
- Change base image
- Add/remove packages
- Configure modules
- Set image metadata

### 2. Files (files/system/)
- Add system files
- User home templates (/etc/skel)
- System configs
- Scripts

### 3. Workflows (.github/workflows/)
- Build schedule (cron)
- ISO variants
- Build options
- Trigger conditions

### 4. Multiple Images
Add to build.yml:
```yaml
matrix:
  recipe:
    - desktop.yml
    - gaming.yml
    - server.yml
```

---

## Related Documentation

- **User Docs:** `README.md` - How to use this repository
- **Setup Guide:** `SETUP.md` - Step-by-step GitHub setup
- **Quick Start:** `QUICKSTART.md` - 5-minute setup
- **This File:** Understanding the architecture

---

## Learning More

To understand the code deeper:
1. Read `/home/matt/bluebuild-full/cli/src/commands/build.rs`
2. Check `/home/matt/bluebuild-full/cli/src/commands/generate_iso.rs`
3. Explore modules in `/home/matt/bluebuild-full/modules/modules/`
4. Review GitHub Action at `/home/matt/bluebuild-full/github-action/action.yml`

The entire BlueBuild ecosystem is in `/home/matt/bluebuild-full/` for your reference!
