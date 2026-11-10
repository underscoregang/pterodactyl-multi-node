# Pterodactyl Panel — Multi-Node Setup Guide

A practical guide to running **one Pterodactyl Panel** with **multiple Wings nodes** hosted on VPS servers in different regions.

> **Scope:** Pterodactyl Panel/Wings 1.x on Ubuntu 22.04/24.04. Confirm version-specific requirements in the [official documentation](https://docs.pterodactyl.io/).

## 1. Architecture

```text
                      https://panel.example.com
                       Pterodactyl Panel
                              |
           +------------------+------------------+
           |                  |                  |
   sg.example.com      us.example.com     ca.example.com      in.example.com
   Singapore VPS       USA VPS            Canada VPS          India VPS
   Wings + Docker      Wings + Docker     Wings + Docker      Wings + Docker
   API: 8080           API: 8080          API: 8080           API: 8080
   SFTP: 2022          SFTP: 2022         SFTP: 2022          SFTP: 2022
```

Each node runs its own Docker and Wings instance. The Panel controls all nodes from a single web interface. You can run the Panel on one of the four VPS servers (4 VPS total), or on a separate host (5 VPS total). **USA is already part of North America**; Canada is used here as a distinct fourth region.

### Example inventory

| Region | Node name | FQDN | Wings API | SFTP |
|---|---|---|---|---|
| Singapore | SG-01 | `sg.example.com` | `8080/TCP` | `2022/TCP` |
| USA | US-01 | `us.example.com` | `8080/TCP` | `2022/TCP` |
| Canada | CA-01 | `ca.example.com` | `8080/TCP` | `2022/TCP` |
| India | IN-01 | `in.example.com` | `8080/TCP` | `2022/TCP` |

**Ports do not conflict across separate VPS hosts/IP addresses.** All four nodes can use `8080` and `2022`. A conflict occurs only when two services attempt to bind the same IP, port, and protocol on the *same host/network namespace*.

## 2. Prerequisites

- One working Pterodactyl Panel 1.x installation with administrator access.
- A VPS per region, preferably KVM virtualization, with a supported 64-bit Linux OS and Docker-compatible kernel.
- SSH/sudo access, sufficient RAM, CPU, disk space, and public network connectivity.
- A DNS name for each node, pointing to its VPS public IP.
- A working TLS certificate for each Wings API hostname when using HTTPS.
- Inbound network rules at both the VPS OS and cloud provider firewall.

Use example hostnames and addresses as placeholders; replace them with your own. Keep Panel-generated tokens and `config.yml` secret.

## 3. Configure DNS

Create one `A` record per VPS in your DNS provider:

| Type | Host | Value |
|---|---|---|
| A | `sg` | `<SG_PUBLIC_IP>` |
| A | `us` | `<US_PUBLIC_IP>` |
| A | `ca` | `<CA_PUBLIC_IP>` |
| A | `in` | `<IN_PUBLIC_IP>` |

For a straightforward Wings installation with direct HTTPS on port 8080, use **DNS-only** rather than a Cloudflare HTTP proxy. Verify from your workstation or server:

```bash
getent hosts sg.example.com
```

Repeat for the other node names.

## 4. Prepare each VPS

SSH into the target VPS (Singapore example):

```bash
ssh root@<SG_PUBLIC_IP>
cat /etc/os-release
uname -m
systemd-detect-virt
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates gnupg
```

Check that the VPS supports Docker. Some container-based VPS products restrict the required kernel/container features.

## 5. Install Docker on each node

Follow the [Docker Engine Ubuntu installation instructions](https://docs.docker.com/engine/install/ubuntu/) for production. A convenient installation method for a fresh VPS is:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo systemctl enable --now docker
sudo docker version
sudo docker run --rm hello-world
```

Review scripts before executing them, especially on production systems.

## 6. Download Wings

```bash
sudo mkdir -p /etc/pterodactyl

case "$(uname -m)" in
  x86_64) WINGS_ARCH=amd64 ;;
  aarch64) WINGS_ARCH=arm64 ;;
  *) echo 'Unsupported CPU architecture'; exit 1 ;;
