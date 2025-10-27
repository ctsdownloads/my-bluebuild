# Quick Start - 5 Minutes to Your Custom OS

The absolute fastest way to get your BlueBuild image building on GitHub.

## Prerequisites

- GitHub account
- Git installed
- cosign installed (`brew install cosign` on macOS, or see SETUP.md)

---

## 🚀 5-Step Setup

### 1. Generate Signing Keys (30 seconds)

```bash
cd /home/matt/bluebuild-workspace
cosign generate-key-pair
# Enter a password when prompted
# This creates: cosign.key and cosign.pub
```

### 2. Create GitHub Repository (1 minute)

```bash
# Via GitHub CLI (fastest)
gh repo create my-bluebuild --public --source=. --remote=origin --push

# OR manually at https://github.com/new
# Then:
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/USERNAME/my-bluebuild.git
git branch -M main
git push -u origin main
```

### 3. Add Signing Secret (1 minute)

```bash
# Copy your private key
cat cosign.key
# Copy the ENTIRE output
```

Then:
1. Go to: https://github.com/USERNAME/my-bluebuild/settings/secrets/actions
2. Click **New repository secret**
3. Name: `SIGNING_SECRET`
4. Value: Paste the cosign.key contents
5. Click **Add secret**

### 4. Enable Actions (30 seconds)

1. Go to: https://github.com/USERNAME/my-bluebuild/actions
2. Click **"I understand my workflows, go ahead and enable them"**
3. Your first build starts automatically!

### 5. Make Image Public (After first build - 30 seconds)

1. Go to: https://github.com/USERNAME?tab=packages
2. Click your image → **Package settings**
3. Scroll to **Change visibility** → **Public**

---

## ✅ Done!

Your image is now:
- Building automatically daily at 06:00 UTC
- Available at: `ghcr.io/USERNAME/myimage:latest`
- Signed and verified
- Ready to use!

---

## 🎯 Use Your Image

### Rebase Existing System

```bash
rpm-ostree rebase ostree-unverified-registry:ghcr.io/USERNAME/myimage:latest
systemctl reboot
```

### Generate ISO

1. Go to: https://github.com/USERNAME/my-bluebuild/actions/workflows/release-iso.yml
2. Click **Run workflow** → **Run workflow**
3. Wait ~30 minutes
4. Download from Artifacts

---

## 🔧 Customize

Edit `recipes/myimage.yml`:

```yaml
modules:
  - type: dnf
    install:
      packages:
        - neovim      # Add your packages
        - fish
```

Commit and push:

```bash
git add recipes/myimage.yml
git commit -m "Add neovim"
git push
```

Image rebuilds automatically!

---

## 📚 Learn More

- Full setup: See `SETUP.md`
- Customization: See `README.md`
- Modules: https://blue-build.org/reference/modules/
- Discord: https://discord.gg/f8MUghd5PB

---

## 🆘 Quick Troubleshooting

**Build failed?**
- Check Actions logs for errors
- Verify SIGNING_SECRET is set correctly
- Check recipes/myimage.yml for syntax errors

**Can't pull image?**
- Make sure package is Public
- Check image name matches: `ghcr.io/USERNAME/myimage:latest`

**ISO generation failed?**
- Ensure image build completed first
- Check ISO workflow logs

Need help? Join the [BlueBuild Discord](https://discord.gg/f8MUghd5PB)!
