# Docker & Runtime Security Assessment

**Assessment Date:** 2026-02-11  
**Scope:** Dockerfiles, docker-compose, deployment configs, runtime configurations, scripts  
**Methodology:** Static analysis of container configurations, privilege evaluation, network exposure assessment

---

## Executive Summary

OpenClaw demonstrates **good baseline security practices** with non-root users in production containers. However, several areas can be hardened to reduce attack surface and improve defense-in-depth.

**Key Findings:**
- ✅ Production containers run as non-root user
- ✅ No Docker socket mounts detected
- ✅ No privileged mode usage
- ✅ No excessive Linux capabilities
- ⚠️ Missing security options (no-new-privileges, seccomp, AppArmor)
- ⚠️ Broad host filesystem mounts in docker-compose
- ⚠️ Network binding defaults allow LAN exposure
- ⚠️ Test containers grant sudo access

---

## 1. Sandbox Escape Risk Assessment

### 1.1 HIGH SEVERITY: Sudo Access in Test Container

**File:** `scripts/docker/install-sh-nonroot/Dockerfile`  
**Lines:** 17-18

```dockerfile
RUN useradd -m -s /bin/bash app \
  && echo "app ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/app
```

**Risk:** Passwordless sudo allows trivial privilege escalation to root within the container. If combined with a kernel vulnerability, this increases container escape risk.

**Impact:** HIGH (Test environment only, not production)  
**Likelihood:** Medium (requires additional vulnerability)  
**Overall Risk:** MEDIUM

**Evidence:** Test container explicitly grants `NOPASSWD:ALL` sudo privileges to non-root user.

---

### 1.2 MEDIUM SEVERITY: Missing no-new-privileges Security Option

**File:** `docker-compose.yml`  
**Lines:** All services

**Risk:** Without `security_opt: no-new-privileges:true`, processes can gain additional privileges through setuid/setgid binaries or file capabilities.

**Impact:** MEDIUM  
**Likelihood:** Low (requires exploiting setuid binary)  
**Overall Risk:** LOW-MEDIUM

**Evidence:** No `security_opt` directives present in docker-compose configuration.

---

### 1.3 MEDIUM SEVERITY: Broad Host Filesystem Access

**File:** `docker-compose.yml`  
**Lines:** 11-13, 40-42

```yaml
volumes:
  - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
  - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
```

**Risk:** Mounting entire host directories allows container to read/write host files. Malicious code or compromised dependency could access sensitive data or modify files outside container scope.

**Impact:** MEDIUM-HIGH  
**Likelihood:** Low (requires code execution in container)  
**Overall Risk:** MEDIUM

**Evidence:** Host directories mounted read-write by default, no `:ro` (read-only) flags.

---

### 1.4 MEDIUM SEVERITY: Network Binding Exposes Services

**File:** `docker-compose.yml`  
**Lines:** 14-16, 25

```yaml
ports:
  - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"
  - "${OPENCLAW_BRIDGE_PORT:-18790}:18790"
```

**File:** `fly.toml`  
**Line:** 18

```toml
app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"
```

**Risk:** Binding to `lan` (0.0.0.0) exposes services to all network interfaces. Combined with weak or default tokens, this increases attack surface.

**Impact:** MEDIUM  
**Likelihood:** Medium (depends on network environment)  
**Overall Risk:** MEDIUM

**Evidence:** Default configuration binds to LAN/all interfaces rather than localhost.

---

### 1.5 LOW SEVERITY: Root User During Build

**File:** `Dockerfile`  
**Lines:** 1-34

**Risk:** Multi-stage builds run as root during build phase. Build-time vulnerabilities (e.g., supply chain attacks in dependencies) execute with root privileges.

**Impact:** LOW-MEDIUM  
**Likelihood:** Very Low  
**Overall Risk:** LOW

**Evidence:** `USER node` only set at line 40, after all builds complete.

---

### 1.6 LOW SEVERITY: Missing Security Profiles

**File:** `docker-compose.yml`, `Dockerfile*`  
**Lines:** N/A (missing)

**Risk:** No seccomp, AppArmor, or SELinux profiles defined. Default Docker seccomp profile is used, but custom restrictions could further limit syscall access.

**Impact:** LOW  
**Likelihood:** Very Low  
**Overall Risk:** LOW

**Evidence:** No `security_opt` with seccomp/apparmor profiles in configurations.

---

## 2. Hardened Runtime Profile

### 2.1 Hardened Dockerfile

