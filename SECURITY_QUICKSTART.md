# Security Hardening Quick Reference

Quick checklist for deploying hardened OpenClaw configurations.

## Pre-Deployment Checklist

### Environment Setup
- [ ] Docker 20.10+ installed
- [ ] docker compose v2+ installed
- [ ] OpenSSL or Python 3 available (for token generation)
- [ ] Non-root user configured for systemd services (if using)

### Configuration Files Ready
- [ ] `Dockerfile.hardened` reviewed
- [ ] `docker-compose.hardened.yml` reviewed
- [ ] Environment variables prepared
- [ ] Firewall rules configured (if exposing to network)

---

## Quick Start Commands

### 1. Build Hardened Image
```bash
cd /path/to/openclaw
docker build -f Dockerfile.hardened -t openclaw:hardened .
```

**Verification:**
```bash
docker run --rm openclaw:hardened id
# Expected: uid=1000(node) gid=1000(node)
```

### 2. Generate Secure Token
```bash
# Option 1: OpenSSL
export OPENCLAW_GATEWAY_TOKEN="$(openssl rand -hex 32)"

# Option 2: Python
export OPENCLAW_GATEWAY_TOKEN="$(python3 -c 'import secrets; print(secrets.token_hex(32))')"

# Save token securely
echo "$OPENCLAW_GATEWAY_TOKEN" > ~/.openclaw-token
chmod 600 ~/.openclaw-token
```

### 3. Prepare Directories
```bash
export OPENCLAW_CONFIG_DIR="$HOME/.openclaw"
export OPENCLAW_WORKSPACE_DIR="$HOME/.openclaw/workspace"

mkdir -p "$OPENCLAW_CONFIG_DIR"
mkdir -p "$OPENCLAW_WORKSPACE_DIR"
chmod 700 "$OPENCLAW_CONFIG_DIR"
```

### 4. Deploy with Hardened Compose
```bash
# Set environment
export OPENCLAW_IMAGE=openclaw:hardened
export OPENCLAW_GATEWAY_BIND=loopback

# Validate configuration
docker compose -f docker-compose.hardened.yml config

# Deploy
docker compose -f docker-compose.hardened.yml up -d openclaw-gateway
```

### 5. Verify Deployment
```bash
# Check status
docker compose -f docker-compose.hardened.yml ps

# Check logs
docker compose -f docker-compose.hardened.yml logs -f openclaw-gateway

# Verify security options
docker inspect openclaw-gateway | jq '.[0].HostConfig.SecurityOpt'
# Expected: ["no-new-privileges:true"]

# Verify port binding
netstat -tlnp | grep 18789
# Expected: 127.0.0.1:18789 (NOT 0.0.0.0:18789)
```

---

## Security Verification Tests

### Test 1: Non-Root User ✅
```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway id
```
**Expected:** `uid=1000(node) gid=1000(node)`

### Test 2: Read-Only Filesystem ✅
```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway touch /test
```
**Expected:** `touch: cannot touch '/test': Read-only file system`

### Test 3: No New Privileges ✅
```bash
docker inspect openclaw-gateway | jq '.[0].HostConfig.SecurityOpt'
```
**Expected:** `["no-new-privileges:true"]`

### Test 4: Dropped Capabilities ✅
```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway sh -c 'capsh --print 2>/dev/null || echo "capsh not available (expected)"'
```
**Expected:** Minimal or no capabilities listed

### Test 5: Localhost Binding ✅
```bash
ss -tlnp | grep 18789
```
**Expected:** `127.0.0.1:18789` (NOT `0.0.0.0:18789`)

### Test 6: Network Connectivity (Should Work) ✅
```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway curl -s https://www.cloudflare.com/cdn-cgi/trace | grep ip
```
**Expected:** Shows public IP (internet access working)

---

## Common Operations

### View Logs
```bash
docker compose -f docker-compose.hardened.yml logs -f openclaw-gateway
```

### Restart Service
```bash
docker compose -f docker-compose.hardened.yml restart openclaw-gateway
```

### Update Image
```bash
# Rebuild
docker build -f Dockerfile.hardened -t openclaw:hardened .

# Recreate containers
docker compose -f docker-compose.hardened.yml up -d --force-recreate
```

### Run CLI Commands
```bash
docker compose -f docker-compose.hardened.yml run --rm openclaw-cli channels status
```

### Access Shell (Debugging)
```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway sh
```

---

## Troubleshooting

### Gateway Won't Start

**Check 1: Volume Permissions**
```bash
ls -la "$OPENCLAW_CONFIG_DIR"
# Should be owned by uid 1000 or current user

# Fix if needed
sudo chown -R 1000:1000 "$OPENCLAW_CONFIG_DIR"
```

