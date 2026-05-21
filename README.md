# GooseRelay VPS Setup Guide


Deployment and operational guide for GooseRelayVPN using:

- Google Apps Script and VPS Tunnel
- Caddy Reverse Proxy
- Docker
- Cloudflare DNS
- VPS Infrastructure
- SOCKS5 Client Routing

This repository documents my personal journey building and debugging a Google Apps Script relay tunnel to a VPS server.

# Original Project

Original GooseRelayVPN project:

https://github.com/Kianmhz/GooseRelayVPN &

Forked version:

https://github.com/T2HASH/gooserelayVpN

## You can also visit my website where i explained the step-by-step with details:
```
https://datatodeploy.com/google-apps-script-vpn-tunnel/
```

All source code credits belong to the original project creator and contributors.

## The path is as follows:

```
Your Browser/App -> Firefox, Telegram, SSH, curl, etc.
  |
  ↓ -> SOCKS5
  |
Goose-client -> Local proxy 127.0.0.1:1080
  |
  ↓ -> AES-256-GCM encryption + optional ZSTD compression
  |
TLS to Google infrasturcture ->SNI= www.google.com,Host= script.google.com
  ↓
Google Apss Script -> Tiny forwarder Cannot decrypt
  |
  ↓ -> Raw encrypted bytes
  |
Caddy reverse proxy
  ↓
GooseRelay server on Ubuntu VPS -> Port 8443
  |
  ↓ -> Decrypt traffic Opens real TCP connection
  |
Internet -> Websites/APIs
```

The traffic is then:

- Compressed
- Encrypted with AES-256-GCM
- Sent via HTTP POST to a Google Apps Script endpoint
- Forwarded to a VPS
- Decrypted
- Lastly sent to the target destination

## Repository Structure

.
├── README.md
├── blog/
├── caddy/
├── config-examples/
├── docker/
├── scripts/
└── systemd/


## Features Covered
- VPS deployment
- Dockerized Caddy reverse proxy
- Cloudflare DNS setup
- HTTPS termination
- Apps Script deployment
- SOCKS5 routing
- macOS setup
- Multiple Apps Script deployments
- Relay troubleshooting
- Reverse proxy debugging
- Docker networking 
- Static Go builds

## VPS Requirements
- Ubuntu 22.04
- 1 GB RAM minimum
- Docker installed
- Domain connected to Cloudflare

## Docker Setup
Minimal Docker Compose example:

```
services:
  caddy:
    image: caddy:alpine
    container_name: caddy
    restart: unless-stopped

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

### Start:

```
docker compose up -d
```

## Reverse Proxy Setup
Example Caddyfile:

```
relay.example.com {
	
	reverse_proxy 127.17.0.1:8443

}
```

Important:
- `127.0.0.1` inside Docker containers points to the container itself.
- `127.17.0.1` is typically the Docker bridge gateway IP.

Get Docker bridge gateway:

```
docker network inspect bridge
```

## Validate and Reload Caddy
Validate first:

```
docker exec caddy caddy validate --config /etc/caddy/Caddyfile
```

The reload:

```
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

## GooseRelay Server Build
Clone upstream source:

```
git clone https://github.com/T2HASH/gooserelayVpN.git
```

Build static binary:

```
CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" \
-o bin/goose-server ./cmd/server
```

Verify static build:

```
ldd bin/goose-server
```

Expected:

```
not a dynamic executable
```

## Runtime Directory Layout
Recommended structure:

```
/srv/gooserelay/
├── source/
├── runtime/
└── config/
```

## Running GooseRelay Server

```
sudo -u goose /srv/gooserelay/runtime/goose-server \
-config /srv/gooserelay/config/server_config.json
```

Health check:

```
curl http://127.0.0.1:8443/healthz
```

Expected response:

```
{"ok":true}
```

## Google Apps Script Setup
1. Open https://script.google.com
2. Create new project
3. Paste Code.gs
4. Deploy as Web app

Settings:
- Execute as: me
- Access: anyone

Result:
```
https://script.google.com/macros/s/XXXX/exec
```

## Multiple Deployments
Important:
- Multiple .gs files inside one project is NOT recommended.
- Create separate Apps Script projectes instead.

Recommended:
```
Project A → URL A
Project B → URL B
Project C → URL C
```

## Example Client Config
```
{
  "socks_host": "0.0.0.0",
  "socks_port": 1080,

  "tunnel_key": "REPLACE_WITH_KEY",

  "relay_urls": [
    "https://script.google.com/macros/s/DEPLOYMENT_A/exec",
    "https://script.google.com/macros/s/DEPLOYMENT_B/exec"
  ]
}
```

## SOCKS5 Usage
Example for curl test:
```
curl --socks5-hostname 127.0.0.1:1080 https://example.com
```

## Safari / Browser Usage
macOS:
- System settings
- Network
- Wi-Fi
- Details
- Proxies
- SOCKS5 Proxy

Set:
- Host: 127.0.0.1
- Port: 1080

## SOCKS Listener Not Reachable
Verify:
```
lsof -nP -iTCP:1080 -sTCP:LISTEN
```

If sharing on LAN:
```
"socks_host": "0.0.0.0"
```

## Performance Expectations
This architecture prioritizes:

stealth,
survivability,
transport camouflage.

It does NOT provide:

low latency,
gaming-grade performance,
WireGuard-class throughput.

Typical issues:

long polling,
Apps Script throttling,
Google-side latency,
occasional HTTP2 resets.

## Security Notes

Before deploying, I manually reviewd:

subprocess execution, shell spawning, cryptography usage, nonce handling, AES-GCM implementation. 
```
grep -R "os/exec" .
grep -R "exec.Command" .
grep -R "NewGCM" .
grep -R "crypto/rand" .
```

## Disclaimer
This repository is for:

- Educational purposes.
- Networking research.
- Deployment documentation.
- Infrastructure experimentation.

Users are responsible for complying with local laws and platform policies.



