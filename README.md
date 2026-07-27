# n8n-unlocked

**n8n-unlocked** is a patch repository that removes the license limitations from [n8n](https://n8n.io) by replacing its commercial license SDK (`@n8n_io/license-sdk`) with a lightweight stub that reports all features as licensed. The repository includes automated GitHub Actions workflows that build and publish unlocked Docker images to Docker Hub.

> ⚠️ **Disclaimer:** This project is for educational and research purposes only. n8n is released under the [Sustainable Use License](https://github.com/n8n-io/n8n/blob/master/LICENSE.md) and a [Enterprise License](https://github.com/n8n-io/n8n/blob/master/LICENSE_EE.md). Use of n8n may require a paid license from the original authors. Please respect the original project's licensing terms.

> 🧑‍🎓 **A personal note:** This is honestly the first time I've ever patched anything, so this whole project is really just me learning how it works. I built it for educational purposes — to understand patching, CI/CD workflows, and how all the pieces fit together. Nothing more.

---

## How it works

The project contains a single patch file ([`license-stub.patch`](./license-stub.patch)) that:

1. **Adds a stub license package** (`packages/@n8n/license-stub`) — a drop-in replacement for `@n8n_io/license-sdk` that:
   - Returns `true` for all feature checks (`isValid`, `hasFeatureEnabled`, `hasQuotaLeft`, etc.)
   - Overrides `getFeatureValue('planName')` to always return `"Enterprise"`
   - Sets the license expiry date to year 2099
   - Logs stub activity but performs no real validation

2. **Redirects the dependency** in `packages/cli/package.json` from the real `@n8n_io/license-sdk` to the local stub via a `workspace:*` reference.

3. **Updates `pnpm-lock.yaml`** to resolve the dependency locally instead of downloading the official package.

---

## Repository structure

```
.
├── .github/workflows/
│   ├── release-cron.yml      # Daily cron: checks for new upstream releases & triggers builds
│   └── release-docker.yml    # Builds & pushes the unlocked Docker image
├── packages/@n8n/license-stub/
│   └── src/
│       ├── index.js           # Stub implementation (all features licensed)
│       └── index.d.ts         # TypeScript declarations matching the original SDK
├── license-stub.patch         # The patch applied to the upstream n8n source
└── README.md
```

---

## GitHub Actions workflows

### `release-cron.yml` (daily schedule)

Runs every day at 12:00 UTC and can also be triggered manually.

1. Fetches the **latest release tag** from the upstream [`n8n-io/n8n`](https://github.com/n8n-io/n8n) repository.
2. Checks if that tag already exists in **this repository** (to avoid duplicates).
3. If the release is new, it **creates a GitHub Release** pointing to the upstream tag.
4. **Dispatches** the `release-docker.yml` workflow to build and publish the Docker image.

### `release-docker.yml` (manual / dispatched)

Triggered manually with a version tag (e.g. `n8n@1.80.0`) or automatically by the cron workflow.

1. **Clones** the upstream n8n repository at the specified tag.
2. **Applies** `license-stub.patch` to the upstream source.
3. **Installs dependencies** with pnpm (frozen lockfile).
4. **Builds** n8n (`pnpm build:n8n`).
5. **Builds and pushes** multi-architecture Docker images (`linux/amd64` + `linux/arm64`) to Docker Hub.

---

## Docker images

Pre-built unlocked images are available on Docker Hub:

**`ahmgam/n8n`**

Tags follow the upstream n8n version scheme (e.g. `ahmgam/n8n:1.80.0`).

### Usage

```bash
# Pull and run the latest unlocked version
docker run -d \
  --name n8n-unlocked \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  ahmgam/n8n:latest

# Or specify a particular version
docker run -d \
  --name n8n-unlocked \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  ahmgam/n8n:1.80.0
```

Then open [http://localhost:5678](http://localhost:5678) in your browser.

> **Note:** You may need to log in or create an account on first launch. All enterprise features (SSO, SAML, RBAC, LDAP, log streaming, etc.) will appear as fully licensed.

---

## Building locally

```bash
# Clone the upstream n8n repository
git clone --depth=1 -b n8n@1.80.0 https://github.com/n8n-io/n8n.git

# Apply the patch
cd n8n
git apply /path/to/n8n-unlocked/license-stub.patch

# Install dependencies and build
pnpm install --frozen-lockfile
pnpm build:n8n

# Build the Docker image
docker build -f docker/images/n8n/Dockerfile \
  --build-arg NODE_VERSION=24.18.0 \
  --build-arg N8N_VERSION=1.80.0 \
  --build-arg N8N_RELEASE_TYPE=stable \
  -t n8n-unlocked .
```