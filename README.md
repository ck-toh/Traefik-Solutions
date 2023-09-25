# Traefik-Solutions

## labels
All connections are based on HTTPS toward Traefik

```
# /portainer - middleware to remove /portainer before sending to application
    labels:
      - traefik.enable=true
      - traefik.http.routers.portainer.rule=Host(`xenon.localdomain`,`armdock500a.localdomain`,`armdock500b.localdomain`) && PathPrefix(`/portainer`)
      - traefik.http.routers.portainer.entrypoints=websecure
      - traefik.http.routers.portainer.tls=true
      - traefik.http.routers.portainer.service=portainer
      - traefik.http.services.portainer.loadbalancer.server.scheme=https
      - traefik.http.services.portainer.loadBalancer.server.port=9443
      - traefik.http.routers.portainer.middlewares=portainer
      - traefik.http.middlewares.portainer.stripprefix.prefixes=/portainer

# /pihole - middleware to swap /pihole/ to /admin/ before sending to application; one problem with pihole after login, it switch to /admin but it can be manually swith back to /pihole
    labels:
      - traefik.enable=true
      - traefik.http.routers.pihole.rule=Host(`xenon.localdomain`,`armdock500a.localdomain`,`armdock500b.localdomain`) && (PathPrefix(`/pihole`) || PathPrefix(`/admin`))
      - traefik.http.routers.pihole.entrypoints=websecure
      - traefik.http.routers.pihole.tls=true
      - traefik.http.routers.pihole.middlewares=pihole
      - traefik.http.middlewares.pihole.replacepathregex.regex=^/pihole/(.*)
      - traefik.http.middlewares.pihole.replacepathregex.replacement=/admin/$$1
      - traefik.http.services.pihole.loadBalancer.server.port=80
      - traefik.docker.network=traefik
```