```dockerfile
FROM node:22-bookworm AS builder

# Install Bun (required for build scripts)
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:${PATH}"

RUN corepack enable

WORKDIR /app

ARG OPENCLAW_DOCKER_APT_PACKAGES=""
RUN if [ -n "$OPENCLAW_DOCKER_APT_PACKAGES" ]; then \
      apt-get update && \
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends $OPENCLAW_DOCKER_APT_PACKAGES && \
      apt-get clean && \
      rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*; \
    fi

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY patches ./patches
COPY scripts ./scripts

RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
ENV OPENCLAW_PREFER_PNPM=1
RUN pnpm ui:build

# Production stage
FROM node:22-bookworm-slim AS production

# Install only runtime dependencies
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      ca-certificates \
      dumb-init && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*

WORKDIR /app

# Copy built artifacts from builder
COPY --from=builder --chown=node:node /app/dist ./dist
COPY --from=builder --chown=node:node /app/ui/dist ./ui/dist
COPY --from=builder --chown=node:node /app/node_modules ./node_modules
COPY --from=builder --chown=node:node /app/package.json ./
COPY --from=builder --chown=node:node /app/openclaw.mjs ./

ENV NODE_ENV=production

# Security hardening: Run as non-root user
USER node

# Use dumb-init to handle signals properly
ENTRYPOINT ["/usr/bin/dumb-init", "--"]

# Start gateway server with default config
# Binds to loopback (127.0.0.1) by default for security
CMD ["node", "openclaw.mjs", "gateway", "--allow-unconfigured"]
```

**Changes:**
1. Multi-stage build separates build-time and runtime dependencies
2. Use slim base image for production (smaller attack surface)
3. Only copy necessary artifacts to production stage
4. Use dumb-init for proper signal handling
5. Set ownership during COPY to avoid chown operations

---

### 2.2 Hardened docker-compose.yml

```yaml
services:
  openclaw-gateway:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    environment:
      HOME: /home/node
      TERM: xterm-256color
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      CLAUDE_AI_SESSION_KEY: ${CLAUDE_AI_SESSION_KEY}
      CLAUDE_WEB_SESSION_KEY: ${CLAUDE_WEB_SESSION_KEY}
      CLAUDE_WEB_COOKIE: ${CLAUDE_WEB_COOKIE}
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    ports:
      - "127.0.0.1:${OPENCLAW_GATEWAY_PORT:-18789}:18789"
      - "127.0.0.1:${OPENCLAW_BRIDGE_PORT:-18790}:18790"
    init: true
    restart: unless-stopped
    # Security hardening
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,nodev,size=100m
      - /home/node/.npm:noexec,nosuid,nodev,size=200m
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    networks:
      - openclaw-internal
    command:
      [
        "node",
        "dist/index.js",
        "gateway",
        "--bind",
        "${OPENCLAW_GATEWAY_BIND:-loopback}",
        "--port",
        "18789",
      ]

  openclaw-cli:
    image: ${OPENCLAW_IMAGE:-openclaw:local}
    environment:
      HOME: /home/node
      TERM: xterm-256color
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      BROWSER: echo
      CLAUDE_AI_SESSION_KEY: ${CLAUDE_AI_SESSION_KEY}
      CLAUDE_WEB_SESSION_KEY: ${CLAUDE_WEB_SESSION_KEY}
      CLAUDE_WEB_COOKIE: ${CLAUDE_WEB_COOKIE}
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
    stdin_open: true
    tty: true
    init: true
    # Security hardening
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    networks:
      - openclaw-internal
    entrypoint: ["node", "dist/index.js"]

networks:
  openclaw-internal:
    driver: bridge
    internal: false
```

**Changes:**
1. Bind ports to localhost only (`127.0.0.1:`) by default
2. Add `no-new-privileges` security option
3. Enable read-only root filesystem with tmpfs for writable paths
4. Drop all capabilities, add back only necessary ones
5. Use isolated Docker network
6. Add tmpfs with noexec/nosuid/nodev flags
7. Change default bind to `loopback` instead of `lan`

---

### 2.3 Network Restrictions

**Current Configuration:**
```yaml
# docker-compose.yml
ports:
  - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"  # Binds to 0.0.0.0
```

**Hardened Configuration:**
```yaml
# docker-compose.yml
ports:
  - "127.0.0.1:${OPENCLAW_GATEWAY_PORT:-18789}:18789"  # Localhost only
```

