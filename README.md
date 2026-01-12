# Putting nsjail in Front of BuildKit: a Practical Sandbox Pattern

Dockerfiles are executable code. If you build untrusted inputs, every `RUN` step can read host files or poke local sockets—unless you fence it in. Here is a concise, reusable recipe for wrapping any BuildKit-based tool (Docker, `buildx`, `nerdctl build`, `ktl`, CI runners) in an nsjail sandbox so the build only sees what you allow.

## Why nsjail + BuildKit
- **Process boundary**: PID/user namespaces stop build steps from assuming host root.
- **Filesystem allowlist**: bind only context, cache, and intentional extras; everything else is tmpfs.
- **Network control**: keep host networking for registries or flip to no-egress for hermetic builds.
- **Drop-in**: BuildKit needs a filesystem and a builder socket; re-exec the client and you are done.

## High-level flow
1. Write or reuse an nsjail policy.
2. Re-exec the build command inside nsjail with minimal binds.
3. Pass stdin/stdout through so the build UI survives.
4. Stream nsjail logs when debugging.

## Minimal policy (linux-ci flavored)
```cfg
name: "buildkit-ci"
hostname: "build"
mode: EXECVE
cwd: "/workspace"
clone_newuts: true
clone_newipc: true
clone_newpid: true
clone_newuser: true
clone_newcgroup: true
clone_newnet: false    # set true for hermetic/no-egress builds
keep_caps: false
seccomp_string: "DEFAULT ALLOW"

mount { dst: "/" fstype: "tmpfs" rw: true }
mount { dst: "/proc" fstype: "proc" }
mount { dst: "/tmp" fstype: "tmpfs" rw: true options: "size=2G" }
mount { dst: "/run" fstype: "tmpfs" rw: true options: "size=128M" }
# DNS + CA trust (read-only)
mount { src: "/etc/resolv.conf" dst: "/etc/resolv.conf" is_bind: true }
mount { src: "/etc/hosts" dst: "/etc/hosts" is_bind: true }
mount { src: "/etc/ssl/certs" dst: "/etc/ssl/certs" is_bind: true }
# Device passthrough for tooling
mount { src: "/dev" dst: "/dev" is_bind: true rw: true }
```
Save this as `sandbox/linux-ci.cfg`.

## Wrap Docker Buildx / nerdctl / any client
A tiny shim script can re-exec the tool inside nsjail. Drop this somewhere in `$PATH` (e.g., `bin/build-with-nsjail`):
```bash
#!/usr/bin/env bash
set -euo pipefail
POLICY=${POLICY_PATH:-$HOME/sandbox/linux-ci.cfg}
BIN=${BIN:-buildx}          # docker buildx, nerdctl, or another wrapper
CTX=${1:-.}
shift || true

# Resolve a cache dir that is safe to mount
CACHE=${CACHE_DIR:-$HOME/.cache/buildkit}
mkdir -p "$CACHE"

# Builder socket (docker daemon by default; swap for rootless/tcp builder as needed)
BUILDER=${BUILDER_SOCKET:-/var/run/docker.sock}

exec nsjail \
  --config "$POLICY" \
  --bindmount "$PWD:/workspace" \
  --bindmount "$CACHE:/bk-cache" \
  --bindmount "$BUILDER:/var/run/docker.sock" \
  --cwd /workspace \
  -- "$BIN" build "$CTX" --builder default "$@"
```
Usage:
```bash
POLICY_PATH=$PWD/sandbox/linux-ci.cfg BIN="docker buildx" \
  ./bin/build-with-nsjail . -f Dockerfile.untrusted
```
Swap `--bindmount ... docker.sock` for a rootless BuildKit socket if you want to avoid exposing the Docker daemon.

## Tightening the screws (hermetic builds)
- Set `clone_newnet: true` and drop the Docker socket; talk to a rootless or remote BuildKit endpoint instead.
- Prefer `buildctl` with an explicit endpoint to keep the surface smaller.
- Use a separate cache directory per trust tier (`~/.cache/buildkit-untrusted`).

## Probing visibility and debugging
Quick check before risky builds:
```bash
HOST_MARKER=/etc/shadow
nsjail --config sandbox/linux-ci.cfg -- \
  /bin/sh -lc "stat $HOST_MARKER && echo visible" || echo hidden
```
For failures, add `--log /tmp/nsjail.log` and read the tail. If user namespaces are unsupported, drop `clone_newuser`; if seccomp blocks threads, keep `seccomp_string: "DEFAULT ALLOW"`.

## CI integration pattern
1. Check in the policy under `sandbox/` and review it like code.
2. Add a wrapper step before builds:
   ```bash
   POLICY_PATH=$PWD/sandbox/linux-ci.cfg BIN="docker buildx" ./bin/build-with-nsjail .
   ```
3. Fail the pipeline if nsjail is missing or the policy cannot be parsed; never silently fall back.
4. Upload the nsjail log on failures.

## Takeaways
- nsjail is a light, configurable cage for BuildKit clients: no daemon changes, just a re-exec.
- Bind only context, cache, and required sockets; keep the rest on tmpfs.
- Version policies and keep a demo script so teammates trust the isolation story.
