---
title: "Local Development Automation"
date: 2024-01-29
author: "Farzan"
tags: ["gitea", "traefik", "local"]
topics: ["devops"]
categories: ["automation"]
# layout: "simple"
showTableOfContents: false
summary: "this is an article about setting up an automated local dev environment"
draft: false
---
{{< toc >}}


## Introduction

there is a point in each of our tech journies where we start getting comfy in the cloud. we get used to the simplicity and the intuitiveness of spinning up nodes with just a few lines of YAML, delegated to a data center somewhere else - probably on the same continent as us. 

but when we working in our local environment things are comparatively more difficult to spin up, run, or maintain. we can start a project, go through all the docs to configure our environment, work on it for a bit and then get pulled away from it for while. coming back to that project would then be much harder since it's very likely we've forgotten what we did right the last time we were focused on it. 

compare that to cloud where our projects, for the most part, are self-contained with their CI/CD pipelines. we could always make new commits to the repo and as long as the pipeline runs we know we are good. 

so I thought to myself what if I could bring the CI/CD methodology to my local? this would simplify the process of starting new projects locally, and would help me maintain them somewhere else that's not my desktop.


my goal is to be able to:
1. start a new project, 
2. define a pipeline for it that would compile the project and export a package to a self hosted respository, 
3. and I could simply pull the package and run it on my current workstation without needing to reconfigure any settings or environment values.

 roughly speaking, this is how it would look:

{{< tabs "what-if" >}}
{{< tab "1. new project" >}}

```bash
> mkcd ~/dev/cool_project && go mod init example.com/cool_project

> cat << EOF > ~/dev/cool_project/main.go
    package main

    import "fmt"

    func main() {
        fmt.Println("Hello, World!")
    }
    EOF 
    
```
{{< /tab >}}




{{< tab "2. add docker and compose" >}}
```bash
> cat << EOF > ~/dev/cool_project/Dockerfile
  FROM golang:latest

  WORKDIR /app

  COPY . .

  RUN go build -o main .

  CMD ["./main"]
  EOF

> cat << EOF > ~/dev/cool_project/docker-compose.yml
  version: '3.8'
  services:
    cool_project:
      image: gitea.local/cool_project:latest
      container_name: cool_project_container
      ports:
        - "8080:8080"
      restart: always
  EOF
```
{{< /tab >}}



{{< tab "3. add a pipeline" >}}
```bash
> cat << EOF > .github/workflows/build-and-push.yml
  name: Build and Push Docker Imag  
  on:
    push:
      branches:
        - mai 
  jobs:
    build-and-push:
      runs-on: ubuntu-lates 
      steps:
        - name: Checkout code
          uses: actions/checkout@v  
        - name: Login to Docker registry
          run: echo "${{ secrets.DOCKER_PASSWORD }}" |  \
          docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin gitea.loca 
        - name: Build and tag Docker image
          run: docker build -t cool_project   
        - name: Get build number
          id: get_build_number
          run: echo ::set-output name=build_number::\${GITHUB_RUN_NUMBER  
        - name: Tag Docker image with build number
          run: docker tag cool_project gitea.local/cool_project:\${{ steps.get_build_number.outputs.build_number }  
        - name: Tag Docker image as latest
          run: docker tag cool_project gitea.local/cool_project:lates 
        - name: Push Docker images to registry
          run: |
            docker push gitea.local/cool_project:\${{ steps.get_build_number.outputs.build_number }}
            docker push gitea.local/cool_project:latest
    EOF
  ```
{{< /tab >}}
{{< /tabs >}}

and voila! from here on a simple commit push and `docker compose up -d` is all it takes to see my changes in actions. and I can come back to that draft of an idea anytime in the future and everything would still work as I left it last time because it would all be self-contained.


## Setting Up a Local Development Environment

<!-- Describe the steps you took to set up your local development environment.
Discuss the installation and configuration of Gitea, Traefik, and Pi-hole. -->

at the heart of this setup sits a version control service, one that would ideally also offer the ability to run pipelines and host package repositories. on the cloud we have GitHub, Azure DevOps and the likes. we could theoritically run self-hosted agents for them on our local network, but then we would need to expose our local network to public, with it comes numerous security and networking considerations such a firewall and ddns. this is a huge undertaking, the goal here is to make life easier, so cloud repositories are a pass.

