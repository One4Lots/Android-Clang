# Android Clang Prebuilt (linux-x86) — Release Bot 🤖

[![Android Clang Prebuilt Release Bot](https://github.com/One4Lots/Android-Clang/actions/workflows/android-clang-release.yml/badge.svg)](https://github.com/One4Lots/Android-Clang/actions/workflows/android-clang-release.yml)

An automated GitHub Actions bot that fetches **one** Android Clang toolchain
(currently **`clang-r584948b`**) from Google's official AOSP prebuilts repo,
packs it into a single compressed archive, and publishes it as a
**GitHub Release** — so you can download a ready-to-use Android Clang
without cloning the multi-gigabyte full prebuilts repository yourself.

- 🗓️ **Runs automatically once a week** — **Friday 00:00 Bangladesh time**
  (Asia/Dhaka, UTC+6; GitHub cron is UTC, so it fires Thursday 18:00 UTC)
- 🖱️ **Can also be run manually** anytime from the Actions tab, with options
- 🔁 **Never creates duplicates** — if that exact upstream snapshot is
  already released, the weekly run skips itself cleanly
- 📲 **Optional Telegram notification** for every new release, sent with
  **link preview disabled**

---

## ⬇️ For users: download & use

1. Go to the [**Releases**](../../releases) page and download:
   - `clang-r584948b-linux-x86_64-<commit>.tar.zst` — the toolchain
   - `clang-r584948b-linux-x86_64-<commit>.tar.zst.sha256` — checksum
2. Verify and extract:

```bash
# Verify integrity
sha256sum -c clang-r584948b-linux-x86_64-*.tar.zst.sha256

# Extract (tar + zstd preserves symlinks and executable permissions)
mkdir -p ~/android-clang
tar -I zstd -xvf clang-r584948b-linux-x86_64-*.tar.zst -C ~/android-clang

# Use it
export PATH="$HOME/android-clang/clang-r584948b/bin:$PATH"
clang --version

# Example: build an Android kernel with this Clang
#   make -j$(nproc) ARCH=arm64 LLVM=1 LLVM_IAS=1 CROSS_COMPILE=aarch64-linux-gnu-
```

> **No zstd?** `sudo apt install zstd` (Debian/Ubuntu), `sudo pacman -S zstd`
> (Arch), `brew install zstd` (macOS). Recent `tar` also works with
> `tar --zstd -xvf <file>`.

Every release note contains the exact **clang name, full `clang --version`
output, target triple, clang host, upstream branch + commit (with link),
release date, archive size and SHA256** — everything needed to know exactly
what you're downloading.

---

## ⚙️ How the bot works

Workflow: [`.github/workflows/android-clang-release.yml`](.github/workflows/android-clang-release.yml)

```text
schedule (weekly) ─┐
                   ├─► Resolve config ─► sparse partial clone of AOSP ─►
manual dispatch ───┘        │            git clone --no-checkout --depth=1
                            │              --filter=blob:none --single-branch
                            │              -b main-kernel <upstream>
                            │            git sparse-checkout set --cone clang-r584948b
                            │            git checkout main-kernel
                            ▼
              Verify sparse checkout (cone mode, branch, only that dir)
                            ▼
              Collect metadata (clang --version, host, upstream SHA)
                            ▼
              Verify toolchain (bin/clang runs, size/file count sane)
                            ▼
              Already released? ──yes──► skip (no duplicate, no spam)
                            │ no
                            ▼
              Compress: tar + zstd -19 ─► .tar.zst + .sha256
                            ▼
              Verify archive (sha256sum -c, zstd -t, contents listing)
                            ▼
              Create GitHub Release (detailed notes, tag = clang-<commit>)
                            ▼
              Telegram notification (optional, preview disabled)
```

### ✅ Verification checks

The bot **checks everything properly** before releasing — it fails the job
rather than publishing a broken toolchain if any check fails:

| Stage | Checks |
|---|---|
| Config | `clang_dir` / `branch` / zstd level are validated (no paths, no traversal) |
| Sparse checkout | `--cone` mode is actually enabled (`core.sparseCheckoutCone=true`), the sparse pattern list contains exactly the target dir, HEAD is on the requested branch, the remote is the real AOSP URL, and **no other `clang-*` directory** was checked out |
| Toolchain | `bin/clang` exists and is executable, `bin/`+`lib/` present, `clang++` resolves, size > 10 MB and > 20 files (a real prebuilt is far larger), broken symlinks reported, and the binary actually **runs** and reports a clang version + target triple |
| Archive | `sha256sum -c` passes, `zstd -t` stream integrity passes, and the tar listing has exactly one top-level directory containing `bin/clang` |
| Release | Archive, checksum and release-notes files must all exist and be non-empty before `gh release create` runs |

### 🗓️ Weekly schedule

**Friday 00:00 Bangladesh time** (Asia/Dhaka, UTC+6, no DST). GitHub
Actions cron only understands UTC, so the workflow uses
`cron: "0 18 * * 4"` — Thursday 18:00 UTC. (GitHub may delay scheduled runs
by a few minutes under load.)

### Why `tar.zst`? (compression choice)

| Format | Verdict |
|---|---|
| **tar + zstd (chosen, level 19, multi-threaded)** | ✅ Excellent compression ratio (much smaller than gzip/lz4), very fast decompression, and `tar` preserves symlinks/permissions/executable bits — which a clang toolchain relies on. Preinstalled on most modern distros. |
| zip | ❌ Unreliable symlink/permission handling for toolchains, worse ratio. |
| tar.gz | ⚠️ Compatible everywhere, but noticeably larger and slower than zstd. |
| tar.lz4 | ❌ Extremely fast, but poor ratio → much bigger downloads. |
| tar.xz | ⚠️ Ratio close to zstd, but far slower to compress *and* decompress. |

### Release tag & duplicate protection

Tags look like `clang-r584948b-0123abcd5678` — clang directory name +
upstream commit. That makes each release an exact, immutable upstream
snapshot. The weekly run compares against existing releases first, so it
only publishes when Google actually changes something (or when you force
it), instead of re-uploading the same toolchain every week.

---

## 🚀 Setup (repo owner)

### 1. Merge the workflow into `main` (required for the schedule)

GitHub only runs **scheduled** workflows from the repository's **default
branch**. Merging this bot into `main` activates the weekly run.
Manual runs work from the Actions tab as soon as the workflow is on `main` too.

> GitHub may pause scheduled workflows on forks after 60 days without
> activity — a manual run re-enables them.

### 2. (Optional) Telegram notifications

1. Talk to [@BotFather](https://t.me/BotFather) → `/newbot` → copy the
   **bot token**.
2. Get your **chat ID**:
   - Personal: message your new bot once, then open
     `https://api.telegram.org/bot<TOKEN>/getUpdates` and read
     `result[].message.chat.id`.
   - Group/channel: add the bot (as admin for channels), post a message,
     then use the same `getUpdates` trick. Channel IDs look like `-100…`.
3. Add two **repository secrets** under
   *Settings → Secrets and variables → Actions → New repository secret*:

   | Secret | Value |
   |---|---|
   | `TELEGRAM_BOT_TOKEN` | Token from BotFather |
   | `TELEGRAM_CHAT_ID` | Your chat / group / channel ID |

That's it. If the secrets are not set, the Telegram step is **skipped
quietly** and the release still succeeds. Messages are sent with
`disable_web_page_preview=true`, so the release/download links never
generate a preview card.

### 3. (Optional) Change the default clang version / branch

Without editing the workflow file, add **repository variables**
(*Settings → Secrets and variables → Actions → Variables*):

| Variable | Example | Default if unset |
|---|---|---|
| `CLANG_DIR` | `clang-6000000` | `clang-r584948b` |
| `CLANG_BRANCH` | `main-kernel` | `main-kernel` |

These are what the **weekly scheduled run** will use.

---

## 🖱️ Run it manually

**Actions tab → “Android Clang Prebuilt Release Bot” → Run workflow**

| Input | Default | Meaning |
|---|---|---|
| `clang_dir` | `clang-r584948b` | Upstream directory to fetch |
| `branch` | `main-kernel` | Upstream branch |
| `auto_detect_latest` | `false` | Ignore `clang_dir`; pick the newest `clang-*` directory on the branch |
| `force_release` | `false` | Delete and rebuild the release even if it already exists |
| `zstd_level` | `19` | Compression level (`22` = smallest/slowest) |

---

## 🛠️ Troubleshooting

| Symptom | Fix |
|---|---|
| Weekly run never happens | Workflow must be merged into the default branch (`main`); check the workflow isn't disabled under Actions settings. |
| `Directory 'clang-…' was not checked out` | That clang version doesn't exist on that branch. The failing log lists the available `clang-*` directories — pick one, or run manually with `auto_detect_latest=true`. |
| Release step says “already exists — skipping” | Working as intended (no upstream change). Re-run manually with `force_release=true` to rebuild. |
| No Telegram message | Check both secrets are set, the bot was messaged/added first, and the chat ID is correct (channels start with `-100`). The release itself is unaffected. |
| Extraction says zstd not found | Install `zstd` (see above) or use `tar --zstd -xvf …` with a recent GNU tar. |

---

## 📦 Source & disclaimer

- Upstream: <https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86>
  (branch `main-kernel`, directory `clang-r584948b` by default)
- Fetch method: `git clone --no-checkout --depth=1 --filter=blob:none --single-branch -b main-kernel …`,
  `git sparse-checkout set --cone clang-r584948b`, `git checkout main-kernel`
  *(`--cone` is passed explicitly and verified in CI — cone mode fetches only
  the named directory plus root-level files; there is no `--sparse` flag in
  current Git)*
- This is an **unofficial redistribution** of Google's Android prebuilt
  Clang for convenience. All rights to the toolchain belong to their
  respective owners; see the license files inside the archive.
