# OpenShell Podman Local Development Setup

This guide documents the setup for running OpenShell locally with Podman and Red Hat base images.

## Built Images

We've successfully built two container images with Red Hat base images:

1. **Gateway Image** (`localhost/openshell/gateway:dev`)
   - Base: `registry.access.redhat.com/hi/core-runtime:latest`
   - Size: ~300 MB
   - Contains: `openshell-gateway` binary (GNU-linked)

2. **Supervisor Image** (`localhost/openshell/supervisor:dev`)
   - Base: `scratch` (minimal)
   - Size: ~17.6 MB
   - Contains: `openshell-sandbox` binary (statically-linked musl)

## Build Process

### Gateway Binary
```bash
# Build the gateway binary (debug mode)
mise run build:gateway
# or
cargo build -p openshell-server --bin openshell-gateway

# Binary location: target/debug/openshell-gateway
```

### Supervisor Binary
```bash
# Build the supervisor binary (release, static musl)
mise x -- cargo zigbuild --release -p openshell-sandbox --bin openshell-sandbox --target x86_64-unknown-linux-musl

# Binary location: target/x86_64-unknown-linux-musl/release/openshell-sandbox
```

### Container Images

#### Stage Binaries
```bash
# Create staging directory
mkdir -p deploy/docker/.build/prebuilt-binaries/amd64

# Stage gateway binary (debug build)
cp target/debug/openshell-gateway deploy/docker/.build/prebuilt-binaries/amd64/

# Stage supervisor binary (release build)
cp target/x86_64-unknown-linux-musl/release/openshell-sandbox deploy/docker/.build/prebuilt-binaries/amd64/
```

#### Build Images with Podman
```bash
# Build gateway image
podman build -f deploy/docker/Dockerfile.gateway --target gateway -t openshell/gateway:dev .

# Build supervisor image
podman build -f deploy/docker/Dockerfile.supervisor --target supervisor -t openshell/supervisor:dev .
```

## Running the Test Environment

We've created a test script that runs the OpenShell gateway with Podman:

```bash
./test-local-gateway.sh
```

This script:
- Builds the gateway and supervisor binaries if needed
- Generates TLS certificates for JWT signing
- Creates a gateway configuration for Podman
- Registers the gateway metadata
- Starts the gateway on `http://127.0.0.1:18080`

### Environment Variables

You can customize the test environment with these variables:

- `OPENSHELL_SERVER_PORT` - Gateway port (default: 18080)
- `OPENSHELL_GATEWAY_NAME` - Gateway name (default: podman-dev)
- `OPENSHELL_SANDBOX_NAMESPACE` - Sandbox namespace (default: podman-dev)
- `OPENSHELL_SANDBOX_IMAGE` - Supervisor image (default: localhost/openshell/supervisor:dev)
- `OPENSHELL_LOG_LEVEL` - Log level (default: debug)

Example:
```bash
OPENSHELL_SERVER_PORT=19080 OPENSHELL_LOG_LEVEL=info ./test-local-gateway.sh
```

## Using the Gateway

Once the gateway is running, you can use the OpenShell CLI:

```bash
# Check status
openshell --gateway podman-dev status

# Or set as default gateway
openshell gateway select podman-dev
openshell status
```

## Red Hat Base Images

The Dockerfiles have been modified to use Red Hat base images:

- **Gateway**: `registry.access.redhat.com/hi/core-runtime:latest`
  - Provides glibc and minimal runtime dependencies
  - Red Hat security hardening
  - Suitable for GNU-linked binaries

- **Supervisor**: `scratch`
  - No base image needed
  - Static musl binary
  - Minimal attack surface

## Your Sandbox Image

You created `deploy/docker/Dockerfile.sandbox` for running agent workloads with:
- Stage 1: UBI9 builder with Node.js 20, git, npm
- Stage 2: Hardened Node.js runtime from images.redhat.com

This appears to be for running OpenCode or similar agent frameworks. Note that the `@opencode/cli` package doesn't exist in npm - you'll need to either:
1. Use a different package name
2. Build a custom agent runtime
3. Copy pre-built agent binaries into the image

## Next Steps

To integrate with OpenShift:

1. **Push images to a registry accessible from OpenShift**:
   ```bash
   podman tag localhost/openshell/gateway:dev <registry>/openshell/gateway:dev
   podman push <registry>/openshell/gateway:dev
   
   podman tag localhost/openshell/supervisor:dev <registry>/openshell/supervisor:dev
   podman push <registry>/openshell/supervisor:dev
   ```

2. **Deploy using Helm** (see `deploy/helm/openshell/`):
   ```bash
   helm install openshell deploy/helm/openshell/ \
     --set gateway.image=<registry>/openshell/gateway:dev \
     --set supervisor.image=<registry>/openshell/supervisor:dev
   ```

3. **Configure for OpenShift**:
   - Update security context constraints
   - Configure service accounts
   - Set up network policies
   - Configure persistent storage

## Troubleshooting

### Build Script Issues

If you encounter the "SCRIPT_DIR has newline" issue when running build scripts:
```bash
# Use podman directly instead of the helper scripts
podman build -f <dockerfile> -t <tag> .
```

This appears to be a fish shell configuration issue with how `pwd` output is captured in bash scripts.

### Musl Build Requires cargo-zigbuild

The supervisor binary must be built with musl for static linking. This requires:
- `cargo-zigbuild` (installed via mise)
- `zig` compiler (installed via mise)

Both are configured in `mise.toml` and installed automatically by mise.
