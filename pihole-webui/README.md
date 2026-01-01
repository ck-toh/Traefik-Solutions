# Pi-Hole v6 with Traefik
# 1. Create External Docker Network for Traefik
# 2. Touch /home/cloud-user/traefik/log/traefik.log
# 3. Start Docker-compose using compose.yml
---
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    command:
      - "--log.level=ERROR"
      - "--log.filePath=/var/log/traefik.log"
      - "--serversTransport.insecureSkipVerify=true"
      - "--api.insecure=false"
      - "--api.dashboard=true"
      - "--ping=true"
      - "--ping.entryPoint=ping"
      - "--ping.manualrouting=true"
      - "--ping.terminatingStatusCode=204"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.traefik.address=:8080"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
    networks:
      - traefik
    ports:
      - 443:443
      - 80:80
    restart: unless-stopped
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/home/cloud-user/traefik/log/traefik.log:/var/log/traefik.log"
    extra_hosts:
      - host.docker.internal:172.17.0.1
    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=(HostRegexp(`^.+\.localdomain$`) && (PathPrefix(`/dashboard`)||PathRegexp(`^/api/(overview|version|entrypoints|http|tcp|udp)`)))
      - traefik.http.routers.api.service=api@internal
      - traefik.http.routers.api.entrypoints=websecure
      - traefik.http.routers.api.tls=true
      - traefik.http.routers.ping.rule=(HostRegexp(`^.+\.localdomain$`) && PathPrefix(`/ping`))
      - traefik.http.routers.ping.service=ping@internal
      - traefik.http.routers.ping.entrypoints=websecure
      - traefik.http.routers.ping.tls=true

  cloudflared:
    #container_name: cloudflared-dns
    image: visibilityspots/cloudflared
    hostname: cloudflare-dns
    restart: unless-stopped
    networks:
      pihole:
        ipv4_address: 172.88.0.3

  pihole:
    image: pihole/pihole:latest
    #container_name: pi-hole
    hostname: radon-pihole
    depends_on:
      - cloudflared
    networks:
      pihole:
      traefik:
    ports:
      - 53:53/tcp
      - 53:53/udp
    labels:
      - traefik.enable=true
      - traefik.http.routers.pihole.rule=(HostRegexp(`^.+\.localdomain$`) && (PathPrefix(`/pihole`) || PathPrefix(`/api`) || PathPrefix(`/admin`)))
      - traefik.http.routers.pihole.entrypoints=websecure
      - traefik.http.routers.pihole.tls=true
      - traefik.http.routers.pihole.middlewares=pihole
      - traefik.http.middlewares.pihole.redirectRegex.regex=^/admin/(.*)
      - traefik.http.middlewares.pihole.redirectRegex.replacement=/
      - traefik.http.services.pihole.loadBalancer.server.port=8089
      - traefik.docker.network=traefik
    volumes:
      - /home/cloud-user/pihole/etc-pihole:/etc/pihole
      - /home/cloud-user/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    dns:
      - 127.0.0.1
      - 192.168.11.1
    restart: unless-stopped
    cap_add:
       - CAP_SYS_NICE
       - CAP_SYS_TIME
    environment:
      TZ: Europe/London
      DNSMASQ_USER: root
      PIHOLE_UID: 0
      FTLCONF_dns_upstreams: 172.88.0.3#5054
      FTLCONF_dns_revServers: 'true,192.168.11.0/24,192.168.11.1,localdomain'
      FTLCONF_webserver_domain: 'photon6401.localdomain'
      FTLCONF_webserver_port: 8089
      FTLCONF_webserver_api_password: ********
      FTLCONF_dns_listeningMode: 'ALL'
      FTLCONF_dns_domain_name: 'localdomain'
      FTLCONF_ntp_sync_server: 'time.google.com'

networks:
  traefik:
    external: true
  pihole:
    driver: bridge
    ipam:
      config:
        - subnet: 172.88.0.0/29
