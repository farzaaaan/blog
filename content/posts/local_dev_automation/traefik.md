I have a few servers in my local network, each of them have a docker engine running a number of applications that I self-host. if I host gitea on any of these hosts I would have to serve it over a specific port that's not _:80,:443_, then I would have to remember what port I used for which application. that doesn't sounds like making life easier. 

with a reverse proxy I can configure _hostname routing_ instead of _port routing_. ergo, instead of _http://foo.lan:80_ leading me to gitea, I could configure _http://gitea.foo.lan_ to do the same. there are a good number of options for reverse proxy with the gold standard being [nginx](https://github.com/nginx/nginx). but in the spirit of making life easier, I decided to go with [traefik](https://github.com/traefik/traefik) for this. 

while with nginx we need to provide verbose configurations for our routing, using traefik we can delegate a large portion of this responsibility to traefik itself via labels.

{{< tabs "traefik-setup">}}
{{< tab "docker-compose.yml" >}}

```yaml
## https://github.com/karvounis/traefik-tutorial-docker-compose-files/ \
## blob/master/standalone/advanced/docker-compose.dashboard.yml

version: "3.7"

services:
  traefik:
    image: traefik:v${TRAEFIK_VERSION}
    command:
      # Entrypoints configuration
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      ## Forces redirection of incoming requests from `web` to `websecure` entrypoint. 
      ## https://doc.traefik.io/traefik/routing/entrypoints/#redirection
      ## - --entrypoints.web.http.redirections.entryPoint.to=websecure
      # Docker provider configuration
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=traefik_public
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
    restart: unless-stopped
    
networks:
  traefik_public:
    name: traefik_public
    driver: bridge
```
{{< /tab >}}
{{< tab ".env" >}}



```shell
COMPOSE_PROJECT_NAME=traefik
HOST_DNS=foo.lan # the dns name I've assigned to the host traefik will be running on.
DYNAMIC_CONFIG=./config.yml # this is where we declare our tls certs for traefik to use
CERTIFICATES_DIR=./certs/ # mounting the certificates (generated in the next section)
TRAEFIK_VERSION=2.10
```
{{< /tab >}}
{{< tab "config.yml" >}}
```yml
tls:
  certificates:
    - certFile: /ssl/foo.lan/cert.pem
      keyFile: /ssl/foo.lan/key.pem
```
{{< /tab >}}
{{< /tabs >}}

a few notes: 
1. we are telling traefik to expose both  _web endpoint_ and _websecure endpoint_. you could set up default https redirection if you choose so. although I'd recommend against it as we'd run into a lot of issues once [we are setting up runner](#continuous-integration-and-deployment-with-gitea-runner).
2. you may have noticed in [last section](#git-and-package-repository) that we exposed gitea on an external network named _traefik\_public_. that network is created here, and it's how traefik know about other services on this particular docker socket.


