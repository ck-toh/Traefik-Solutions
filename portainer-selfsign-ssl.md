# Traefik with Portainer - Self signed Certificate

```
---
version: "3.3"
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    command:
      #- "--log.level=DEBUG"
      #- "--log.filePath=/var/log/traefik.log"
      - "--serversTransport.insecureSkipVerify=true"
      - "--api.insecure=false"
      - "--api.dashboard=true"
      - "--ping=true"
      - "--ping.entryPoint=ping"
      - "--ping.manualrouting=true"
      - "--ping.terminatingStatusCode=204"
      - "--providers.docker=true"
      #- "--providers.file.directory=/etc/traefik/dynamic_conf"
      - "--providers.docker.exposedbydefault=false"
      #- "--entrypoints.dnstcp.address=:53"
      #- "--entrypoints.dnsudp.address=:53/udp"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.traefik.address=:8080"
      - "--entrypoints.ping.address=:8082"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
    networks:
      - traefik
    ports:
      #- 53:53
      #- 53:53/udp
      - 443:443
      - 80:80
      - 8080:8080
      - 8082:8082
    restart: unless-stopped
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      #- "/home/docker/traefik/dynamic_conf/dynamic_conf.yml:/etc/traefik/dynamic_conf/dynamic_conf.yml:ro"
      #- "/home/docker/traefik/tls/cert.pem:/etc/ssl/certs/cert.pem:ro"
      #- "/home/docker/traefik/tls/cert_priv.key:/etc/ssl/private/cert_priv.key:ro"
    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=Host(`xenon.localdomain`,`armdock500a.localdomain`,`armdock500b.localdomain`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))
      - traefik.http.routers.api.service=api@internal
      - traefik.http.routers.api.entrypoints=traefik
      - traefik.http.routers.api.tls=true
      - traefik.http.routers.ping.rule=Host(`xenon.localdomain`,`armdock500a.localdomain`,`armdock500b.localdomain`) && PathPrefix(`/ping`)
      - traefik.http.routers.ping.service=ping@internal
      - traefik.http.routers.ping.entrypoints=ping
      - traefik.http.routers.ping.tls=true

  portainer:
    image: portainer/portainer-ce:latest
    depends_on:
      - traefik
    networks:
      - traefik
    #ports:
    #  - 8000:8000
    #  - 9443:9443
    restart: unless-stopped
    volumes:
      - "/home/docker/portainer/data:/data"
      - "/var/run/docker.sock:/var/run/docker.sock"
    labels:
      - traefik.enable=true
      - traefik.http.routers.portainer.rule=Host(`xenon.localdomain`,`armdock500a.localdomain`,`armdock500b.localdomain`)
      - traefik.http.routers.portainer.entrypoints=websecure
      - traefik.http.routers.portainer.tls=true
      - traefik.http.routers.portainer.service=portainer
      - traefik.http.services.portainer.loadbalancer.server.scheme=https
      - traefik.http.services.portainer.loadBalancer.server.port=9443
networks:
  traefik:
    external: true
```
