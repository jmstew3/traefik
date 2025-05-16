# Traefik with Cloudflare Tunnel Setup

This project provides a secure and robust reverse proxy setup using Traefik and Cloudflare Tunnel. It enables secure access to your services through Cloudflare's network while maintaining full control over your infrastructure.

## Features

- Traefik v2 as the reverse proxy
- Cloudflare Tunnel integration for secure external access
- Docker Compose based deployment
- Dynamic configuration support
- Traefik Dashboard access
- IP allowlist middleware for Cloudflare IPs
- TLS support
- Example whoami service

## Prerequisites

- Docker and Docker Compose
- Cloudflare account with a registered domain
- Cloudflare Tunnel token

## Setup

1. Clone this repository:
```bash
git clone <repository-url>
cd Traefik-3c
```

2. Create necessary directories:
```bash
mkdir -p certs logs
```

3. Copy the example environment file and configure it:
```bash
cp .env.example .env
```

4. Configure the following variables in `.env`:
- `CF_TUNNEL_TOKEN`: Your Cloudflare Tunnel token
- `BASE_DOMAIN_1`: Your base domain (e.g., example.com)
- `RPROXY_TUNNEL`: Your tunnel network name
- `RPROXY_GENERAL`: Your general network name
- `TRAEFIK_DASHBOARD_HOST_IP`: (Optional) Host IP for Traefik dashboard access
- `TRAEFIK_DASHBOARD_HOST_MAPPED_PORT`: Host port for Traefik dashboard

5. Create Docker networks:
```bash
# The network names must match your .env configuration:
# If your .env has:
# RPROXY_TUNNEL=TRAEFIK
# RPROXY_GENERAL=TRAEFIK-GENERAL
# Then create networks as:
docker network create TRAEFIK
docker network create TRAEFIK-GENERAL
```

⚠️ **Important Network Configuration Notes**:
- The Docker networks defined in `docker-compose.yml` use environment variables:
  ```yaml
  networks:
    --> traefik: # explicitly use the same value you defined from RPROXY_TUNNEL in .env
      external: true
      name: traefik    
    
    --> traefik_general: # explicitly use the same value from RPROXY_GENERAL in .env
      external: true
      name: traefik_general  
  ```
- Make sure the names for the networks you create match EXACTLY with your .env variables
- Network names are case-sensitive

## Configuration Files

### docker-compose.yml
- Defines three services:
  - `cloudflared`: Runs the Cloudflare Tunnel
  - `traefik`: The reverse proxy service
  - `whoami`: An example service

### dynamic/
- `middlewares-ipallowlist.yml`: Contains Cloudflare IP ranges for filtering
- `tls.yml`: TLS configuration for secure connections

## Usage

1. Start the services:
```bash
docker-compose up -d
```

2. Access the Traefik dashboard:
- By default: http://127.0.0.1:8080
- If configured with custom host IP: http://<your-host-ip>:8080

3. Access the whoami service:
- Through your Cloudflare domain: https://who.<your-domain>

## Security Features

- Cloudflare Tunnel for secure external access
- IP allowlist middleware to only allow Cloudflare IPs
- TLS encryption support
- Secure dashboard access configuration

## Directory Structure

```
.
├── docker-compose.yml
├── .env
├── certs/
├── logs/
└── dynamic/
    ├── middlewares-ipallowlist.yml
    └── tls.yml
```

## Monitoring

- Traefik logs are stored in `./logs/traefik.log`
- Access logs are stored in `./logs/access.log`
- Dashboard provides real-time monitoring of routes and services

## Customization

1. Add new services by defining them in `docker-compose.yml`
2. Configure routing rules using Docker labels
3. Add custom middlewares in the `dynamic/` directory
4. Modify TLS configuration in `dynamic/tls.yml`

## Troubleshooting

1. Check service status:
```bash
docker-compose ps
```

2. View logs:
```bash
docker-compose logs -f
```

3. Verify network connectivity:
```bash
docker network ls
docker network inspect traefik-tunnel
```

## Contributing

Feel free to submit issues and pull requests.

## License

MIT License