to achieve whta I'm after I need: 
1. [git and package repository](#git-and-package-repository), 
2. [reverse proxy](#reverse-proxy), 
3. [a dns server](#dns-server), 
4. [and a custom root certificate to go with my setup](#securing-access-with-ssltls).

before we begin, you could find all the source code [here](https://sourcecode)






### Git and Package Repository

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




### Reverse Proxy


I have a few servers in my local network, each of them have a docker engine running a number of applications that I self-host. if I host gitea on any of these hosts I would have to serve it over a specific port that's not _:80,:443_, then I would have to remember what port I used to which application. that doesn't sounds like making life easier. 

with a reverse proxy I can configure _hostname routing_ instead of _port routing_. ergo, instead of _http://foo.lan:80_ leading me to gitea, I could configure _http://gitea.foo.lan_ to do the same. there are a good number of options for reverse proxy with the gold standard being [nginx](https://github.com/nginx/nginx). but in the spirit of making life easier, I decided to go with [traefik](https://github.com/traefik/traefik) for this. 

while with nginx we need to provide verbose configurations for our routing, using traefik we can configure a large number of this responsibility to traefik itself via labels.

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
1. we are telling traefik to route all incoming traffic to _web endpoint_ to _websecure endpoint_. if you have applications on this particular host that do not need https you could disable this by removing `- --entrypoints.web.http.redirections.entryPoint.to=websecure` 
2. you may have noticed in [last section](#git-and-package-repository) that we exposed gitea on an external network named _traefik\_public_. that network is created here, and it's how traefik know about other services on this particular docker socket.



### DNS Server

continuing with the goal of making life easier, once I have my local git repository I would not want to refer to it with an IP. for a few reasons, namely: 

1. most local devises lease their IP from the _DHCP server_ and this IP could change. we could solve for it using _static ips_, however, 
2.  `192.168.0.29/user/repo.git` does not feel right, or clean, or methodical, or easy. 

so we need a dns server on our local network. for this I went with [pi-hole](https://github.com/pi-hole/pi-hole). although there are other options such as [blocky](https://github.com/0xERR0R/blocky). or one could also just edit _hosts_ file and assign a qualifier to an ip, although I wouldn't recommend.

**pick a server in your local that is online as long as there is electriciy to your home**. setup your dns server here. there are lots of great documentations on pihole, so I won't go into details and will just focus on running it in a docker.


{{< notice "tip" "pihole networking" >}}
there two ways to setup networking for pihole, or any other dns service you go with.

{{< accordion title="bridge mode" class="accordion-minimal">}}
1. in bridge mode, pihole will attach itself to port _:53_ of the host. so we need to ensure there are no other services on the host that are using this port. for instance: I setup pihole on my synology that had the synology dns service running which I had to shut down.

2. in this mode we will use host's ip address as our dns server. this means that other containers running in this docker engine on this host will not be able to resolve dns names over its host ip address. [more here](https://docs.docker.com/network/drivers/bridge/).

{{< /accordion >}}

{{< accordion title="macvlan mode" class="accordion-minimal">}}
1. [macvlan](https://docs.docker.com/network/drivers/macvlan/) requests a dedicate ip address for our network from the router. hence allowing us to use _:53_ of this ip instead of host's. this is a good option if we do need _:53_ to remain free for host.
2. the problem with mcvlan is that the host, where the docker engine is running, would not be able to resolve to it. this means that our host cannot use our internal dns server. [there are ways to resolve this](https://stackoverflow.com/a/64360858), but it adds unneccessary maintenance to our setup.  
{{< /accordion >}}

 


{{< /notice >}}



there are good [documentation](https://github.com/pi-hole/docker-pi-hole) on installing pihole with docker. we could use the following for a simple installation:



{{< tabs "pihole-setup" >}}

{{< tab "docker-compose.yml" >}}


```yml
version: "3"

# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    # For DHCP it is recommended to remove these ports and instead add: network_mode: "host"
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp" # Only required if you are using Pi-hole as your DHCP server
      - "1080:80" # I'm reserving port 80 for traefik
    environment:
      TZ: 'America/Toronto'
      # WEBPASSWORD: 'set a secure password here or it will be random'
    # Volumes store your data between container upgrades
    volumes:
      - '/my/network/share/pihole/pihole:/etc/pihole'
      - '/my/network/share/pihole/dnsmasq.d:/etc/dnsmasq.d'
      - '/my/network/share/pihole/resolv.conf:/etc/resolv.conf'
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    cap_add:
      - NET_ADMIN # Required if you are using Pi-hole as your DHCP server, else not needed
    restart: unless-stopped
```
{{< /tab >}}

{{< tab "resolv.conf"  >}}
you may have noticed that I'm mounting _/etc/resolv.conf_ file. this file tells a container os what hostnames to use to resolve dns. since this container would sit behind my router, where I would set its own ip as my dns server it would attempt to resolve hosts using its own ip and unsuprisingly fail. so to avoid this, create a _resolv.conf_ file and set its content to following:

<br>

```shell
> cat << EOF > /my/network/share/pihole/resolv.conf
  nameserver 127.0.0.1
  nameserver 8.8.8.8
  EOF
```
{{< /tab >}}

{{< /tabs >}}

### Securing Access with SSL/TLS

one final config we need would be adding SSL to our [gitea](#gitea) endpoint. this is because to use the docker repository hosted on gitea we need to run `docker login` and it requires a secure connection.

there are several ways to achieve this, ranging from the [basic openssl method](), to using custom libraries such as [mkcert](https://github.com/FiloSottile/mkcert) and [minica](https://github.com/jsha/minica). I decided to use the latter, but any of the above work. 

to create a root certificate, and subsequently a certificate for our local dns endpoints we could do:
{{< notice "tip" >}}
  whichever method you choose to use, you'd still need to install the root certificate on every machine that's involved in your workflow. this includes your desktop/laptop as well as docker hosts and containers. 

 {{< accordion  title="on windows" class="accordion-minimal" >}}
  ```bash
  # - Run below.
  # - Double click on .pfx to install. choose local machine, and choose Trusted Root.. Store.
  # - Go to Cert Manager, find minica. Double Click -> Details Tab -> \
  #   Copy to File. Anywhere you want.
  # - Go to generated .cer file, double click, install for local machine \
  #   in Trusted Root... Store.
  # - Restart browsers.
  openssl pkcs12 -export -out rootCA.pfx -inkey rootCA.key -in rootCA.pem
  ```
  <br/>
  {{< /accordion >}}
  {{< accordion  title="on linux" class="accordion-minimal" >}}
  ```bash
  openssl x509 -in rootCA.key -inform PEM -out foo.crt
  sudo cp foo.crt /usr/local/share/ca-certificates
  sudo update-ca-certificates
  ```
  <br/>
  {{< /accordion >}}
  {{< accordion  title="on containers" class="accordion-minimal" >}}
  ```yaml
  ## docker-compose.yml
  ...
  services:
    ...
    volumes:
      - /etc/ssl/certs:/etc/ssl/certs:ro
  ...
  ```
  <br/>
  {{< /accordion >}}
{{< /notice >}}

{{< tabs "certificate-create" >}}
{{< tab "minica" >}}
```bash
## install
go install github.com/jsha/minica@latest

# Generate a root key and cert in minica-key.pem, and minica.pem, then
# generate and sign an end-entity key and cert, storing them in ./foo.lan/
$ minica --domains foo.lan

# Wildcard
$ minica --domains '*.foo.lan'
```
{{< /tab >}}
{{< tab "mkcert" >}}
```bash
## from: https://github.com/FiloSottile/mkcert

$ mkcert -install
# Created a new local CA 💥
# The local CA is now installed in the system trust store! ⚡️
# The local CA is now installed in the Firefox trust \ 
# store (requires browser restart)! 🦊

$ mkcert "foo.lan" "*.foo.lan"

# Created a new certificate valid for the following names 📜
#  - "foo.lan"
#  - "*.foo.lan"
# 
# 
# The certificate is at "./foo.lan+2.pem" \
# and the key at "./foo.lan+2-key.pem" ✅
```
{{< /tab >}}
{{< tab "openssl" >}}
```bash
## from: https://github.com/karvounis/traefik-tutorial- \
## docker-compose-files/blob/master/standalone/advanced/README.md

mkdir -p certs/{ca,traefik}
openssl genrsa -out certs/ca/rootCA.key 4096
openssl req -x509 -new -nodes \
  -key certs/ca/rootCA.key -sha256 -days 3650 \
  -out certs/ca/rootCA.pem \
  -subj "/C=GR/L=Toronto/O=Traefik Local/CN=Traefik Root CA/OU=CA Dep"

openssl genrsa -out certs/traefik/traefik.key 4096
openssl req -new \
  -key certs/traefik/traefik.key -out certs/traefik/traefik.csr \
  -subj "/C=GR/L=Toronto/O=Traefik Local/CN=*.foo.com/OU=Dev.to"
openssl x509 -req -in certs/traefik/traefik.csr \
  -CA certs/ca/rootCA.pem -CAkey certs/ca/rootCA.key \
  -CAcreateserial -out certs/traefik/traefik.crt -days 3650 -sha256
```


{{< /tab >}}
{{< /tabs >}}




## Continuous Integration and Deployment with Gitea Runner


so we have our local environment setup and ready. now we need workflows, and to run the flows we need [act runners](https://docs.gitea.com/next/usage/actions/act-runner/). gitea act runners use [github actions](https://docs.github.com/en/actions) syntax and tags. 

let's set up our first act runner: 

{{< notice "tip" "configuration-for-runner" >}}
we can generate the configuration needed for runner as described [here](https://docs.gitea.com/next/usage/actions/act-runner/#configuration)

or simply: 

    docker run --entrypoint="" --rm -it gitea/act_runner:latest act_runner generate-config > config.yaml

{{< /notice >}}

{{< tabs "gitea-runner" >}}
{{< tab "docker-compose.yml" >}}
```yaml
version: "3.8"
services:
  runner:
    image: gitea/act_runner:nightly
    environment:
      CONFIG_FILE: /config.yaml
      GITEA_INSTANCE_URL: "${INSTANCE_URL}"
      GITEA_RUNNER_REGISTRATION_TOKEN: "${REGISTRATION_TOKEN}"
      GITEA_RUNNER_NAME: "${RUNNER_NAME}"
      GITEA_RUNNER_LABELS: "${RUNNER_LABELS}"
    volumes:
      - ./config.yaml:/config.yaml
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/ssl/certs:/etc/ssl/certs:ro
```
{{< /tab >}}

{{< tab ".env" >}}
```shell
INSTANCE_URL=https://gitea.foo.lan
## Obtaining token: https://docs.gitea.com/next/usage/actions/act-runner/#obtain-a-registration-token
REGISTRATION_TOKEN=[TOKEN]
RUNNER_NAME=[name]
## https://docs.gitea.com/next/usage/actions/act-runner/#labels
RUNNER_LABELS=ubuntu-latest:docker://node:16-bullseye, \
              cth-ubuntu-latest:docker://catthehacker/ubuntu:act-latest
```
{{< /tab >}}

{{< /tabs >}}

and with this we should be ready to start our first project with automated workflow.

## Version Control and Docker Repository Management with Gitea
1. let's start a hello_world project
```
mkdir hello_world && cd hello_world
go mod init gitea.foo.lan/hello_world
```
2. create `main.go` and add following to it:
```go
package main

import (
	"log"

	"github.com/gofiber/fiber/v2"
)

func main() {
	app := fiber.New()

	app.Get("/", func(c *fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	log.Fatal(app.Listen(":3000"))
}
```
3. add `Dockerfile`
```dockerfile
FROM golang:alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o main .

FROM alpine:latest  

WORKDIR /root/

COPY --from=builder /app/main .

EXPOSE 3000

CMD ["./main"]

```

4. and finally, add our action to `.gitea/workflows/ci.yaml`
```yaml
name: ci

on:
  push:
    branches:
      - 'main'
jobs:
  build:
    runs-on: cth-ubuntu-latest

    steps:
      -
        name: Extract Gitea Server URL
        id: extract-url
        run: >-
         echo "::set-output name=url::$(
          echo ${{ gitea.server_url }} | 
          sed 's~^https://~~ '
          )"
      -
        name: Checkout
        uses: actions/checkout@v4
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        # https://docs.docker.com/build/buildkit/toml-configuration/
        with:
          buildkitd-flags: --debug
          config-inline: |
            [registry."${{ steps.extract-url.outputs.url }}"]
              insecure=true
              http=true
      - 
        name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          registry: ${{ steps.extract-url.outputs.url }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      -
        name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          ## https://docs.gitea.com/next/usage/packages/container#push-an-image
          tags: "${{ steps.extract-url.outputs.url }}/${{gitea.repository}}:${{gitea.sha}}, \
                 ${{ steps.extract-url.outputs.url }}/${{gitea.repository}}:latest"
```

and voila! commit the changes, and push them to your gitea instance. you could see the pipeline kickoff shortly after the commit. once the pipeline completes, you should see the docker file in `https://gitea.foo.lan/user/repository/packages`.

all that's left is to run our latest package:

    docker run -d -p 3000:3000 gitea.foo.lan/user/hello-world:latest

 
    curl localhost:3000
    # output: Hello, World!

## Conclusion

it was quite a bit of work to get to this point, but I look at it as an investment for the future. considering gitea uses github actions syntax and approach, expanding on our current setup would be relatively simple in that we could customize our local cicd infrastructure by creating custom actions and chaining them together.

hope you enjoyed this!

## References and Further Reading

- [Gitea Documentation](https://docs.gitea.com/category/installation)
- [Gitea Act Runner](https://docs.gitea.com/next/usage/actions/act-runner/)
- [Gitea Act Container with Docker](https://github.com/catthehacker/docker_images)
- [Docker BuildX](https://docs.docker.com/build/ci/github-actions/)

