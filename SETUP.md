# Complete Setup Instructions

This guide walks you through setting up your BlueBuild repository on GitHub from scratch.

## Prerequisites

- GitHub account
- Git installed locally
- `cosign` installed (for signing keys)

## Step-by-Step Setup

### 1. Install cosign (If Not Already Installed)

**Linux:**
```bash
# Download and install
LATEST=$(curl -s https://api.github.com/repos/sigstore/cosign/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -Lo cosign https://github.com/sigstore/cosign/releases/download/${LATEST}/cosign-linux-amd64
chmod +x cosign
sudo mv cosign /usr/local/bin/
```

**macOS:**
```bash
brew install cosign
```

**Windows:**
Download from: https://github.com/sigstore/cosign/releases

### 2. Generate Signing Keys

```bash
cd /home/matt/bluebuild-workspace

# Generate key pair (you'll be prompted for a password)
cosign generate-key-pair

# This creates:
# - cosign.key (PRIVATE - keep secret!)
# - cosign.pub (PUBLIC - can share)
```

**Save your cosign.key contents:**
```bash
cat cosign.key
```

Copy the entire output (including `-----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----` and `-----END ENCRYPTED COSIGN PRIVATE KEY-----`).

### 3. Create GitHub Repository

**Option A: Via GitHub Web Interface**
1. Go to https://github.com/new
2. Repository name: `my-bluebuild` (or any name)
3. Description: "My custom Fedora Atomic Desktop"
4. Public or Private (recommend Public for GHCR)
5. Click **Create repository**

**Option B: Via GitHub CLI**
```bash
gh repo create my-bluebuild --public --source=. --remote=origin
```

### 4. Add Signing Secret to GitHub

1. Go to your repository on GitHub
2. Click **Settings** (top right)
3. In left sidebar: **Secrets and variables** → **Actions**
4. Click **New repository secret**
5. Name: `SIGNING_SECRET`
6. Value: Paste the entire contents of `cosign.key` from step 2
7. Click **Add secret**

### 5. Make GitHub Package Public

After your first image build:

1. Go to your GitHub profile
2. Click **Packages** tab
3. Find your image package (e.g., `myimage`)
4. Click on it → **Package settings** (right sidebar)
5. Scroll to **Danger Zone**
6. Click **Change visibility** → **Public**
7. Confirm

This allows anyone to pull and rebase to your image.

### 6. Push to GitHub

```bash
cd /home/matt/bluebuild-workspace

# Initialize git (if not already done)
git init

# Add all files
git add .

# First commit
git commit -m "Initial BlueBuild setup"

# Add remote (replace USERNAME with your GitHub username)
git remote add origin https://github.com/USERNAME/my-bluebuild.git

# Push to GitHub
git branch -M main
git push -u origin main
```

### 7. Enable GitHub Actions

1. Go to your repository on GitHub
2. Click **Actions** tab
3. Click **"I understand my workflows, go ahead and enable them"**
4. The first build will start automatically

### 8. Monitor First Build

1. Go to **Actions** tab
2. Click on the running workflow
3. Watch the build progress (takes ~15-20 minutes)
4. Check for any errors in the logs

### 9. Generate Your First ISO

**Option A: Manual Workflow Trigger**
1. Go to **Actions** → **Generate ISO**
2. Click **Run workflow** dropdown
3. Click green **Run workflow** button
4. Wait ~20-30 minutes
5. Download from **Artifacts** section

**Option B: Create a Release**
```bash
git tag v1.0
git push origin v1.0
```
ISO will be attached to the GitHub Release automatically.

### 10. Use Your Custom Image

**On an existing Fedora Silverblue/Kinoite system:**
```bash
# Replace USERNAME with your GitHub username
rpm-ostree rebase ostree-unverified-registry:ghcr.io/USERNAME/myimage:latest

# Reboot
systemctl reboot
```

**Fresh install:**
1. Download the ISO from Releases or Artifacts
2. Write to USB with `dd`, Balena Etcher, or Ventoy
3. Boot and install

---

## Customizing Your Image

### Edit Recipe

```bash
nano recipes/myimage.yml
```

Add/remove packages, change base image, add modules, etc.

### Commit and Push

```bash
git add recipes/myimage.yml
git commit -m "Add neovim and fish shell"
git push
```

GitHub Actions will automatically rebuild your image.

---

## Verification

### Check Image Was Pushed

1. Go to https://github.com/USERNAME?tab=packages
2. You should see `myimage` package
3. Click it to see tags and details

### Check Image on Command Line

```bash
# Pull your image (replace USERNAME)
podman pull ghcr.io/USERNAME/myimage:latest

# Inspect it
podman inspect ghcr.io/USERNAME/myimage:latest
```

### Verify Signature

```bash
# Install cosign
# Download your public key from the repo
curl -O https://raw.githubusercontent.com/USERNAME/my-bluebuild/main/cosign.pub

# Verify signature (replace USERNAME)
cosign verify --key cosign.pub ghcr.io/USERNAME/myimage:latest
```

---

## Common Issues

### "No cosign.key file found"
- Make sure you ran `cosign generate-key-pair` in the workspace directory
- Check that `cosign.key` exists locally (but is gitignored)

### "SIGNING_SECRET not found"
- Verify you added the secret in GitHub repo settings
- Name must be exactly `SIGNING_SECRET`
- Value must be the full contents of `cosign.key`

### "Package is private"
- Go to GitHub Packages and change visibility to Public
- Required for others to pull your image

### "Workflows not running"
- Check that Actions are enabled in repo settings
- Verify workflows exist in `.github/workflows/`
- Check if branch name matches (should be `main`)

---

## Next Steps

- Read the [BlueBuild documentation](https://blue-build.org/)
- Explore [available modules](https://blue-build.org/reference/modules/)
- Join the [BlueBuild Discord](https://discord.gg/f8MUghd5PB)
- Check out [Universal Blue images](https://universal-blue.org/images/)
- Share your creation with the community!

---

## Security Notes

- **Never commit `cosign.key`** - it's already in `.gitignore`
- Keep your cosign password secure
- Store `cosign.key` in a safe place as backup
- Rotate keys periodically for production use
- Consider using hardware keys (YubiKey) for signing in production

---

## Troubleshooting Checklist

- [ ] `cosign.key` generated and saved
- [ ] SIGNING_SECRET added to GitHub repo secrets
- [ ] Repository pushed to GitHub
- [ ] GitHub Actions enabled
- [ ] First workflow completed successfully
- [ ] Package made public on GitHub
- [ ] ISO generated (if desired)
- [ ] Able to pull image with podman/docker

If all checkboxes are complete, you're ready to use your custom OS!
