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
---
{{< toc >}}


## Introduction

there is a point in each of our tech journies where we start getting comfy in the cloud. we get used to the simplicity and the intuitiveness of spinning up nodes with just a few lines of YAML, delegated to a data center somewhere else - probably on the same continent as us. as professionals we navigate streamlined workflows that take millions of dollars in investments, and as hobbyists we have the simplicity of the likes of GitHub, Cloudflare, Netlify, etc.. at our fingertips.

so yeah, life in the cloud is comfy and easy, and we get used to it. so it's no surprise I started finding myself having a hard time starting new projects on my local machine. every new project requires a certain amount of configuration. configurations that are often niche enough that I would forget once I stopped working on that project for while. which would then make debugging or going back to that project much harder and undesirable. I am speaking about those tiny environment variables, build contexts and the likes. and yes there are makefiles, but those only go so far and often are not completely host agnostic.

so I thought to myself what if I try to bring that simplicity of cloud to my local network. what if instead of I having to keep configurations on my desktop/wsl and wrestling with mainting them... I had CI CD pipeliens running on my local network? 

what if I could just: 

 

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

and voila! from here on a simple commit push and `docker compose up -d` is all it takes to see my changes in actions. and I can come back to that draft of an idea anytime in the future and everything would still work as I left it last time because it would all be maintained in its own repo and pipelines.


## Setting Up a Local Development Environment

<!-- Describe the steps you took to set up your local development environment.
Discuss the installation and configuration of Gitea, Traefik, and Pi-hole. -->

at the heart of this setup sits a version control service, one that would ideally also offer the ability to run pipelines and host package repositories. on the cloud we have GitHub, Azure DevOps and the likes. we could theoritically run self-hosted agents for them on our local network, but then we would need a firewall and reverse proxy to handle traffic between cloud and local. we'd likely also need a dynamic dns. in itself this a huge undertaking, add to it the maintenance of our firewall and security, because we are exposing to public and we'd be creating more headache than solving. the goal here is to make life easier, so cloud repositories are not an option.

