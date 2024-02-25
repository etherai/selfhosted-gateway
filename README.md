# Self-hosted Gateway
**Jump to [Getting Started](#getting-started)**
## Features and Benefits
- Docker native self-hosted alternative to Cloudflare Tunnels, Tailscale Funnel, ngrok and others.
- Entirely self-hosted and self-managed, includes local and remote tunneling components.
- No custom code, this project leverages existing battled tested FOSS components:
  - WireGuard
  - Nginx (Gateway)
  - Caddy (Client)
- Automatic client side HTTPS cert provisioning thanks to Caddy's automatic https.
- Remote client IPs passed to local container via proxy protocol
- Enable basic authentication by specifying env variable containig username and password
- Proxy generic TCP/UDP traffic to localhost with socat

## Video Overview & Setup Guide
<a href="http://www.youtube.com/watch?feature=player_embedded&v=VCH8-XOikQc" target="_blank">
 <img src="http://img.youtube.com/vi/VCH8-XOikQc/0.jpg" alt="Watch the video" width="560" height="315" border="10" />
</a>


## Overview

This project automates the provisioning of **Reverse Proxy-over-VPN (RPoVPN)** WireGuard tunnels with Caddy and NGINX. It is particularly well suited for exposing docker compose services defined in a `docker-compose` file to the public Internet. There's no code or APIs, just an ultra generic NGINX config and some short provisioning bash script. TLS certs are provisioned automatically with Caddy's Automatic HTTPS feature via Let's Encrypt or ZeroSSL.

## Use cases

1. **RPoVPN is a common strategy for remotely accessing applications self-hosted at home. It solves problems such as:**
  - Self-hosting behind double-NAT or via an ISP that does CGNAT (Starlink, Mobile Internet).
  - Inability to portforward on your local network due to insufficient access.
  - Having a dynamically allocated IP that may change frequently.

2. **Using RPoVPN is ideal for self-hosting from both a network security and privacy perspective:**
  - Obviates the need for a static IP or expose your home's public IP address to the world.
  - Utilizes advanced network isolation capabilities of Docker (thanks to Linux network namespaces) in order to isolate locally exposed services from your home network and other local docker services.
  - Built on open-source technologies (WireGuard, Caddy and NGINX).

## Getting Started

### Prerequisites
- Domain
  - Ability to create an `A` record for a domain name.
- Gateway 
  - A publically addressable Linux host to act as the `gateway`, typically a cloud VPS (Hetzner, Digital Ocean, etc..) with the following requirements:
  - SSH access
  - Ports 80/443 open (http/https)
  - The UDP port range listed by `cat /proc/sys/net/ipv4/ip_local_port_range` open to the Internet.
  - `docker`, `git` & `make` installed on the Gateway
- Client
  - An existing `docker-compose.yml` that you would like to expose to the Internet.
  - `docker`, `git` & `make` installed locally

### Steps
1. Point `*.mydomain.com` (DNS A Record) to the IPv4 & IPv6 address of your VPS Gateway host.

2. Connect to the `gateway` via SSH and setup the `gateway` service:
```console
foo@gateway:~$ git clone ... && cd selfhosted-gateway
foo@gateway:~/selfhosted-gateway$ make docker
foo@gateway:~/selfhosted-gateway$ make setup
foo@gateway:~/selfhosted-gateway$ make gateway
```

3. To generate a `link` docker compose snippet run the following commands:
```console
foo@local:~$ git clone ... && cd selfhosted-gateway
foo@local:~/selfhosted-gateway$ make docker
foo@local:~/selfhosted-gateway$ make link GATEWAY=root@123.456.789.101 FQDN=nginx.mydomain.com EXPOSE=nginx:80
  link:
    image: fractalnetworks/gateway-client:latest
    environment:
      LINK_DOMAIN: nginx.mydomain.com
      EXPOSE: nginx:80
      GATEWAY_CLIENT_WG_PRIVKEY: 4M7Ap0euzTxq7gTA/WIYIt3nU+i2FvHUc9eYTFQ2CGI=
      GATEWAY_LINK_WG_PUBKEY: Wipd6Pv7ttmII4/Oj82I5tmGZwuw6ucsE3G+hwsMR08=
      GATEWAY_ENDPOINT: 123.456.789.101:49185
    cap_add:
      - NET_ADMIN
```

4. Add the generated snippet to your `docker-compose.yml` file:
```yaml
version: '3.9'
services:
  nginx:
    image: nginx:latest
  link:
    image: fractalnetworks/gateway-client:latest
    environment:
      LINK_DOMAIN: nginx.mydomain.com
      EXPOSE: nginx:80
      GATEWAY_CLIENT_WG_PRIVKEY: 4M7Ap0euzTxq7gTA/WIYIt3nU+i2FvHUc9eYTFQ2CGI=
      GATEWAY_LINK_WG_PUBKEY: Wipd6Pv7ttmII4/Oj82I5tmGZwuw6ucsE3G+hwsMR08=
      GATEWAY_ENDPOINT: 123.456.789.101:49185
    cap_add:
      - NET_ADMIN
```

5. Start your docker compose project as you would normally (`docker compose up -d`)
6. 
7. This will established the `link` to the `gateway` and automatically provision a TLS-certificate

**You may repeat steps 3-5 for as many services as you would like to expose using the same gateway**

## Extra

### Architecture

<!-- TODO: Add a network diagram -->

```shell
├── ...
└── src
    ├── client-link  # WireGuard instance for the client. Also handles SSL termination with Caddy
    │   └── ...
    ├── create-link  # CLI script for establishing a link.
    │   └── ...
    ├── gateway      # NGINX reverse proxy to distribute requests to each gateway-link instance.
    │   └── ...
    └── gateway-link # WireGuard instance for the gateway.
        └── ...
```

### Terminology
- `Link` - A dedicated WireGuard tunnel between a local container (client) and the remote container running on the Gateway through which Reverse Proxy traffic is routed. A link is comprised of 2 pieces, the local or client link and the gateway or remote link.

### Split DNS without SSL Termination

In the event you already have a reverse proxy which performs SSL termination for your apps/services you can enable `FORWARD_ONLY` mode. Suppose you are using Traefik for SSL termination:

1. On your local LAN you will resolve \*.sub.mydomain.com to your local Traefik IP
2. On your external DNS for your domain you will resolve \*.sub.mydomain.com to the IP of your VPS
3. In your compose file add an additional two variables: `EXPOSE_HTTPS` and `FORWARD_ONLY`

```yaml
version: '3.9'
services:
  app:
    image: traefik:latest
  link:
    image: fractalnetworks/gateway-client:latest
    environment:
      LINK_DOMAIN: sub.mydomain.com
      EXPOSE: app:80
      EXPOSE_HTTPS: app:443
      FORWARD_ONLY: "True"
      GATEWAY_CLIENT_WG_PRIVKEY: 4M7Ap0euzTxq7gTA/WIYIt3nU+i2FvHUc9eYTFQ2CGI=
      GATEWAY_LINK_WG_PUBKEY: Wipd6Pv7ttmII4/Oj82I5tmGZwuw6ucsE3G+hwsMR08=
      GATEWAY_ENDPOINT: 5.161.127.102:49185
    cap_add:
      - NET_ADMIN
```
You will see logs from the link container indicating it is in forward only mode:
```
traefikv2_link.1.qvijxtwiu0wb@docker01    | + socat TCP4-LISTEN:8443,fork,reuseaddr TCP4:app:443,reuseaddr
traefikv2_link.1.qvijxtwiu0wb@docker01    | + socat TCP4-LISTEN:8080,fork,reuseaddr TCP4:app:80,reuseaddr
```

### TLS Backend

If the backend container already has a TLS certification, the connection between Caddy and the backend container can be switched to TLS/HTTPS with the `CADDY_TLS_PROXY` parameter.
In case the certificate is self-signed, the addition `CADDY_TLS_INSECURE` can be used to deactivate the certificate check.

This will continue to create a certificate for the backend via Let's Encrypt.

```yaml
version: '3.9'
services:
  app:
    image: traefik:latest
  link:
    image: fractalnetworks/gateway-client:latest
    environment:
      LINK_DOMAIN: sub.mydomain.com
      EXPOSE:  https://app:80
      CADDY_TLS_PROXY: true
      # Optional
      # CADDY_TLS_INSECURE: true
      GATEWAY_CLIENT_WG_PRIVKEY: 4M7Ap0euzTxq7gTA/WIYIt3nU+i2FvHUc9eYTFQ2CGI=
      GATEWAY_LINK_WG_PUBKEY: Wipd6Pv7ttmII4/Oj82I5tmGZwuw6ucsE3G+hwsMR08=
      GATEWAY_ENDPOINT: 5.161.127.102:49185
    cap_add:
      - NET_ADMIN
```

### Show all links running on a Gateway
```
$ docker ps
```

### Limitations

- Currently only IPv4 is supported
- Raw UDP proxying is supported but is currently untested & undocumented, see bottom of `gateway/link-entrypoint.sh`.

### FAQ

- How is this better than setting up nginx and WireGuard myself on a VPS?

The goal of this project is to self-hosting more accessible and reproducible. This selfhosted-gateway leverages a "ZeroTrust" network architecture (see diagram above). Each "Link" provides a dedicated WireGuard tunnel that is isolated from other containers and the underlying. This isolation is provided by Docker Compose's creation of a private Docker network for each compose file (project).

- Can I still access the service from my local network?

You will need to expose ports in your Docker host as you would traditionally, but this is no longer necessary:
```
ports:
 - 80:80
 - 443:443
```

### Support

Community support is available via our Matrix Channel https://matrix.to/#/#fractal:ether.ai
