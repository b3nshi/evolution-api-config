# Troubleshooting Guide

This document covers common issues and their solutions when deploying Evolution API with Docker.

## 🔥 Firewall Configuration

### Problem: Docker Bypassing UFW (Uncomplicated Firewall)

When running Docker on a Raspberry Pi (or any Linux system) with UFW enabled, you may notice that Docker containers are accessible from the network even when UFW rules should block them. This happens because Docker manages its own iptables rules that take precedence over UFW.

#### Why This Happens

Docker, by default, creates its own iptables rules to manage container networking. These rules are inserted **before** UFW's rules in the iptables chain, which means:

1. Docker adds permissive rules like `ALLOW ANY` for exposed ports
2. UFW rules are processed after Docker's rules
3. Traffic is allowed by Docker before UFW can block it
4. This effectively bypasses your firewall configuration

**Example**: Even if you set `ufw deny 8089`, Docker's rule `-A DOCKER -p tcp --dport 8089 -j ACCEPT` will allow traffic before UFW can deny it.

#### Solution: Configure Docker to Respect UFW

To make Docker respect UFW firewall rules, you need to tell Docker to stop managing iptables directly.

##### Step 1: Configure Docker Daemon

Edit the Docker daemon configuration file:

```bash
sudo nano /etc/docker/daemon.json
```

**Note**: If the file doesn't exist or is empty, create it.

##### Step 2: Disable Docker's iptables Management

Add or modify the `iptables` setting. The file should look like this:

```json
{
  "iptables": false
}
```

**Important**: If the file already contains other configuration options, add `"iptables": false` to the existing JSON object, separated by commas:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "iptables": false
}
```

##### Step 3: Restart Docker

After saving the file, restart the Docker service:

```bash
sudo systemctl restart docker
```

**Warning**: After restarting Docker with `iptables: false`, existing containers may lose network connectivity. You'll need to restart your containers:

```bash
cd /path/to/evolution-api
docker-compose down
docker-compose up -d
```

##### Step 4: Configure UFW Rules

Now you can use UFW to control access to your Docker ports:

```bash
# Allow access from specific IP (recommended)
sudo ufw allow from 192.168.1.0/24 to any port 8089
sudo ufw allow from 192.168.1.0/24 to any port 3006

# Or allow from anywhere (less secure)
sudo ufw allow 8089/tcp
sudo ufw allow 3006/tcp

# Or deny and only allow specific IPs
sudo ufw deny 8089
sudo ufw allow from 192.168.1.100 to any port 8089
```

##### Step 5: Verify UFW Status

Check that UFW is active and your rules are in place:

```bash
sudo ufw status verbose
```

#### Important Considerations

⚠️ **Security Implications**:

1. **With `iptables: false`**: 
   - Docker won't automatically set up port forwarding
   - You must manually configure iptables or use UFW
   - More control over firewall rules
   - Requires manual management of Docker networking

2. **Without `iptables: false`** (default):
   - Docker automatically manages networking
   - Easier setup, but less control
   - Docker rules bypass UFW
   - May expose ports unintentionally

#### Alternative Approaches

If you prefer not to disable Docker's iptables management, consider these alternatives:

##### Option 1: Use Docker's Built-in Firewall Rules

Configure Docker networks with specific IP ranges and use Docker's networking features:

```yaml
networks:
  evolution-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

##### Option 2: Use a Reverse Proxy with Access Control

Set up nginx or traefik as a reverse proxy with IP whitelisting:

```nginx
# nginx configuration example
location / {
    allow 192.168.1.0/24;
    deny all;
    proxy_pass http://evolution_api:8080;
}
```

##### Option 3: Use Docker's `--publish-addr` Option

When starting containers, bind to specific interfaces:

```yaml
ports:
  - "192.168.1.58:8089:8080"  # Only bind to specific IP
```

However, this requires knowing your IP in advance and may not work well with dynamic IPs.

#### Verification Steps

After configuring, verify the setup:

1. **Check Docker configuration**:
   ```bash
   cat /etc/docker/daemon.json
   ```

2. **Check UFW rules**:
   ```bash
   sudo ufw status numbered
   ```

3. **Test from another machine**:
   ```bash
   # Should work if allowed by UFW
   curl http://192.168.1.58:8089/instance/fetchInstances -H "apikey: YOUR_KEY"
   
   # Should fail if blocked by UFW
   curl http://192.168.1.58:8089/instance/fetchInstances -H "apikey: YOUR_KEY"
   ```

4. **Check iptables rules**:
   ```bash
   sudo iptables -L -n -v | grep 8089
   ```

#### Troubleshooting

**Issue**: Containers can't communicate after setting `iptables: false`

**Solution**: You may need to manually add iptables rules or ensure Docker networks are properly configured. Consider using `docker network` commands to manage connectivity.

**Issue**: UFW rules not working even after configuration

**Solution**: 
1. Ensure UFW is enabled: `sudo ufw enable`
2. Check that Docker was restarted: `sudo systemctl status docker`
3. Verify containers were recreated: `docker-compose ps`
4. Check iptables rules order: `sudo iptables -L -n -v --line-numbers`

**Issue**: Can't access services from network after configuration

**Solution**:
1. Check if UFW is blocking: `sudo ufw status`
2. Verify port bindings: `docker-compose ps`
3. Test locally first: `curl http://localhost:8089`
4. Check Docker logs: `docker-compose logs api`

#### Best Practices

1. **For Production**:
   - Use `iptables: false` for better security control
   - Configure UFW to only allow trusted IPs/subnets
   - Use a reverse proxy with SSL/TLS
   - Regularly audit firewall rules

2. **For Development**:
   - Default Docker iptables management is usually fine
   - Use UFW to block specific ports if needed
   - Consider using VPN for remote access

3. **For Raspberry Pi on Local Network**:
   - Allow access from your local subnet (e.g., `192.168.1.0/24`)
   - Block external access if not needed
   - Monitor logs for unauthorized access attempts

#### Additional Resources

- [Docker iptables Documentation](https://docs.docker.com/network/iptables/)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)

---

## Other Common Issues

### Port Already in Use

**Error**: `Error response from daemon: Ports are not available`

**Solution**:
```bash
# Find what's using the port
sudo lsof -i :8089
sudo netstat -tulpn | grep 8089

# Stop the conflicting service or change ports in docker-compose.yml
```

### Database Connection Issues

See the main [README.md](../README.md#troubleshooting) for database troubleshooting.

### Container Won't Start

```bash
# Check logs
docker-compose logs [service-name]

# Check container status
docker-compose ps

# Restart specific service
docker-compose restart [service-name]
```

### Permission Denied Errors

```bash
# Fix Docker socket permissions (if needed)
sudo usermod -aG docker $USER
newgrp docker
```

---

For more troubleshooting help, refer to the main [README.md](../README.md) or check the [Evolution API GitHub Issues](https://github.com/EvolutionAPI/evolution-api/issues).