esac

sudo curl -fL \
  -o /usr/local/bin/wings \
  "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_${WINGS_ARCH}"
sudo chmod +x /usr/local/bin/wings
/usr/local/bin/wings version
```

For a controlled deployment, pin and verify a specific compatible Wings release rather than using `latest`.

## 7. Create locations and nodes in the Panel

1. Log in to the Panel as an administrator.
2. Go to **Admin → Locations** and create `Singapore`, `USA`, `Canada`, and `India`.
3. Go to **Admin → Nodes → Create New**.
4. For SG-01, enter values similar to:

| Setting | Example |
|---|---|
| Name | `SG-01` |
| Location | `Singapore` |
| FQDN | `sg.example.com` |
| Communicate over SSL | `Use SSL` |
| Behind Proxy | `Not Behind Proxy` (direct Wings) |
| Daemon Port | `8080` |
| Daemon SFTP Port | `2022` |
| Server File Directory | `/var/lib/pterodactyl/volumes` |
| Total Memory | `8192` MB (adjust to available RAM) |
| Total Disk | `51200` MB (adjust to available disk) |

5. Save the node and repeat for each region using its own FQDN and capacities.

> Do not allocate all RAM to game servers if the same VPS also runs the Panel, database, Redis, or other services. Reserve headroom for the OS and supporting services.

## 8. Configure HTTPS for Wings

For a direct HTTPS Wings API on `sg.example.com:8080`, obtain a TLS certificate on the Singapore VPS. Ensure port 80 is reachable and not occupied while using Certbot standalone:

```bash
sudo apt install -y certbot
sudo certbot certonly --standalone \
  -d sg.example.com \
  --agree-tos \
  -m admin@example.com
```

Certificate files typically appear at:

```text
/etc/letsencrypt/live/sg.example.com/fullchain.pem
/etc/letsencrypt/live/sg.example.com/privkey.pem
```

In the Panel-generated Wings configuration, confirm that TLS is enabled and `api.ssl.cert` / `api.ssl.key` point to those files. Wings runs as root in the service example below, so it can read the private key.

Certbot renewal must be tested. Ensure Wings reloads/restarts after certificate renewal so it picks up the new certificate. For example, configure an appropriate Certbot deploy hook for your environment and test renewal with:

```bash
sudo certbot renew --dry-run
```

If the Panel and Wings share a host or another web server already uses port 80, use a compatible DNS challenge or existing reverse-proxy/certificate strategy instead of blindly running standalone Certbot.

## 9. Link Wings to the Panel

Open **Admin → Nodes → SG-01 → Configuration** in the Panel.

**Option A — Auto-deploy:** Generate a deployment token in the Panel and run the *exact command the Panel displays* on the corresponding VPS. Tokens are sensitive and may expire.

**Option B — Manual:** Copy the generated YAML into:

```bash
sudo nano /etc/pterodactyl/config.yml
sudo chmod 600 /etc/pterodactyl/config.yml
```

Do **not** copy another node's `config.yml`: every node needs its own identity and token. Check the node FQDN, Panel URL, TLS paths, API port, and SFTP port.

## 10. Configure firewalls

Typical inbound ports on **each node**:

| Port | Protocol | Purpose |
|---|---|---|
| `22` | TCP | SSH administration |
| `80` | TCP | HTTP-01 certificate issuance/renewal if used |
| `8080` | TCP | Wings API (HTTPS in this guide) |
| `2022` | TCP | Wings SFTP |
| Game allocations | TCP and/or UDP | Depends on each game |

For UFW (adjust the SSH port and game ports as needed):

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 2022/tcp
sudo ufw allow 25565/tcp
sudo ufw status
```

