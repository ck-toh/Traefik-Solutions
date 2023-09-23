# Setting up HA of Traefik with Keepalived

1. Install Traefik in Docker Container with PING enabled
```
version: "3.3"

services:
  traefik:
    image: "traefik:latest"
    container_name: "traefik"
    command:
      #- "--log.level=DEBUG"
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
2. Install Keepalived on all Traefik Nodes
```
vrrp_script chk_myscript {
  script       "/home/docker/traefik/scripts/ping-traefik.sh"
  interval 1   # check every 1 seconds
  fall 2       # require 2 failures for KO
  rise 2       # require 2 successes for OK
}

vrrp_instance VI_1 {
        state MASTER
        interface eth0
        virtual_router_id 53
        unicast_src_ip 192.168.11.166
        unicast_peer {
           192.168.11.174
        }
        priority 200
        advert_int 1
        authentication {
              auth_type PASS
              auth_pass 12345
        }
        virtual_ipaddress {
              192.168.11.53/24
        }
        track_script {
              chk_myscript
        }
}
```
```
vrrp_script chk_myscript {
  script       "/home/docker/traefik/scripts/ping-traefik.sh"
  interval 1   # check every 1 seconds
  fall 2       # require 2 failures for KO
  rise 2       # require 2 successes for OK
}

vrrp_instance VI_1 {
        state BACKUP
        interface eth0
        virtual_router_id 53
        unicast_src_ip 192.168.11.174
        unicast_peer {
           192.168.11.166
        }
        priority 199
        advert_int 1
        authentication {
              auth_type PASS
              auth_pass 12345
        }
        virtual_ipaddress {
              192.168.11.53/24
        }
        track_script {
              chk_myscript
        }
}
```
3. Script used to Track if Traefik is running
```
#!/bin/bash
A=`curl -k https://xenon.localdomain:8082/ping`
if [ "$A" == "OK" ];then
    echo "ok";
    exit 0;
else
    echo "ko";
    exit 1;
fi
```