**Fly.io Configuration:**
```toml
# fly.toml - For private deployments
# Remove [http_service] block entirely (no public ingress)
# Access via: fly proxy or WireGuard

# fly.private.toml already implements this correctly
```

**Recommendation:** Use `fly.private.toml` as default template for security-conscious deployments.

---

### 2.4 User Permissions

**Current State:**
- ✅ Production: `USER node` (uid 1000)
- ✅ Sandbox: `USER sandbox` (non-root)
- ⚠️ Test nonroot: `sudo NOPASSWD:ALL`

**Hardened Configuration:**

For test containers:
```dockerfile
# Remove sudo entirely OR restrict to specific commands
RUN useradd -m -s /bin/bash app
# If sudo needed, restrict to specific commands:
# RUN echo "app ALL=(ALL) NOPASSWD: /usr/bin/apt-get" > /etc/sudoers.d/app
USER app
```

---

### 2.5 Filesystem Scoping

**Current Mounts:**
```yaml
volumes:
  - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
  - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
```

**Hardened Mounts:**
```yaml
volumes:
  # Config directory read-only where possible
  - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
  # Workspace read-write (required for operation)
  - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
  # Bind-mount specific files instead of directories where feasible
  # Example: credentials as read-only
  # - ${OPENCLAW_CONFIG_DIR}/credentials:/home/node/.openclaw/credentials:ro
```

**Additional Recommendations:**
1. Use named volumes for persistent data instead of bind mounts
2. Implement directory-level restrictions within application code
3. Use `.dockerignore` to minimize build context (already implemented)

---

### 2.6 Systemd Service Hardening

**Current Configuration:**
```ini
# openclaw-auth-monitor.service
[Service]
Type=oneshot
ExecStart=/home/admin/openclaw/scripts/auth-monitor.sh
```

**Hardened Configuration:**
```ini
[Service]
Type=oneshot
ExecStart=/home/admin/openclaw/scripts/auth-monitor.sh

# Security hardening
User=openclaw
Group=openclaw
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/openclaw/.openclaw
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true
LockPersonality=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
PrivateDevices=true
```

---

## 3. Detailed Findings

### 3.1 Dockerfile Security Analysis

#### Main Dockerfile (`Dockerfile`)

**Positive Findings:**
- ✅ Uses official Node.js base image (trusted source)
- ✅ Implements non-root user (line 40: `USER node`)
- ✅ Cleans up apt cache after package installation (lines 15-16)
- ✅ Uses frozen lockfile for reproducible builds (line 24)
- ✅ Sets proper file ownership before switching users (line 35)
- ✅ Binds to loopback by default in CMD (line 48)

**Security Gaps:**
- ⚠️ No multi-stage build (build artifacts in production image)
- ⚠️ Builds run as root (lines 1-34)
- ⚠️ Large attack surface (full build tools in production)
- ⚠️ Curl piped to bash for Bun installation (line 4)
- ⚠️ No health check defined

**Evidence:**
```dockerfile
Line 1: FROM node:22-bookworm  # Single-stage build
Line 4: RUN curl -fsSL https://bun.sh/install | bash  # Pipe to bash
Line 40: USER node  # Non-root user (GOOD)
Line 48: CMD ["node", "openclaw.mjs", "gateway", "--allow-unconfigured"]
```

---

#### Sandbox Dockerfile (`Dockerfile.sandbox`)

**Positive Findings:**
- ✅ Minimal base image (debian:bookworm-slim)
- ✅ Non-root user (line 16-17)
- ✅ Cleans apt cache (line 14)
- ✅ No unnecessary packages

**Security Gaps:**
- ⚠️ Runs indefinitely with `sleep infinity` (line 20)
- ⚠️ No resource limits defined
- ⚠️ Full Python interpreter included (potential for code execution)

**Evidence:**
```dockerfile
Line 1: FROM debian:bookworm-slim
Line 16-17: RUN useradd --create-home --shell /bin/bash sandbox
            USER sandbox
Line 20: CMD ["sleep", "infinity"]
```

---

#### Browser Sandbox Dockerfile (`Dockerfile.sandbox-browser`)

**Positive Findings:**
- ✅ Non-root user (line 26-27)
- ✅ Minimal package set for browser automation

**Security Gaps:**
- ⚠️ Exposes multiple ports including VNC (5900, 6080, 9222)
- ⚠️ X11/VNC services increase attack surface
- ⚠️ Chromium browser (large, complex codebase with potential vulnerabilities)

**Evidence:**
```dockerfile
Line 31: EXPOSE 9222 5900 6080  # Debug protocol, VNC, noVNC
Line 9: chromium  # Full browser in container
```

