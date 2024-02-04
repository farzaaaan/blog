there are a number open source implementations of Git, honourable mentions are [gitness](https://github.com/harness/gitness) and [gogs](https://github.com/gogs/gogs) each brining a lot to the table. although ultimately I went with [gitea](https://github.com/go-gitea/gitea). what you choose depends on your preference, and of course this setup, for the most part, will work for any of the three.

gitea is [well documented](https://docs.gitea.com/) and I highly recommend reading through the docs. there is a dedicated section for [installing on docker](https://docs.gitea.com/installation/install-with-docker) as well as a [handy config sheet](https://docs.gitea.com/administration/config-cheat-sheet). here's a simple setup:

{{< tabs "gitea-setup">}}
{{< tab "docker-compose.yml" >}}
```yml
version: "3"

networks:
  gitea:
    name: gitea
    driver: bridge
  traefik_public:
    external: true

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    depends_on:
      gitea-cache:
        condition: service_healthy
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - TZ=America/Toronto
      - DOMAIN=${GITEA_DOMAIN}
      - ROOT_URL=https://${GITEA_DOMAIN}/
      - GITEA__cache__ENABLED=true
      - GITEA__cache__ADAPTER=redis
      - GITEA__cache__HOST=redis://gitea-cache:6379/0?pool_size=100&idle_timeout=180s
      - GITEA__cache__ITEM_TTL=24h
      - HTTP_PORT=3000
    restart: always
    networks:
      - traefik_public
      - gitea
    volumes:
      - ${GITEA_DATA}:/data
    ports:
      - "3000:3000"
      - "222:22"
    labels:
      - traefik.enable=true
      - traefik.http.routers.gitea_route.entrypoints=websecure
      - traefik.http.routers.gitea_route.rule=Host(`${GITEA_DOMAIN}`)
      - traefik.http.routers.gitea_route.service=gitea_service
      - traefik.http.routers.gitea_route.tls=true
      - traefik.http.services.gitea_service.loadbalancer.server.port=3000
      - traefik.docker.network=traefik_public
      - traefik.http.routers.gitea_route_http.entrypoints=web
      - traefik.http.routers.gitea_route_http.rule=Host(`${GITEA_DOMAIN}`)
      - traefik.http.routers.gitea_route_http.service=gitea_service
  gitea-cache:
    container_name: gitea-cache
    image: redis:6-alpine
    restart: unless-stopped
    networks:
      - gitea
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 15s
      timeout: 3s
      retries: 10
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
```
{{< /tab >}}
{{< tab ".env" >}}


```shell
COMPOSE_PROJECT_NAME=gitea
GITEA_DOMAIN=gitea.foo.lan 
GITEA_DATA=/my/network/share/gitea-data

```
{{< /tab >}}
{{< /tabs >}}

a few notes: 
1. _gitea.foo.lan_ is the domain I am serving my instance of gitea at. this is a _CNAME_ pointing to _foo.lan_, which is an _A Record_. both are set in my [dns server](#dns-server).
2. _ROOT\_URL_ and _traefik_ labels indicate that we're using _https_ for our connection. _tls_ is terminated at _traefik_ as indicated by the labels. we will later create this tls certificate [here](#securing-access-with-ssltls) and provide those certs to traefik [here](#reverse-proxy) 
3. I've set up _redis cache_ to be used by gitea [as documented here](https://docs.gitea.com/administration/config-cheat-sheet#cache-cache).