**Important:** Verify your SSH allow rule and cloud firewall before running `sudo ufw enable`, or you may lock yourself out. Prefer limiting management/API access to trusted IPs where practical, while allowing the connections needed by the Panel and clients. Open only game allocation ports you actually use; some games need UDP as well as TCP.

## 11. Test Wings interactively

```bash
sudo /usr/local/bin/wings --debug
```

Inspect the output for Docker, TLS, configuration, and Panel connection errors. Press **Ctrl+C** to stop the foreground process before enabling the service.

## 12. Create the systemd service

```bash
sudo nano /etc/systemd/system/wings.service
```

Paste:

```ini
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
ExecStart=/usr/local/bin/wings
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wings
sudo systemctl status wings
sudo journalctl -u wings -n 100 --no-pager
```

Expected service state: `active (running)`.

## 13. Add game-server allocations

In the Panel, open **Admin → Nodes → SG-01 → Allocations**. Add the node's IP and the ports you want game servers to use, for example:

| Node | Allocation IP | Example ports |
|---|---|---|
| SG-01 | `<SG_PUBLIC_IP>` | `25565–25575` |
| US-01 | `<US_PUBLIC_IP>` | `25565–25575` |
| CA-01 | `<CA_PUBLIC_IP>` | `25565–25575` |
| IN-01 | `<IN_PUBLIC_IP>` | `25565–25575` |

The same game port numbers can be reused on **different VPS IPs**. Each allocation on a given node must be unique. If a VPS uses NAT, configure the appropriate internal allocation IP and public port forwarding. Open the relevant game ports in the firewalls.

## 14. Validate each region

- [ ] DNS resolves to the correct node IP.
- [ ] Docker starts and can run a test container.
- [ ] Wings configuration is unique to this node.
- [ ] TLS certificate matches the node FQDN.
- [ ] Wings systemd service is active.
- [ ] Panel shows the node as connected/healthy.
- [ ] Allocations exist and match allowed firewall ports.
- [ ] A test game server starts and accepts connections.
- [ ] TLS renewal and restart/reload are tested.

Repeat the same process for US-01, CA-01, and IN-01.

## 15. Troubleshooting

### Is port 8080 or 2022 already in use?

```bash
sudo ss -tulpn | grep -E ':(8080|2022)\b'
```

If the port is in use on **that same VPS**, identify the process and choose a non-conflicting setup. Usage on another VPS is not a conflict.

### Wings will not start

```bash
sudo systemctl status wings
sudo journalctl -u wings -n 100 --no-pager
sudo systemctl status docker
```

Check `config.yml`, Docker health, file permissions, certificate paths, and connectivity to the Panel.

### Node shows offline

Verify DNS, SSL hostname and expiry, firewall rules, the Wings listening port, and that the Panel can reach the Wings API. For direct TLS:

```bash
curl -Iv https://sg.example.com:8080
```

An HTTP error response may still prove that the TLS endpoint is reachable; connection or certificate errors are the main things to investigate.

### SFTP connection fails

Confirm Wings is listening on `2022/TCP`, that the port is allowed through both firewalls, and that the Panel's node SFTP setting matches.

### Game server cannot be reached

Check the allocation IP/port, TCP versus UDP requirements, host/cloud firewall rules, and whether the game process is actually running. For NAT setups, also verify port forwarding.

## 16. Security and operations

- Use unique credentials and Panel-generated configurations for each node.
- Protect `config.yml` and private TLS keys; never publish them to GitHub.
- Keep Docker, Wings, and the OS updated with tested maintenance procedures.
- Back up Panel data and game-server data; a Panel backup alone is not a backup of all node volumes.
- Monitor disk, RAM, CPU, networking, and certificate expiration separately for every region.
- Consider hosting the Panel separately for better isolation as the service grows.

## References

- [Pterodactyl Wings installation](https://docs.pterodactyl.io/v1/wings/installing)
- [Pterodactyl documentation](https://docs.pterodactyl.io/)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Pterodactyl Wings releases](https://github.com/pterodactyl/wings/releases)
