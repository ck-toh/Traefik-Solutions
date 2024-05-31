# Traefik-Solutions
## Host rules change in Traefik v3.

In Traefik v2, Host filer can be multiple hostname seperated by comma, however this no longers work in v3. Instead, we need to define each host separately and use `OR` for matching rule
```
      # Traefik v2 syntax
      - traefik.http.routers.myapp.rule=Host(`a.localdomain`,`b.localdomain`,`c.localdomain`) && PathPrefix(`/myapp`)
      # Traefik v3 syntax
      - traefik.http.routers.myapp.rule=(Host(`a.localdomain`) || Host(`b.localdomain`) || Host(`c.localdomain`) ) && PathPrefix(`/myapp`)
```

## labels
All connections are based on HTTPS toward Traefik

```
# /portainer - middleware to remove /portainer before sending to application
    labels:
      - traefik.enable=true
      - traefik.http.routers.portainer.rule=(HostRegexp(`^.+\.localdomain$`)||Host(`armdock500b.localdomain`)) && PathPrefix(`/portainer`)
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
      - traefik.http.routers.pihole.rule=(Host(`armdock500a.localdomain`)||Host(`armdock500b.localdomain`)||Host(`dns.localdomain`)) && (PathPrefix(`/pihole`) || PathPrefix(`/admin`))
      - traefik.http.routers.pihole.entrypoints=websecure
      - traefik.http.routers.pihole.tls=true
      - traefik.http.routers.pihole.middlewares=pihole
      - traefik.http.middlewares.pihole.replacepathregex.regex=^/pihole/(.*)
      - traefik.http.middlewares.pihole.replacepathregex.replacement=/admin/$$1
      - traefik.http.services.pihole.loadBalancer.server.port=80
      - traefik.docker.network=traefik


/unifi - Ubiquiti UniFi controller
    labels:
      - traefik.enable=true
      - traefik.http.routers.unifi.rule=Host(`unifi.localdomain`) #&& (PathPrefix(`/unifi/`) || PathPrefix(`/manage/`))
      - traefik.http.routers.unifi.entrypoints=websecure
      - traefik.http.routers.unifi.tls=true
      - traefik.http.routers.unifi.service=unifi
      - traefik.http.services.unifi.loadbalancer.server.scheme=https
      - traefik.http.services.unifi.loadBalancer.server.port=8443

/wolweb - WOL Web
    labels:
      - traefik.enable=true
      - traefik.http.routers.wolweb.rule=Host(`wakeonlan.localdomain`) && PathPrefix(`/wolweb`)
      - traefik.http.routers.wolweb.entrypoints=websecure
      - traefik.http.routers.wolweb.tls=true
      - traefik.http.routers.wolweb.service=wolweb
      - traefik.http.services.wolweb.loadbalancer.server.scheme=http
      - traefik.http.services.wolweb.loadBalancer.server.port=8089
```
