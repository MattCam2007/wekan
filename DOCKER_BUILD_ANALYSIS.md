# Docker Build Analysis: Moving from Pre-built Zip to Local Build

## Current Docker Build Process (Dockerfile)

The current `Dockerfile` downloads a pre-built WeKan bundle zip from GitHub releases:

```
WEKAN_ZIP_URL="https://github.com/wekan/wekan/releases/download/v${VERSION}/wekan-${VERSION}-${WEKAN_ARCH}.zip"
```

This zip contains a complete, self-contained WeKan application bundle.

## What the GitHub Release Zip Contains

Based on the build scripts (`releases/build-bundle-*.sh`) and Dockerfile analysis, the zip file contains:

### 1. **Meteor Application Bundle** (`.build/bundle` equivalent)
- `main.js` - Application entry point
- `programs/server/` - Server-side code with npm dependencies (partially installed)
- `programs/web.browser/` - Modern client bundle
- `programs/web.browser.legacy/` - Legacy client bundle (for older browsers)
- `programs/web.cordova/` - Cordova/mobile bundle (if applicable)

### 2. **Pre-installed Native Node.js Modules**
The bundle comes with native modules already compiled for the target architecture:
- `bcrypt` - Password hashing
- `fibers` - (Meteor 1.x only - not needed for Meteor 3)
- Other native npm dependencies from `package.json`

### 3. **Architecture-Specific Binaries**
- **FerretDB binary** (`/build/ferretdb`) - For architectures without MongoDB Community Edition (ppc64le, s390x, riscv64)
- **MongoDB Database Tools** - Embedded in the bundle for backup/restore:
  - `bsondump`, `mongodump`, `mongorestore`, `mongoexport`, `mongoimport`, `mongostat`, `mongotop`

### 4. **Self-contained Node.js Runtime**
- Bundled Node.js (`/build/node`) - Used by the offline launcher scripts
- Launcher scripts: `start-wekan.sh`, `start-wekan.bat`

### 5. **Entrypoint & Recovery Files** (also copied separately in Dockerfile)
- `wekan-entrypoint.sh` - Database backend selector (starts FerretDB or uses external MongoDB)
- `recovery-bridge.mjs` - Standalone "recovering data" page for FerretDB recovery

## What We Need to Build Locally (Replacing the Zip Download)

### Step 1: Build the Meteor Bundle
```bash
# Requires Meteor 3.x installed
meteor build .build --directory
```
This creates `.build/bundle/` with the complete application.

### Step 2: Install Server npm Dependencies
```bash
cd .build/bundle/programs/server
npm install --omit=dev
```
This compiles native modules for the **build machine's architecture**.

### Step 3: Handle Multi-Architecture Support
For architectures different from the build machine (e.g., building amd64 image on arm64 host):
- Use `npm rebuild` for target architecture (requires cross-compilation toolchain)
- Or use Docker Buildx with QEMU for native compilation

### Step 4: Add Architecture-Specific Binaries
- **FerretDB**: Download pre-built binary for target architecture from FerretDB releases
- **MongoDB Database Tools**: Download from MongoDB releases for target architecture
- These are NOT built from source - they are downloaded separately

### Step 5: Include Entrypoint & Recovery Files
Already in repo at:
- `releases/ferretdb/wekan-entrypoint.sh`
- `releases/ferretdb/recovery-bridge.mjs`

## Local Docker Build Requirements

### Build Dependencies (in Dockerfile)
```dockerfile
ENV BUILD_DEPS="apt-utils gnupg wget bzip2 g++ curl libarchive-tools build-essential git ca-certificates python3 unzip"
```

### Runtime Dependencies
- Node.js v24.18.0 (installed in Dockerfile)
- npm 11.12.1
- Meteor 3.5-rc.2 (for build only, not in final image)

### For Multi-arch Builds (Docker Buildx)
```bash
docker buildx create --name mybuilder --driver docker-container --use
docker buildx inspect --bootstrap
docker buildx build --platform linux/amd64,linux/arm64 --load -t wekan-local .
```

## What Will Be Missing If We Just Remove the Zip Download

| Component | Currently from Zip | Can Build Locally? | Notes |
|-----------|-------------------|-------------------|-------|
| Meteor bundle | ✅ Pre-built | ✅ Yes (`meteor build`) | Requires Meteor tool |
| Server npm deps | ✅ Pre-installed | ✅ Yes (`npm install`) | Compiles native modules |
| Client bundles | ✅ Pre-built | ✅ Yes (`meteor build`) | Includes modern + legacy |
| Native modules (bcrypt, etc) | ✅ Pre-compiled | ✅ Yes (via npm install) | Must compile on target arch |
| FerretDB binary | ✅ Included | ❌ No - download separately | Go binary, not in npm |
| MongoDB Database Tools | ✅ Included | ❌ No - download separately | C++ binaries from MongoDB |
| Node.js runtime | ✅ Bundled | ❌ No - install separately | Already done in Dockerfile |
| Entrypoint scripts | ✅ Included | ✅ Yes - copy from repo | Already in releases/ferretdb/ |

## Recommended Approach for `Dockerfile.local`

1. **Build Stage**: Use Meteor to build the bundle
   ```dockerfile
   # Install Meteor
   curl https://install.meteor.com/ | sh
   meteor build .build --directory
   ```

2. **Install Stage**: Install server dependencies
   ```dockerfile
   cd .build/bundle/programs/server && npm install --omit=dev
   ```

3. **Runtime Stage**: Copy bundle + download FerretDB + MongoDB Tools
   ```dockerfile
   # Copy built bundle
   COPY --from=builder /app/.build/bundle /build
   
   # Download FerretDB for arch
   # Download MongoDB Database Tools for arch
   
   # Copy entrypoint scripts
   COPY releases/ferretdb/wekan-entrypoint.sh /build/
   COPY releases/ferretdb/recovery-bridge.mjs /build/
   ```

4. **Use Multi-stage Build** to keep final image small

## Key Files to Reference

- `/home/matt/Projects/wekAIn/wekan/Dockerfile` - Current production Dockerfile
- `/home/matt/Projects/wekAIn/wekan/releases/build-bundle-*.sh` - How release bundles are created
- `/home/matt/Projects/wekAIn/wekan/releases/docker-build.sh` - Multi-arch Docker build process
- `/home/matt/Projects/wekAIn/wekan/build.sh` - Local build function (`build_wekan`)
- `/home/matt/Projects/wekAIn/wekan/release/ferretdb/wekan-entrypoint.sh` - Entrypoint script
- `/home/matt/Projects/wekAIn/wekan/package.json` - Dependencies and Meteor config