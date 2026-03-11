```skill
---
name: v2v-dev-loop
description: >
  Build virt-v2v from source, generate a dev Dockerfile that overlays
  custom-built virt-v2v-in-place onto the Fedora 44 base image, build
  and push the Docker image to docker.io/michaelbi/fedora-libguestfs.
  Automates the developer inner loop for testing virt-v2v code changes
  in the vm-migration container environment.
license: Apache-2.0
---

# v2v Dev Loop — Build, Containerize & Push Custom virt-v2v

## When to activate

Activate this skill when the user says any of:
- `/v2v-dev-loop`, "build and push v2v", "dev loop"
- "containerize virt-v2v", "test v2v changes in container"
- "build v2v image", "push v2v image"
- "package virt-v2v-in-place"

Do NOT activate for:
- Pure OCaml build without containerization (just run `make` directly)
- Production image builds or releases
- Changes to vm-migration-wrc blueprints or Go services

## Environment

| Item | Path / Value |
|------|-------------|
| virt-v2v source | `/root/dev/virt-v2v` |
| Base Dockerfile | `/root/dev/vm-migration-wrc/baseimage/Dockerfile` |
| Build context | `/root/dev/vm-migration-wrc` |
| Docker registry | `docker.io/michaelbi/fedora-libguestfs` |
| Built binary | `<virt-v2v-source>/in-place/virt-v2v-in-place` |
| Build log | `/tmp/v2v-build.log` |
| Temp Dockerfile | `/tmp/Dockerfile.v2v-dev` |

### Prerequisites

- Docker (or podman) with buildx support
- Fedora 43+ (matches CI matrix in `.github/workflows/main.yml`)
- Build deps installed via `dnf builddep -y virt-v2v` (pulls in the full
  OCaml toolchain, autoconf, automake, libtool, libguestfs-devel,
  libnbd-devel, and all other required packages automatically)
- Go managed via [gvm](https://github.com/moovweb/gvm): activate with
  `gvm use go1.25.5` (required for building the `mojav` Go binary in
  vm-migration-wrc)
- SSH key for GitHub: `eval $(ssh-agent) && ssh-add ~/.ssh/wr56`
  (required for `git push` to `git@github.com:Solutions-Innovation/vm-migration-wrc.git`)
- `docker login docker.io` completed for the michaelbi account (only for
  local Docker builds — not needed when using the GitHub Actions approach)

## Workflow

### Step 1: Gather configuration (interactive)

Use `ask_questions` to collect options. All questions in a single call:

**Question 1 — Pipeline stages** (multiSelect):
- Header: `Stages`
- Question: `Which stages should run?`
- Options:
  - `Build from source` — compile virt-v2v OCaml code (recommended)
  - `Build Docker image` — generate dev Dockerfile + build image (recommended)
  - `Push to registry` — tag + push to docker.io/michaelbi (recommended)
  - `Verify image` — run smoke test on built image
- Default: all selected

**Question 2 — Image tag** (freeform):
- Header: `Image tag`
- Question: `Docker image tag? Leave empty for auto-generated.`
- Options:
  - Auto: `dev-<branch>-<short-sha>-<YYYYMMDD>` (recommended)
  - `latest-dev`
- Auto-generate if empty using:
  ```bash
  BRANCH=$(cd /root/dev/virt-v2v && git rev-parse --abbrev-ref HEAD | sed 's|/|-|g')
  SHA=$(cd /root/dev/virt-v2v && git rev-parse --short HEAD)
  DATE=$(date +%Y%m%d)
  TAG="dev-${BRANCH}-${SHA}-${DATE}"
  ```

**Question 3 — Build mode** (select):
- Header: `Build mode`
- Question: `Full or incremental build?`
- Options:
  - `Incremental` — run `make` only (assumes configure already done) (recommended)
  - `Full` — run `autoreconf -i && ./configure && make` from scratch

**Question 4 — vm-migration-wrc ref** (freeform):
- Header: `vm-migration-wrc ref`
- Question: `Which tag or branch to checkout in vm-migration-wrc?`
- Show available tags/branches to help the user choose:
  ```bash
  cd /root/dev/vm-migration-wrc
  git tag --sort=-creatordate | head -10
  git branch -a | head -10
  ```
- Options:
  - Current HEAD (recommended) — keep whatever is checked out
  - User-specified tag or branch name
- If the user specifies a ref, checkout before building:
  ```bash
  cd /root/dev/vm-migration-wrc && git checkout <ref>
  ```

**Question 5 — Image build workflow** (select):
- Header: `Image build workflow`
- Question: `How should the Docker image be built?`
- Options:
  - `Local dev build` — build locally using vm-migration-wrc Makefile
    (fast iteration, uses `make docker-fedora-libguestfs`) (recommended)
  - `PR-based CI build` — push branch + PR to vm-migration-wrc, let
    GitHub Actions build (production-safe, avoids local Docker issues)
- Default: `Local dev build`

### Step 2: Build virt-v2v from source

If the `Build from source` stage is selected:

> **Reference:** This step mirrors the Fedora CI job in
> `.github/workflows/main.yml` which runs on every push/PR to master.

1. **Install all build dependencies** (one-shot, mirrors CI):
   ```bash
   dnf builddep -y virt-v2v
   # Also ensure git is available (needed for submodule checkout)
   dnf install -y git
   ```
   This single `dnf builddep` command resolves every library, OCaml
   package, and tool required by the build (libguestfs-devel, libnbd-devel,
   ocaml-findlib, ocaml-libguestfs-devel, autoconf, automake, libtool,
   etc.) — no need to install individual packages.

2. **Check build readiness:**
   ```bash
   cd /root/dev/virt-v2v
   git diff --stat
   ```
   Show the user what source changes will be compiled.

3. **Run build** (based on build mode from Step 1):

   *Full build (mirrors CI "Compile the code" step):*
   ```bash
   cd /root/dev/virt-v2v
   git submodule update --init
   autoreconf -fiv 2>&1 | tee /tmp/v2v-autoreconf.log | tail -10
   ./configure CFLAGS="-fPIC -g -O2" 2>&1 | tee /tmp/v2v-configure.log | tail -20
   make -j$(nproc) 2>&1 | tee /tmp/v2v-build.log | tail -30
   ```

   Key differences from the old instructions:
   - `autoreconf -fiv` (force + install + verbose) not just `-i`
   - `./configure CFLAGS="-fPIC -g -O2"` — matches CI flags for
     position-independent code and debug symbols
   - Separate log files for each phase to aid debugging

   *Incremental build:*
   ```bash
   cd /root/dev/virt-v2v
   make -j$(nproc) 2>&1 | tee /tmp/v2v-build.log | tail -30
   ```

4. **Verify build succeeded:**
   - Check exit code of `make` — if non-zero, display last 50 lines of
     `/tmp/v2v-build.log` and **stop**.
   - Verify binary exists:
     ```bash
     ls -la /root/dev/virt-v2v/in-place/virt-v2v-in-place
     ```

5. **Quick smoke test** (optional, if Verify stage selected):
   ```bash
   cd /root/dev/virt-v2v && ./run virt-v2v-in-place --help 2>&1 | head -5
   ```
   Confirm the binary runs and shows expected options.

### Step 3: Build & push Docker image

Based on Question 5, choose **Mode A** (local dev) or **Mode B** (PR-based CI).

---

#### Mode A: Local Dev Build (fast iteration)

Build the Docker image locally using the vm-migration-wrc Makefile.
Best for rapid testing — image is available immediately without
waiting for CI.

##### A.1 — Checkout vm-migration-wrc ref

```bash
cd /root/dev/vm-migration-wrc
```
If the user specified a ref in Question 4:
```bash
git checkout <ref>
```
Otherwise keep current HEAD. Show the user what ref is active:
```bash
git log --oneline -1
```

##### A.2 — Build mojav binary (skip if recent)

The base Dockerfile COPYs `mojav` from `baseimage/files/`. Check if
it already exists and is recent enough to skip rebuilding:
```bash
if [ -f baseimage/files/mojav ]; then
  AGE=$(( $(date +%s) - $(stat -c %Y baseimage/files/mojav) ))
  if [ $AGE -lt 86400 ]; then
    echo "mojav binary is recent (${AGE}s old), skipping rebuild"
  else
    echo "mojav binary is stale (${AGE}s old), rebuilding..."
    source ~/.gvm/scripts/gvm && gvm use go1.25.5
    make build-mojav
  fi
