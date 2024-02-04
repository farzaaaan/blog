---
title: "continuous integration on local network"
slug: "continuous-integration-on-local"
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

there is a point where we all start getting comfy in the cloud. we get used to the simplicity and the intuitiveness of spinning up nodes with just a few lines of YAML, delegated to a data center somewhere else - probably on the same continent as us. 

but when working in our local environment things get comparatively more difficult. we can start a project, go through all the docs to configure our environment, work on it for a bit and then get pulled away from it for while. coming back to that project would then be much harder since it's very likely we've forgotten what we did right the last time we were focused on it. 

compare that to cloud where our projects, for the most part, are self-contained with their CI/CD pipelines. we could always make new commits to the repo and as long as the pipeline runs we know we are good. 

so I thought to myself what if I could bring the CI/CD methodology to my local? this would simplify the process of starting new projects locally, and would help me maintain them somewhere else that's not my desktop.


{{< inject "intro-goal.md" >}}


## Setting Up a Local Development Environment

at the heart of this setup sits a version control service, one that would ideally also offer the ability to run pipelines and host package repositories. on the cloud we have GitHub, Azure DevOps and the likes. we could theoritically run self-hosted agents for them on our local network, but then we would need to expose our local network to public, with it comes numerous security and networking considerations such a firewall and ddns. this is a huge undertaking, the goal here is to make life easier, so cloud repositories are a pass.

to achieve what I'm after I need: 
1. [git and package repository](#git-and-package-repository), 
2. [reverse proxy](#reverse-proxy), 
3. [a dns server](#dns-server), 
4. [and a custom root certificate to go with my setup](#securing-access-with-ssltls).

before we begin, you could find all the source code [here](https://github.com/farzaaaan/local_continuous_integration)



### Git and Package Repository

{{< inject "gitea.md" >}}


### Reverse Proxy

{{< inject "traefik.md" >}}

### DNS Server

{{< inject "pihole.md" >}}

### Securing Access with SSL/TLS

{{< inject "minica.md" >}}


## Continuous Integration and Deployment with Gitea Runner


{{< inject "gitea-runner.md" >}}

## Conclusion

it was quite a bit of work to get to this point, but I look at it as an investment for the future. considering gitea uses github actions syntax and approach, expanding on our current setup would be relatively simple in that we could customize our local cicd infrastructure by creating custom actions and chaining them together.

hope you enjoyed this!

## References and Further Reading

- [Gitea Documentation](https://docs.gitea.com/category/installation)
- [Gitea Act Runner](https://docs.gitea.com/next/usage/actions/act-runner/)
- [Gitea Act Container with Docker](https://github.com/catthehacker/docker_images)
- [Docker BuildX](https://docs.docker.com/build/ci/github-actions/)
- [Traefik Docker Installation](https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-official-docker-image)
- [Traefik Reference](https://doc.traefik.io/traefik/reference/static-configuration/overview/)