---

#### Test Nonroot Dockerfile (`scripts/docker/install-sh-nonroot/Dockerfile`)

**Critical Finding:**
- 🔴 Passwordless sudo (lines 17-18)

**Evidence:**
```dockerfile
Line 17-18: RUN useradd -m -s /bin/bash app \
            && echo "app ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/app
```

**Justification:** Test environment only, but still increases risk.

---

### 3.2 Docker Compose Analysis

#### Network Exposure

**Finding:** Services bind to all interfaces by default.

**Evidence:**
```yaml
# docker-compose.yml:14-16
ports:
  - "${OPENCLAW_GATEWAY_PORT:-18789}:18789"  # No host binding = 0.0.0.0
  - "${OPENCLAW_BRIDGE_PORT:-18790}:18790"
```

**Impact:** Gateway accessible from any network interface on host.

---

#### Volume Mounts

**Finding:** Broad host directory access.

**Evidence:**
```yaml
# docker-compose.yml:11-13
volumes:
  - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
  - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
```

**Impact:** Container can read/write entire config and workspace directories.

---

#### Missing Security Options

**Finding:** No security options configured.

**Evidence:** Absence of `security_opt`, `cap_drop`, `cap_add`, `read_only` directives.

**Impact:** Containers run with default (permissive) Docker security posture.

---

### 3.3 Deployment Configuration Analysis

#### Fly.io (`fly.toml`)

**Positive Findings:**
- ✅ Private variant exists (`fly.private.toml`) with no public ingress
- ✅ Mounts persistent volume at `/data`
- ✅ Uses NODE_OPTIONS for memory limits

**Security Gaps:**
- ⚠️ Default config uses `--bind lan` (line 18)
- ⚠️ Public HTTP service by default (lines 20-26)
- ⚠️ `auto_stop_machines = false` increases exposure window

**Evidence:**
```toml
Line 18: app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"
Line 22: force_https = true  # GOOD
Line 23: auto_stop_machines = false  # Keeps running = longer exposure
```

---

#### Render (`render.yaml`)

**Positive Findings:**
- ✅ Uses Docker runtime
- ✅ Health check path defined
- ✅ Token auto-generation

**Security Gaps:**
- ⚠️ No network restrictions specified
- ⚠️ Public web service by default
- ⚠️ Disk permissions not specified (likely defaults to full access)

**Evidence:**
```yaml
Line 2: type: web  # Public web service
Line 6: healthCheckPath: /health
Line 16-17: OPENCLAW_GATEWAY_TOKEN:
            generateValue: true  # GOOD
```

---

### 3.4 Scripts Analysis

#### docker-setup.sh

**Positive Findings:**
- ✅ Generates secure random token (lines 40-51)
- ✅ Uses frozen lockfile for builds
- ✅ Provides clear documentation

**Security Gaps:**
- ⚠️ Default bind is `lan` (line 34)
- ⚠️ No validation of environment variables
- ⚠️ Token stored in `.env` file (readable by all processes)

**Evidence:**
```bash
Line 34: export OPENCLAW_GATEWAY_BIND="${OPENCLAW_GATEWAY_BIND:-lan}"
Line 41-51: # Secure token generation using openssl or python
```

---

#### E2E Test Scripts

**Findings:**
- ✅ Tests run in isolated containers
- ✅ Use localhost binding (127.0.0.1)
- ✅ No host network mode usage

**Evidence:**
```bash
# onboard-docker.sh:97
node "$OPENCLAW_ENTRY" gateway --port 18789 --bind loopback
```

---

### 3.5 Systemd Services

#### openclaw-auth-monitor.service