else
  echo "mojav binary not found, building..."
  source ~/.gvm/scripts/gvm && gvm use go1.25.5
  make build-mojav
fi
```
Verify:
```bash
ls -la baseimage/files/mojav
```

##### A.3 — Copy custom virt-v2v-in-place binary

```bash
\cp /root/dev/virt-v2v/in-place/virt-v2v-in-place \
   /root/dev/vm-migration-wrc/baseimage/files/virt-v2v-in-place
chmod +x /root/dev/vm-migration-wrc/baseimage/files/virt-v2v-in-place
```
Note: use `\cp` to bypass the `cp -i` alias.

##### A.4 — Patch Dockerfile (dry-run first)

Check if the overlay is already present; if not, append it:
```bash
grep -q 'virt-v2v-in-place' /root/dev/vm-migration-wrc/baseimage/Dockerfile
if [ $? -ne 0 ]; then
  cat /root/dev/virt-v2v/.github/skills/v2v-dev-loop/assets/Dockerfile.dev-template \
      >> /root/dev/vm-migration-wrc/baseimage/Dockerfile
fi
```

Apply the **kernel-install container fix** — inject before the first
`microdnf install` to prevent the kernel-core posttrans scriptlet
from failing (see Known Issues):
```bash
grep -q 'ln -sf /usr/bin/true /usr/bin/kernel-install' \
    /root/dev/vm-migration-wrc/baseimage/Dockerfile
