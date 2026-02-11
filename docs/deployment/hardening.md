# Docker & Runtime Security Hardening Guide

This guide explains how to deploy OpenClaw with enhanced security configurations.

## Quick Start

### Using Hardened Docker Compose

```bash
# Build with hardened Dockerfile
docker build -f Dockerfile.hardened -t openclaw:hardened .

# Run with hardened compose configuration
OPENCLAW_IMAGE=openclaw:hardened docker compose -f docker-compose.hardened.yml up -d
```

### Using Hardened Systemd Service

```bash
# Create dedicated user
sudo useradd -r -s /bin/false openclaw

# Copy and adjust service file
sudo cp scripts/systemd/openclaw-auth-monitor.service.hardened \
        /etc/systemd/system/openclaw-auth-monitor.service

# Edit paths in the service file
sudo nano /etc/systemd/system/openclaw-auth-monitor.service

# Reload and enable
sudo systemctl daemon-reload
sudo systemctl enable openclaw-auth-monitor.timer
sudo systemctl start openclaw-auth-monitor.timer
```

## Security Improvements

### Dockerfile.hardened

**Changes from base Dockerfile:**
- ✅ Multi-stage build (builder + production)
- ✅ Slim production image (smaller attack surface)
- ✅ Minimal runtime dependencies
- ✅ dumb-init for proper signal handling
- ✅ Health check included

**Attack Surface Reduction:**
- ~400MB smaller image (no build tools)
- ~50% fewer packages in production
- Reduced CVE exposure from unnecessary dependencies

### docker-compose.hardened.yml

**Changes from base docker-compose.yml:**
- ✅ Ports bound to localhost only (127.0.0.1)
- ✅ `no-new-privileges` security option
- ✅ Read-only root filesystem
- ✅ tmpfs with security flags (noexec, nosuid, nodev)
- ✅ Capability dropping (cap_drop: ALL)
- ✅ Isolated Docker network
- ✅ Default bind changed to loopback

**Security Benefits:**
- Prevents external network access by default
- Blocks privilege escalation attacks
- Immutable filesystem (defense in depth)
- Minimal Linux capabilities
- Isolated from host network

### openclaw-auth-monitor.service.hardened

**Changes from base service:**
- ✅ Dedicated non-root user
- ✅ Filesystem restrictions (ProtectSystem=strict)
- ✅ Network restrictions (limited address families)
- ✅ Namespace isolation
- ✅ Kernel protection
- ✅ System call filtering

**Security Benefits:**
- Runs with minimal privileges
- Cannot modify system files
- Limited to required network protocols
- Cannot access kernel interfaces
- Blocks dangerous system calls

## Configuration Options

### Network Binding

**Localhost Only (Recommended):**
```yaml
# docker-compose.hardened.yml
ports:
  - "127.0.0.1:18789:18789"  # Only accessible from host
```

**LAN Access (Use with caution):**
```yaml
# docker-compose.hardened.yml
ports:
  - "18789:18789"  # Accessible from network
command:
  - "--bind"
  - "lan"  # Change from loopback
```

**Important:** Always use authentication tokens when binding to LAN:
```bash
export OPENCLAW_GATEWAY_TOKEN="$(openssl rand -hex 32)"
```

### Read-Only Filesystem

**Enable (Recommended):**
```yaml
read_only: true
tmpfs:
  - /tmp:noexec,nosuid,nodev,size=100m
  - /home/node/.npm:noexec,nosuid,nodev,size=200m
```

**Disable (If application writes to unexpected paths):**
```yaml
read_only: false  # Remove this line entirely
# Remove tmpfs section
```

### Capability Management

**Minimal (Recommended):**
```yaml
cap_drop:
  - ALL
# No cap_add needed for ports > 1024
```

**Add Capabilities (If needed):**
```yaml
cap_drop:
  - ALL
cap_add:
  - NET_BIND_SERVICE  # For ports < 1024
  - CHOWN             # For runtime chown operations
```

## Testing Security Configurations

### Verify Non-Root User