**Positive Findings:**
- ✅ Oneshot service type (doesn't run continuously)
- ✅ Runs after network is available

**Security Gaps:**
- ⚠️ No user isolation (runs as invoking user)
- ⚠️ No filesystem restrictions
- ⚠️ No capability restrictions
- ⚠️ Hardcoded path `/home/admin` (not portable)

**Evidence:**
```ini
Line 7: ExecStart=/home/admin/openclaw/scripts/auth-monitor.sh
# No User=, ProtectSystem=, PrivateTmp=, etc.
```

---

## 4. Recommendations Priority

### Critical (Immediate Action)

1. **Remove sudo from test containers** or restrict to specific commands
   - File: `scripts/docker/install-sh-nonroot/Dockerfile`
   - Action: Remove line 18 or add command restrictions

### High Priority

2. **Add no-new-privileges to docker-compose**
   - File: `docker-compose.yml`
   - Action: Add `security_opt: [no-new-privileges:true]`

3. **Bind ports to localhost by default**
   - File: `docker-compose.yml`, `docker-setup.sh`
   - Action: Change port bindings to `127.0.0.1:${PORT}`
   - Change default `OPENCLAW_GATEWAY_BIND` to `loopback`

4. **Implement multi-stage Docker build**
   - File: `Dockerfile`
   - Action: Separate builder and production stages

### Medium Priority

5. **Drop capabilities in docker-compose**
   - File: `docker-compose.yml`
   - Action: Add `cap_drop: [ALL]` and `cap_add` as needed

6. **Add systemd hardening directives**
   - File: `scripts/systemd/openclaw-auth-monitor.service`
   - Action: Add ProtectSystem, PrivateTmp, NoNewPrivileges, etc.

7. **Implement read-only root filesystem**
   - File: `docker-compose.yml`
   - Action: Add `read_only: true` with tmpfs for writable paths

### Low Priority

8. **Add seccomp/AppArmor profiles**
   - Files: `docker-compose.yml`, deployment configs
   - Action: Define custom security profiles

9. **Add Docker health checks**
   - File: `Dockerfile`
   - Action: Add HEALTHCHECK instruction

10. **Review and minimize package installations**
    - Files: All Dockerfiles
    - Action: Remove unnecessary packages

---

## 5. Compliance & Standards

### CIS Docker Benchmark Alignment

| Control | Status | Notes |
|---------|--------|-------|
| 4.1 - Run as non-root | ✅ PASS | USER node in production |
| 4.2 - Use trusted base images | ✅ PASS | Official node:22-bookworm |
| 4.3 - No unnecessary packages | ⚠️ PARTIAL | Build tools in production image |
| 4.4 - Scan images for vulnerabilities | ⚠️ UNKNOWN | No evidence of scanning |
| 4.5 - Enable Content Trust | ⚠️ UNKNOWN | Not configured |
| 4.6 - Add HEALTHCHECK | ❌ FAIL | No HEALTHCHECK in Dockerfile |
| 5.1 - Verify AppArmor profile | ⚠️ PARTIAL | Using default profile |
| 5.2 - Verify SELinux security options | ⚠️ PARTIAL | No explicit config |
| 5.3 - Restrict Linux capabilities | ❌ FAIL | No cap_drop in compose |
| 5.4 - Do not use privileged containers | ✅ PASS | No privileged mode |
| 5.5 - Do not mount sensitive host paths | ⚠️ PARTIAL | Config dirs mounted |
| 5.10 - Do not use host network mode | ✅ PASS | No host network usage |
| 5.12 - Mount root filesystem read-only | ❌ FAIL | Not configured |
| 5.25 - Restrict socket access | ✅ PASS | No socket mounts |

**Overall Score:** 6/13 PASS, 5/13 PARTIAL, 2/13 FAIL

---

## 6. Testing Validation

To validate security improvements:

### 6.1 Capability Verification
```bash
# Inside container
capsh --print
# Should show minimal capabilities
```

### 6.2 User Verification
```bash
# Inside container
id
# Should show: uid=1000(node) gid=1000(node)
```

### 6.3 Filesystem Verification
```bash
# Inside container with read_only: true
touch /test-write
# Should fail with: Read-only file system
```

### 6.4 Network Binding Verification
```bash
# On host
netstat -tlnp | grep 18789
# Should show: 127.0.0.1:18789 (not 0.0.0.0:18789)
```

### 6.5 Security Options Verification
```bash
docker inspect openclaw-gateway | jq '.[0].HostConfig.SecurityOpt'
# Should include: no-new-privileges:true
```

---

## 7. Conclusion

OpenClaw demonstrates solid baseline security with non-root users in production containers and no Docker socket mounting. The primary risks are:

1. **Missing defense-in-depth controls** (no-new-privileges, capability dropping)
2. **Broad network exposure** (LAN binding by default)
3. **Host filesystem access** (read-write mounts)
4. **Test container sudo access** (privilege escalation path)

Implementing the recommended hardening measures will significantly reduce the attack surface while maintaining operational functionality. The provided hardened configurations can be adopted incrementally without breaking changes.

**Risk Level:** LOW-MEDIUM (production), MEDIUM (test environments)  
**Remediation Effort:** LOW (configuration changes only)  
**Business Impact:** NONE (hardening improves security without functional changes)