if [ $? -ne 0 ]; then
  sed -i '/^RUN microdnf install/i RUN mkdir -p /boot && ln -sf /usr/bin/true /usr/bin/kernel-install' \
      /root/dev/vm-migration-wrc/baseimage/Dockerfile
fi
```

**Dry-run review** — show the diff so the user can confirm:
```bash
cd /root/dev/vm-migration-wrc && git diff baseimage/Dockerfile
```
Wait for user confirmation before proceeding.

##### A.5 — Build Docker image using Makefile

The Makefile target `docker-fedora-libguestfs` runs:
`docker buildx build . --platform linux/amd64 --file baseimage/Dockerfile`
and tags as `ghcr.io/solutions-innovation/fedora-libguestfs:<branch>`.

```bash
cd /root/dev/vm-migration-wrc
make docker-fedora-libguestfs 2>&1 | tee /tmp/v2v-docker-build.log | tail -20
```

If the user specified a custom tag in Question 2, also re-tag:
```bash
MAKE_TAG=$(cd /root/dev/vm-migration-wrc && git rev-parse --abbrev-ref HEAD | sed 's|/|-|g')
docker tag ghcr.io/solutions-innovation/fedora-libguestfs:${MAKE_TAG} \
    docker.io/michaelbi/fedora-libguestfs:${TAG}
```

If Docker build fails, display last 50 lines of log and **stop**.

##### A.6 — Push to registry (requires user consent)

If the `Push to registry` stage is selected, confirm with user:
```bash
# Show what will be pushed
docker image ls | grep fedora-libguestfs | head -5
```

Push (with explicit consent):
```bash
docker push docker.io/michaelbi/fedora-libguestfs:${TAG}
```
Or to ghcr.io if preferred:
```bash
docker push ghcr.io/solutions-innovation/fedora-libguestfs:${MAKE_TAG}
```

##### A.7 — Smoke test (if Verify stage selected)

```bash
docker run --rm docker.io/michaelbi/fedora-libguestfs:${TAG} \
    virt-v2v-in-place --help 2>&1 | head -10
```
Confirm the custom binary is present and shows expected options.

##### A.8 — Auto-revert working tree

Restore the Dockerfile and remove temporary binaries so the
vm-migration-wrc working tree stays clean:
```bash
cd /root/dev/vm-migration-wrc
git checkout baseimage/Dockerfile
rm -f baseimage/files/virt-v2v-in-place
# Keep mojav — it's frequently rebuilt and gitignored
```

---

#### Mode B: PR-based CI Build (production-safe)

Push the custom binary to vm-migration-wrc via a PR and let GitHub
Actions build the image. Best when local Docker builds fail or when
the image must go through CI validation.

##### B.1 — Set up SSH agent

```bash
eval $(ssh-agent) && ssh-add ~/.ssh/wr56
```

##### B.2 — Create a feature branch from main

```bash
cd /root/dev/vm-migration-wrc
git checkout main && git pull origin main
V2V_BRANCH=$(cd /root/dev/virt-v2v && git rev-parse --abbrev-ref HEAD | sed 's|/|-|g')
V2V_SHA=$(cd /root/dev/virt-v2v && git rev-parse --short HEAD)
BRANCH="v2v-${V2V_BRANCH}-${V2V_SHA}"
git checkout -b "${BRANCH}"
```

##### B.3 — Copy the custom binary

```bash
\cp /root/dev/virt-v2v/in-place/virt-v2v-in-place \
   /root/dev/vm-migration-wrc/baseimage/files/virt-v2v-in-place
chmod +x /root/dev/vm-migration-wrc/baseimage/files/virt-v2v-in-place
```

##### B.4 — Modify the Dockerfile

```bash
grep -q 'virt-v2v-in-place' /root/dev/vm-migration-wrc/baseimage/Dockerfile
if [ $? -ne 0 ]; then
  cat /root/dev/virt-v2v/.github/skills/v2v-dev-loop/assets/Dockerfile.dev-template \
      >> /root/dev/vm-migration-wrc/baseimage/Dockerfile
fi
cd /root/dev/vm-migration-wrc && git diff baseimage/Dockerfile
```

##### B.5 — Commit and push

```bash
cd /root/dev/vm-migration-wrc
git add baseimage/files/virt-v2v-in-place baseimage/Dockerfile
git commit -m "feat: overlay custom virt-v2v-in-place from ${V2V_BRANCH}@${V2V_SHA}

