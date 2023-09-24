# Traefik Proxy to Pi-hole Web UI

## Traefik Container
```
version: "3.3"
services:
  traefik:
    image: "traefik:latest"
    container_name: "traefik"
    command:
      - "--api.insecure=false"
      - "--api.dashboard=true"
      - "--ping=true"
      - "--ping.entryPoint=ping"
      - "--ping.manualrouting=true"
      - "--ping.terminatingStatusCode=204"
      - "--providers.docker=true"
      - "--providers.file.directory=/etc/traefik/dynamic_conf"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.traefik.address=:8080"
      - "--entrypoints.ping.address=:8082"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
    networks:
      - traefik
    ports:
      - "443:443"
      - "80:80"
      - "8080:8080"
      - "8082:8082"
    restart: unless-stopped
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/home/docker/traefik/dynamic_conf/dynamic_conf.yml:/etc/traefik/dynamic_conf/dynamic_conf.yml:ro"
      - "/home/docker/traefik/tls/cert.pem:/etc/ssl/certs/cert.pem:ro"
      - "/home/docker/traefik/tls/cert_priv.key:/etc/ssl/private/cert_priv.key:ro"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`xenon.localdomain`,`xenon-a.localdomain`,`xenon-b.localdomain`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.api.service=api@internal"
      - "traefik.http.routers.api.entrypoints=traefik"
      - "traefik.http.routers.api.tls=true"
      - "traefik.http.routers.ping.rule=Host(`xenon.localdomain`,`xenon-a.localdomain`,`xenon-b.localdomain`) && PathPrefix(`/ping`)"
      - "traefik.http.routers.ping.service=ping@internal"
      - "traefik.http.routers.ping.entrypoints=ping"
      - "traefik.http.routers.ping.tls=true"
networks:
  traefik:
    external: true

```

## Pi-hole Container
- Only DNS ports are exposed, WebUI is handled by Traefik

```
version: "3.3"
services:
  cloudflared:
    container_name: cloudflared
    image: visibilityspots/cloudflared
    hostname: cloudflare-dns
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.88.0.3
  pihole:
    image: pihole/pihole:latest
    #container_name: pihole
    hostname: xenon-b
    depends_on:
      - cloudflared
    networks:
      pihole:
      traefik:
    deploy:
      replicas: 1
    ports:
      - 53:53/tcp
      - 53:53/udp
    labels:
      - traefik.enable=true
      - traefik.http.routers.pihole.rule=Host(`xenon.localdomain`,`xenon-a.localdomain`,`xenon-b.localdomain`) && PathPrefix(`/admin`)
      - traefik.http.routers.pihole.entrypoints=websecure
      - traefik.http.routers.pihole.tls=true
      - traefik.http.middlewares.pihole.addprefix.prefix=/admin
      - traefik.http.services.pihole.loadBalancer.server.port=80
      - traefik.docker.network=traefik
    volumes:
      - /home/docker/pihole/etc-pihole:/etc/pihole
      - /home/docker/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    dns:
      - 127.0.0.1
      - 192.168.11.1
    restart: unless-stopped
    #cap_add:
    #  - NET_ADMIN
    environment:
      TZ: Europe/London
      DNS1: 172.88.0.3#5054
      DNS2: 172.88.0.3#5054
      WEBPASSWORD: 4DPkgqHO
      DNSMASQ_USER: root
      PIHOLE_UID: 0
networks:
  pihole:
    driver: bridge
    ipam:
      config:
        - subnet: 172.88.0.0/29
  traefik:
    external: true
```