**Check 2: Read-Only Filesystem Issues**
```bash
# Check logs for write errors
docker compose -f docker-compose.hardened.yml logs openclaw-gateway | grep "EROFS\|Read-only"

# If found, add tmpfs for the path
# Edit docker-compose.hardened.yml and add to tmpfs section
```

**Check 3: Port Conflicts**
```bash
# Check if port is in use
ss -tlnp | grep 18789

# Kill conflicting process or change port
export OPENCLAW_GATEWAY_PORT=18790
```

### Cannot Connect from Another Machine

**Expected Behavior:** Ports bound to localhost only (security feature)

**Solution 1: SSH Tunnel (Recommended)**
```bash
ssh -L 18789:localhost:18789 user@openclaw-host
# Then connect to localhost:18789 on your machine
```

**Solution 2: Change Binding (Less Secure)**
```bash
# Edit docker-compose.hardened.yml
# Change: "127.0.0.1:18789:18789"
# To:     "18789:18789"
# And set: OPENCLAW_GATEWAY_BIND=lan

# IMPORTANT: Ensure firewall rules are in place!
```

### Permission Denied in Container
```bash
# Check volume ownership
docker compose -f docker-compose.hardened.yml exec openclaw-gateway ls -la /home/node/.openclaw

# Fix host-side ownership
sudo chown -R 1000:1000 "$OPENCLAW_CONFIG_DIR"
sudo chown -R 1000:1000 "$OPENCLAW_WORKSPACE_DIR"
```

### Write Errors to tmpfs
```bash
# Check tmpfs size
docker compose -f docker-compose.hardened.yml exec openclaw-gateway df -h | grep tmpfs

# Increase size in docker-compose.hardened.yml
# Change: /tmp:noexec,nosuid,nodev,size=100m
# To:     /tmp:noexec,nosuid,nodev,size=500m
```

---

## Rollback Procedure

If issues arise, rollback to standard configuration:

```bash
# Stop hardened deployment
docker compose -f docker-compose.hardened.yml down

# Start standard deployment
export OPENCLAW_IMAGE=openclaw:local
docker compose up -d openclaw-gateway
```

---

## Environment Variables Reference

### Required
- `OPENCLAW_GATEWAY_TOKEN` - Authentication token (64-char hex)
- `OPENCLAW_CONFIG_DIR` - Config directory path (e.g., `$HOME/.openclaw`)
- `OPENCLAW_WORKSPACE_DIR` - Workspace path (e.g., `$HOME/.openclaw/workspace`)

### Optional
- `OPENCLAW_IMAGE` - Docker image tag (default: `openclaw:local`)
- `OPENCLAW_GATEWAY_PORT` - Host port for gateway (default: `18789`)
- `OPENCLAW_BRIDGE_PORT` - Host port for bridge (default: `18790`)
- `OPENCLAW_GATEWAY_BIND` - Bind mode: `loopback` or `lan` (default: `loopback`)
- `CLAUDE_AI_SESSION_KEY` - Claude AI session key (optional)
- `CLAUDE_WEB_SESSION_KEY` - Claude Web session key (optional)
- `CLAUDE_WEB_COOKIE` - Claude Web cookie (optional)

---

## Security Monitoring

### Daily Checks
```bash
# Check running containers
docker ps | grep openclaw

# Check for security alerts
docker compose -f docker-compose.hardened.yml logs openclaw-gateway | grep -i "error\|warning\|security"

# Check port bindings
ss -tlnp | grep openclaw
```

### Weekly Checks
```bash
# Update base image
docker pull node:22-bookworm-slim

# Rebuild hardened image
docker build -f Dockerfile.hardened -t openclaw:hardened .

# Scan for vulnerabilities (if trivy installed)
trivy image openclaw:hardened
```

### Monthly Checks
- Review access logs
- Rotate authentication tokens
- Update systemd service files
- Review firewall rules
- Check for OpenClaw updates

---

## Additional Resources

- **Full Assessment:** `SECURITY_ASSESSMENT_DOCKER.md`
- **Executive Summary:** `SECURITY_SUMMARY.md`
- **Deployment Guide:** `docs/deployment/hardening.md`
- **Docker Docs:** https://docs.docker.com/engine/security/
- **CIS Benchmark:** https://www.cisecurity.org/benchmark/docker

---

## Support

**Issues:** Open GitHub issue with `security` label  
**Questions:** Check documentation or community forums  
**Security Concerns:** Follow responsible disclosure in `SECURITY.md`

---

*Last Updated: 2026-02-11*
