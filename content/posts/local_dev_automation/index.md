---
title: "Local Development Automation"
date: 2024-01-29
author: "Farzan"
tags: ["gitea", "traefik", "local"]
topics: ["devops"]
categories: ["automation"]
# layout: "simple"
showTableOfContents: true
summary: "this is an article about setting up an automated local dev environment"
---

{{< lead >}} Build locally as if you're on the cloud, easier said than done, amiright? {{< /lead >}}

this is the beginning
 
## Background

## Requirements

## Solution

### SSL

### Traefik

### Gitea

### Gitea - Actions

## Final

```yml
version: "3.7"

services:
  traefik:
    image: traefik:v${TRAEFIK_VERSION}
    command:
      # Entrypoints configuration
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      ## Forces redirection of incoming requests from `web` to `websecure` entrypoint. https://doc.traefik.io/traefik/routing/entrypoints/#redirection
      - --entrypoints.web.http.redirections.entryPoint.to=websecure
      # Docker provider configuration
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.endpoint=tcp://socket_proxy:2375
      - --providers.docker.network=${TRAEFIK_NETWORK}
      # File provider configuration
      - --providers.file.directory=/traefik/config/dynamic
      # Logging configuration
      - --log.level=debug
      - --log.format=json
      # Enable dashboard https://doc.traefik.io/traefik/operations/dashboard/#secure-mode
      - --api.dashboard=true
    labels:
      - traefik.enable=true
      - traefik.http.routers.dashboard.rule=Host(`traefik.${HOST_DNS}`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))
      - traefik.http.routers.dashboard.tls=true
      - traefik.http.routers.dashboard.entrypoints=websecure
      - traefik.http.routers.dashboard.service=api@internal
    ports:
      - 80:80
      - 443:443
    volumes:
      - ${CERTIFICATES_DIR}:/ssl/:ro
      - ${DYNAMIC_CONFIG}:/traefik/config/dynamic/conf.yml:ro
    networks:
      - traefik_public
      - socket_proxy
    restart: unless-stopped
    depends_on:
      - socket_proxy

  # https://github.com/traefik/whoami
  whoami:
    image: traefik/whoami:v1.7.1
    labels:
      - traefik.enable=true
      # Router configuration
      ## Listen to the `websecure` entrypoint
      - traefik.http.routers.whoami_route.entrypoints=websecure
      - traefik.http.routers.whoami_route.rule=Host(`whoami.${HOST_DNS}`)
      - traefik.http.routers.whoami_route.service=whoami_service
      ## Enable tls for the `whoami_route`. Even if you do not set this to true, the redirection from http to https will do it for you.
      - traefik.http.routers.whoami_route.tls=true
      - traefik.http.services.whoami_service.loadbalancer.server.port=80
    networks:
      - traefik_public

  # https://github.com/Tecnativa/docker-socket-proxy
  socket_proxy:
    image: tecnativa/docker-socket-proxy:latest
    restart: unless-stopped
    environment:
      NETWORKS: 1
      SERVICES: 1
      CONTAINERS: 1
      TASKS: 1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - socket_proxy

networks:
  traefik_public:
    name: traefik_public
    driver: bridge
  socket_proxy:
    name: socket_proxy
    driver: bridge
```