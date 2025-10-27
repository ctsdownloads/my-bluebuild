# GitHub Setup Checklist

## ✅ Completed Steps

- [x] Signing keys generated (`cosign.key` and `cosign.pub`)
- [x] Git repository initialized
- [x] Initial commit created
- [x] Ready to push to GitHub

---

## 📋 Next Steps (Do These Now)

### Step 1: Save Your Private Key

Your signing key password: `bluebuild-temp-password`

**IMPORTANT:** Copy this entire private key (you'll paste it into GitHub in Step 3):

```
-----BEGIN ENCRYPTED SIGSTORE PRIVATE KEY-----
eyJrZGYiOnsibmFtZSI6InNjcnlwdCIsInBhcmFtcyI6eyJOIjo2NTUzNiwiciI6
OCwicCI6MX0sInNhbHQiOiI2ZDRHWHJOMFE5MkJqSVk5d2ZOMHMxTXV4YWtML3NC
bEFGWW9FaUV2eFdJPSJ9LCJjaXBoZXIiOnsibmFtZSI6Im5hY2wvc2VjcmV0Ym94
Iiwibm9uY2UiOiI3ZCtTWHd1T29GYXpQN2ZOU3ZoeWRGV2Q2TWI0cDZ5OCJ9LCJj
aXBoZXJ0ZXh0IjoiWk5kY090b3BPQjNsZ3BkeE1Ba1dEZlFYT0tLR1FvMVA3RGtJ
ZDArRU5VWkRwQ3lmVThoWnVXZ0hMclZ0OEI4RXNWNHBLTlh1dTJYZEcwV3VNR0tq
YUJwcU5PMy9VTzIyM1FKRlN1bmN2enlnTFJIQXZ3c2tZUzZNeGhVTjFyY2NER2JF
dnJiRWRHZndwZ3RoSllMa3hCNzhFdm0ybjV3NEJSemZzWmVUSnpPQSt0MWY2cDBE
M0t6UkgwQVUwKzl3L3lvb1dOejhwbVR2WHc9PSJ9
-----END ENCRYPTED SIGSTORE PRIVATE KEY-----
```

---

### Step 2: Create GitHub Repository

**Option A - Using GitHub CLI (Recommended):**
```bash
# Make sure you're in the workspace
cd /home/matt/bluebuild-workspace

# Create and push in one command
gh repo create my-bluebuild --public --source=. --remote=origin --push
```

**Option B - Manual via GitHub Web:**

1. Go to: https://github.com/new

2. Fill in:
   - Repository name: `my-bluebuild` (or your preferred name)
   - Description: `Custom Fedora Atomic Desktop with BlueBuild`
   - Visibility: **Public** (required for free GitHub Container Registry)
   - DO NOT initialize with README (we already have files)

3. Click **Create repository**

4. Then run these commands:
   ```bash
   cd /home/matt/bluebuild-workspace

   # Replace USERNAME with your GitHub username
   git remote add origin https://github.com/USERNAME/my-bluebuild.git
   git push -u origin main
   ```

---

### Step 3: Add SIGNING_SECRET to GitHub

**This is CRITICAL - without this, image builds will fail!**

1. Go to your repository on GitHub

2. Click **Settings** (top right of repo page)

3. In left sidebar: **Secrets and variables** → **Actions**

4. Click **New repository secret** (green button)

5. Fill in:
   - Name: `SIGNING_SECRET` (must be exactly this)
   - Value: Paste the ENTIRE private key from Step 1 above
     (including the BEGIN and END lines)

6. Click **Add secret**

---

### Step 4: Enable GitHub Actions

1. Go to your repository on GitHub

2. Click **Actions** tab (top menu)

3. You'll see a message about workflows
   - Click **"I understand my workflows, go ahead and enable them"**

4. The first build will start automatically!
   - Click on the workflow run to watch progress
   - Build takes ~15-20 minutes

---

### Step 5: Make Image Public (After First Build Completes)

1. Wait for first build to complete successfully

2. Go to your GitHub profile: `https://github.com/USERNAME`

3. Click **Packages** tab

4. Find your package (should be called `myimage`)

5. Click on it

6. On the right sidebar: **Package settings**

7. Scroll down to **Danger Zone**

8. Click **Change visibility** → Select **Public** → Confirm

---

## 🎉 You're Done!

After completing these steps:

✅ Your image will be at: `ghcr.io/USERNAME/myimage:latest`
✅ Builds happen automatically daily at 06:00 UTC
✅ You can generate ISOs anytime from GitHub Actions
✅ You can rebase systems to your custom image

---

## 🚀 Using Your Image

### Rebase an Existing System

```bash
# Replace USERNAME with your GitHub username
rpm-ostree rebase ostree-unverified-registry:ghcr.io/USERNAME/myimage:latest

# Reboot to switch to your image
systemctl reboot
```

### Generate an ISO

**Method 1 - Manual Trigger:**
1. Go to: `https://github.com/USERNAME/my-bluebuild/actions/workflows/release-iso.yml`
2. Click **Run workflow** (button on right)
3. Click green **Run workflow** button
4. Wait ~30 minutes
5. Download from **Artifacts** section

**Method 2 - Create Release:**
```bash
cd /home/matt/bluebuild-workspace
git tag v1.0
git push origin v1.0
```
ISO will appear in GitHub Releases automatically.

---

## 🔧 Customizing

Edit `recipes/myimage.yml` to add/remove packages:

```bash
cd /home/matt/bluebuild-workspace
nano recipes/myimage.yml

# Make your changes, then:
git add recipes/myimage.yml
git commit -m "Add neovim and fish shell"
git push
```

GitHub will rebuild automatically!

---

## 🆘 Troubleshooting

### Build fails with "cosign: error"
- Make sure you added SIGNING_SECRET correctly
- Verify the secret name is exactly `SIGNING_SECRET`
- Check that you pasted the entire key including BEGIN/END lines

### Can't push to GitHub
- Verify your GitHub username in the remote URL
- Check you have push access to the repository
- Try: `gh auth status` to check authentication

### Package not visible
- Make sure you changed package visibility to Public
- Wait a few minutes after first build completes

### Workflow not running
- Check that Actions are enabled (Settings → Actions → Allow all actions)
- Verify the workflow files are in `.github/workflows/`

---

## 📚 Next Reading

- `README.md` - Full documentation
- `QUICKSTART.md` - Quick reference
- `ARCHITECTURE.md` - How it all works
- BlueBuild Docs: https://blue-build.org/

---

## Important Files in This Repo

```
cosign.key          ← NEVER commit this! (already in .gitignore)
cosign.pub          ← Public key (safe to share)
recipes/myimage.yml ← Your OS configuration
.github/workflows/  ← Automation workflows
files/system/       ← Custom files to include
```

The `cosign.key` file is on your local machine only - it's not in git (protected by .gitignore).