Built from libguestfs/virt-v2v branch '${V2V_BRANCH}' at commit ${V2V_SHA}.
This replaces the RPM-installed virt-v2v-in-place with a custom build."
git push origin "${BRANCH}"
```

##### B.6 — Create PR

Using `gh` CLI (if authenticated) or MCP tools:
```bash
gh pr create \
  --title "feat: custom virt-v2v-in-place from ${V2V_BRANCH}@${V2V_SHA}" \
  --body "Built from libguestfs/virt-v2v branch ${V2V_BRANCH} at ${V2V_SHA}." \
  --base main --head "${BRANCH}"
```
Otherwise provide the browser URL:
`https://github.com/Solutions-Innovation/vm-migration-wrc/pull/new/${BRANCH}`

##### B.7 — Ask user to monitor CI

Print the PR URL and remind the user to check the GitHub Actions run.

---

### Step 4: Cleanup and report

**For Mode A (local dev):**
```
=== v2v Dev Loop Complete (Local Build) ===
Source:     /root/dev/virt-v2v (branch: <branch>, commit: <sha>)
Changes:    <git diff --stat summary>
Image:      docker.io/michaelbi/fedora-libguestfs:<tag>
Also:       ghcr.io/solutions-innovation/fedora-libguestfs:<make-tag>
Cleanup:    Dockerfile reverted, temp binary removed
Build log:  /tmp/v2v-docker-build.log
```

**For Mode B (PR-based CI):**
```
=== v2v Dev Loop Complete (CI Build) ===
Source:     /root/dev/virt-v2v (branch: <branch>, commit: <sha>)
Changes:    <git diff --stat summary>
WRC Branch: v2v-<branch>-<sha> (pushed to origin)
PR:         <PR URL>
Next:       Review and merge the PR to trigger CI image build
Build log:  /tmp/v2v-build.log
```

## References

| File | Content | When to read |
|------|---------|-------------|
| [.github/workflows/main.yml](.github/workflows/main.yml) | CI build job for Fedora (builddep, autoreconf, configure, make) | Step 2 — authoritative build sequence |
| [assets/Dockerfile.dev-template](assets/Dockerfile.dev-template) | Dockerfile overlay lines appended to base | Step 3 — overlay COPY for custom binary |
| `/root/dev/vm-migration-wrc/Makefile` | `docker-fedora-libguestfs` target builds image; `build-mojav` builds Go CLI | Step 3A — local dev build |

## Rules

- **ALWAYS** build from `/root/dev/virt-v2v` source directory
- **ALWAYS** use `/root/dev/vm-migration-wrc` as Docker build context (it contains VDDK, ovftool, etc. in baseimage/files/)
- **ALWAYS** read the base Dockerfile at runtime (never hardcode its contents — it may change)
- **ALWAYS** clean up the temp binary from `vm-migration-wrc/baseimage/files/` after build
- **ALWAYS** auto-revert Dockerfile changes after local dev build (Mode A Step A.8)
- **ALWAYS** ask user for consent before pushing to Docker registry
- **ALWAYS** capture build output to `/tmp/v2v-build.log` or `/tmp/v2v-docker-build.log`
- **ALWAYS** show `git diff` of Dockerfile changes and wait for user confirmation before building (dry-run)
- In **Mode A** (local dev), Dockerfile modifications are temporary and auto-reverted
- In **Mode B** (PR-based CI), Dockerfile modifications are committed to the branch
- **NEVER** push without explicit user confirmation
- **NEVER** skip the build step when source files have changed (`git status` is dirty)
- If `make` fails, display the build log and **stop** — do not proceed to Docker build
- If Docker build fails, **stop** — do not proceed to push

## Known Issues

### kernel-core posttrans scriptlet fails in local Docker builds

When building the Docker image locally, the `microdnf install` step may fail
with `Transaction failed: Rpm transaction failed` after the `kernel-core`
posttrans scriptlet reports `Operation not permitted`.

**Root cause:** `libguestfs` and `qemu-kvm` pull in `kernel-core` as a
dependency. The kernel's post-install script (`kernel-install`) tries to
write to `/boot/<machine-id>` to configure the bootloader — an operation
blocked by the container runtime's seccomp profile and filesystem restrictions.
This works in GitHub Actions because its Docker environment runs with a more
permissive security profile than local Docker Desktop / rootless Podman.

**Fix:** The Dockerfile generation in Step 3.4 injects a line before the
`microdnf install` command that:
1. Creates `/boot` so the scriptlet's stat() call succeeds
2. Masks `/usr/bin/kernel-install` with a symlink to `/usr/bin/true`
```dockerfile
RUN mkdir -p /boot && ln -sf /usr/bin/true /usr/bin/kernel-install
```
This causes the posttrans scriptlet to find `/boot` and invoke a no-op
`kernel-install`, so the transaction succeeds. Safe because containers
do not need a configured bootloader.
```