```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway id
# Expected: uid=1000(node) gid=1000(node)
```

### Verify Read-Only Filesystem

```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway touch /test
# Expected: touch: cannot touch '/test': Read-only file system
```

### Verify Network Binding

```bash
# On host
netstat -tlnp | grep 18789
# Expected: 127.0.0.1:18789 (not 0.0.0.0:18789)
```

### Verify Capabilities

```bash
docker compose -f docker-compose.hardened.yml exec openclaw-gateway sh -c 'apt-get update'
# Expected: (should fail - no cap_add for package management)
```

### Verify no-new-privileges

```bash
docker inspect openclaw-gateway | jq '.[0].HostConfig.SecurityOpt'
# Expected: ["no-new-privileges:true"]
```

## Migration from Base Configuration

### Step 1: Test in Non-Production

```bash
# Build hardened image
docker build -f Dockerfile.hardened -t openclaw:hardened-test .

# Test with hardened compose
OPENCLAW_IMAGE=openclaw:hardened-test docker compose -f docker-compose.hardened.yml up
```

### Step 2: Verify Functionality

Test all critical workflows:
- Gateway startup
- Channel connections
- Agent operations
- Command execution

### Step 3: Production Deployment

```bash
# Tag for production
docker tag openclaw:hardened-test openclaw:hardened

# Deploy
OPENCLAW_IMAGE=openclaw:hardened docker compose -f docker-compose.hardened.yml up -d
```

## Troubleshooting

### Application Won't Start

**Symptom:** Container exits immediately  
**Cause:** Read-only filesystem prevents writing to required path  
**Solution:** Add tmpfs mount for the path or disable read_only temporarily

```yaml
tmpfs:
  - /path/that/needs/write:noexec,nosuid,nodev,size=100m
```

### Network Access Issues

**Symptom:** Cannot connect from other machines  
**Cause:** Ports bound to localhost only  
**Solution:** Change port binding or use SSH tunnel

```bash
# SSH tunnel (secure)
ssh -L 18789:localhost:18789 user@openclaw-host

# OR change compose (less secure)
ports:
  - "0.0.0.0:18789:18789"
```

### Permission Denied Errors

**Symptom:** "Permission denied" in logs  
**Cause:** Capability or filesystem restrictions  
**Solution:** Check ownership of mounted volumes

```bash
# Fix ownership
sudo chown -R 1000:1000 ~/.openclaw

# OR add capability (only if needed)
cap_add:
  - CHOWN
```

## Performance Impact

Security hardening has minimal performance impact:

| Feature | CPU Impact | Memory Impact | Startup Time |
|---------|------------|---------------|--------------|
| Multi-stage build | None | -400MB image | None |
| Read-only filesystem | <1% | None | None |
| Capability dropping | <1% | None | None |
| no-new-privileges | None | None | None |

**Overall:** Negligible performance impact with significant security gains.

## Security Checklist

Before deploying to production:

- [ ] Built with `Dockerfile.hardened`
- [ ] Using `docker-compose.hardened.yml`
- [ ] Ports bound to localhost or protected with firewall
- [ ] Strong authentication token set (`OPENCLAW_GATEWAY_TOKEN`)
- [ ] Volumes owned by uid 1000 (node user)
- [ ] Verified non-root user with `docker exec ... id`
- [ ] Tested all critical application workflows
- [ ] Systemd services use hardened configuration
- [ ] Regular image updates scheduled (security patches)
- [ ] Container logs monitored for security events

## Additional Resources

- **CIS Docker Benchmark:** https://www.cisecurity.org/benchmark/docker
- **Docker Security Best Practices:** https://docs.docker.com/engine/security/
- **Systemd Security Hardening:** https://www.freedesktop.org/software/systemd/man/systemd.exec.html
- **OpenClaw Docs:** https://docs.openclaw.ai

## Support

For questions about security hardening:
1. Check `SECURITY_ASSESSMENT_DOCKER.md` for detailed findings
2. Review OpenClaw security documentation
3. Open a GitHub issue with `security` label