there are a number open source implementations of Git, honourable mentions are [gitness](https://github.com/harness/gitness) and [gogs](https://github.com/gogs/gogs) each brining a lot to the table. although ultimately I went with [gitea](https://github.com/go-gitea/gitea). what you choose ultimately depends on your preference, and of course this setup, for the most part, would work for any of the three.

continuing with the goal of making life easier, once I have my local git repository I would not want to refer to it with an IP. first: most local devices lease their IP from a DHCP server that's usually on the router, so there is no gaurantee that the IPs assigned to a given harware address will remain constant. we could solve this by assigning static IPs, either via the DHCP server or via the client, however, second: 192.168.0.29/user/repo.git does not feel right, or clean, or methodical, or easy. so we need a dns server on our local network. for this I went with [pi-hole](https://github.com/pi-hole/pi-hole). although there are other options such as [blocky](https://github.com/0xERR0R/blocky). or one could also just edit _hosts_ file and assign a qualifier to an ip, although I wouldn't recommend.

in addition to a dns server, we would also want to have a reverse proxy. the purpose of the reverse proxy is so that we could avoid having to use port numbers to reach our repositories. this wouldn't be an issue if our services were served over _:80,:443_, however I don't want to dedicate an entire server to just one application. I have a few servers in my local network, each of them have a docker engine running a number of applications that I self-host. so taking port _:80,:443_ would interfere with the rest of my setups as I would then have to resolve to the other apps running on the same host using port number. there are of course a good number of options for reverse proxy with the gold standard being [nginx](https://github.com/nginx/nginx). but in the spirit of making life easier, I decided to go with [traefik](https://github.com/traefik/traefik) for this.

this should help us achieve everything we want, with the exception of SSL/TLS that's required by some package managers such as docker's, but we'll cover that in the next section.

before we begin, you could find all the source code [here](https://sourcecode)

### pi-hole

pick a server in your local that is online as long as there is electriciy to your home. setup your dns server here. there are lots of great documentations on pihole, so I won't go into details and will just focus on running it in a docker.

there are two ways you could run your pihole in docker, 1: in bridge mode, 2: in mcvlan mode. the problem with mcvlan is that the host, where docker engine is running, would not be able to resolve to it. [there are ways to resolve this](https://stackoverflow.com/a/64360858), but it adds unneccessary maintenance to our setup. the problem with bridge mode, on the other hand, is that other containers running on the same docker socket would not be able to resolve to our pihole. this can be resolved by not running the applications that need access to our local dns server on the same host as our dns server. this was the better option for me.

to run pihole, save this `docker-compose.yml` file, and run `docker compose up -d`



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
you may have noticed that I'm mounting _/etc/resolv.conf_ file. this file tells a container os what hostnames to use to resolve dns. since this would sit behind my router, where I would set this containers ip as my dns server it would attempt to resolve hosts using its own ip and unsuprisingly fail. so to avoid this, create a _resolv.conf_ file and set its content to following:

<br>

```shell
> cat << EOF > /my/network/share/pihole/resolv.conf
  nameserver 127.0.0.1
  nameserver 8.8.8.8
  EOF
```
{{< /tab >}}

{{< /tabs >}}




### gitea

gitea is [well documented](https://docs.gitea.com/) and I highly recommend reading through it. there is a dedicated doc for [installing on docker](https://docs.gitea.com/installation/install-with-docker) as well as a [handy config sheet](https://docs.gitea.com/administration/config-cheat-sheet). to install it, save and run the following `docker-compose.yml`:

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
> cat << EOF > /my/network/share/gitea/.env
  COMPOSE_PROJECT_NAME=gitea
  
  GITEA_DOMAIN=gitea.beebox.lan 

  GITEA_DATA=/my/network/share/gitea-data
  EOF
```
{{< /tab >}}
{{< /tabs >}}
you might have noticed _traefik_ configurations in the compose, we'll be deploying traefik [next](#traefik) but for now those could be ignored. notice we are also deploying a _redis cache_ to be used by gitea [as documented here](https://docs.gitea.com/administration/config-cheat-sheet#cache-cache).

the domain I'm hosting my gitea at is _gitea.beebox.lan_, with _beebox.lan_ being an A record I set in pihole pointing to the ip of my server, and _gitea.beebox.lan_ is a cname poiting to that dns. 

_gitea.beebox.lan_ is the hostname that is sent to traefik, and traefik will forwrd those requests to gitea container's port 3000, sorta. actually it doesn't forward the traffic directly to container port because we are first terimnating TLS, then routing the traffic. [more about ssl/tls here](#securing-access-with-ssltls)


### traefik

{{< tabs "traefik-setup">}}
{{< tab "docker-compose.yml" >}}

```yaml
## https://github.com/karvounis/traefik-tutorial-docker-compose-files/blob/master/standalone/advanced/docker-compose.dashboard.yml

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
{{< /tab >}}
{{< tab ".env" >}}
you would need this environment file to go with it:


```shell
> cat << EOF > /my/network/share/gitea/.env
  COMPOSE_PROJECT_NAME=traefik
  HOST_DNS=beebox.lan
  DYNAMIC_CONFIG=./config.yml
  CERTIFICATES_DIR=./certs/
  TRAEFIK_VERSION=2.10
  EOF
```
{{< /tab >}}
{{< /tabs >}}
notice that in addition to treafik, we're also deploying sock-proxy for additional security. and while we allow traefik access to socketproxy via _socket\_proxy network_, we also expose traefik on _traefik\_public_ network. the latter is the network we would use for treafik to discover other services, as we did with [gitea](#gitea)

## Securing Access with SSL/TLS

- Explain the importance of SSL/TLS for secure communication.
- Detail the process of creating a root certificate with Minica.
- Describe how you implemented SSL/TLS for accessing Gitea and other services.

### minica

## Routing and Load Balancing with Traefik

Discuss the role of Traefik in routing and load balancing.
Explain how you configured Traefik to route traffic to different services based on hostnames.
Highlight any challenges you encountered and how you resolved them.

talk about running dns server on the same host that runs traefik and how it wouldn't resolve if bridge, and wouldn't resolve from host to mcvlan so better to separate the hosts.

## Continuous Integration and Deployment with Gitea Runner

Explain the concept of continuous integration and deployment (CI/CD).
Describe how you set up Gitea Runner to automate the build and deployment process.
Discuss the benefits of automating your development workflow.
## Version Control and Docker Repository Management with Gitea

Discuss the importance of version control and Docker repository management.
Explain how Gitea serves as your version control system and Docker repository.
Highlight any features of Gitea that were particularly useful for your workflow.

## Conclusion

Summarize the key points of your blog post.
Reflect on the benefits of your development process.
Encourage readers to consider similar approaches in their own projects.

## References and Further Reading

Provide links to relevant documentation, tutorials, and resources.
Acknowledge any external sources or individuals who contributed to your process.