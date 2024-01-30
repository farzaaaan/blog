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

I recently went through and set up a local environment for development in terms of automation. for this, I installed gitea for my version control. but didn't want to have access gitea over an ip, so I installed traefik and a dns server - pihole. then I tried accessing the docker repo on gitea but docker login needs ssl so I ended up creating a root certificate with minica. I create cnames from my dns server to the host of my traefik, then I used traefik to resolve the hostname to the right application and terminated the TLS at it and sent http to the final application, gitea.

I then installed gitea runner on a server that I intend to continuously update as I develop, and created a workflow in gitea to run an generate a new tagged docker image and automatically store it in the docker repo of gitea. I then had a dockercompose running on my dev machine that I could rerun and get the latest image.
 
## Introduction

- Introduce the purpose of your blog post.
- Explain why you decided to set up a local development environment and the tools you chose to use.

At some point in our tech life we start getting comfy in the cloud. we get used to the simplicity, and intuition of the environments. with how easy it to write a few lines of yaml and have another server, somewhere probably on the same continent as us, run it and it turn have another computer, probably somewhere else on the same continent, bring up a node of some sort and run our code on it. 

it's a luxury we often take for granted. as professionals we often interact with enterprise level configurations that cost millions a year and because of efficiency have likely made great efficiency to reproducible, repititive workflows that we often find ourselves consumers of. and as hobbyists, well it gets rediculously simpler even. two to five clicks and someone like me could deploy this blog on Cloudflare Pages, host the code on GitHub, and have an entire static site generator at his fingertip with a couple of cli lines. 

so yeah, life in the cloud is comfy and easy, and we get used to it. so it's no surprise I started finding myself having a hard time starting new projects on my local machine. every new project requires a certain amount of configuration. configurations that are often niche enough that I would forget once I stopped working on a project for while. which would then make debugging or going back to that project much harder and undesirable. I am speaking about those tiny environment variables that I needed for my dockerfiles to compile successfully. those pesky build contexts. and yes there are makefiles, but those also go only so far and often rely on input to do any complex task.

so I thought to myself what if I try to bring the cloud on my local network. what if instead of I having to keep configurations on my desktop or wsl and wrestling with mainting it... what if I have CI CD pipeliens running on my local repo? 

what if I could just: 

1.  
    ```bash
    mkcd ~/dev/cool_project && go mod init example.com/cool_project
    ```

2. 
    ```bash
    cat << EOF > ~/dev/cool_project/main.go
    package main

    import "fmt"

    func main() {
        fmt.Println("Hello, World!")
    }
    EOF 
    ```
3. 
    ```bash
    cat << EOF > ~/dev/cool_project/Dockerfile
    FROM golang:latest
    
    WORKDIR /app
    
    COPY . .
    
    RUN go build -o main .
    
    CMD ["./main"]
    EOF
    ```
4. 
    ```bash
    cat << EOF > ~/dev/cool_project/docker-compose.yml
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
5. 
    ```bash
    cat << EOF > .github/workflows/build-and-push.yml
    name: Build and Push Docker Image

    on:
      push:
        branches:
          - main

    jobs:
      build-and-push:
        runs-on: ubuntu-latest

        steps:
          - name: Checkout code
            uses: actions/checkout@v2

          - name: Login to Docker registry
            run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin gitea.local

          - name: Build and tag Docker image
            run: docker build -t cool_project .

          - name: Get build number
            id: get_build_number
            run: echo ::set-output name=build_number::\${GITHUB_RUN_NUMBER}

          - name: Tag Docker image with build number
            run: docker tag cool_project gitea.local/cool_project:\${{ steps.get_build_number.outputs.build_number }}

          - name: Tag Docker image as latest
            run: docker tag cool_project gitea.local/cool_project:latest

          - name: Push Docker images to registry
            run: |
              docker push gitea.local/cool_project:\${{ steps.get_build_number.outputs.build_number }}
              docker push gitea.local/cool_project:latest
    EOF
    ```
6. and from this point on every time I have a new change and push my change, I could just run `docker compose up -d` and see my change.
## Setting Up a Local Development Environment

Describe the steps you took to set up your local development environment.
Discuss the installation and configuration of Gitea, Traefik, and Pi-hole.

## Securing Access with SSL/TLS

- Explain the importance of SSL/TLS for secure communication.
- Detail the process of creating a root certificate with Minica.
- Describe how you implemented SSL/TLS for accessing Gitea and other services.

## Routing and Load Balancing with Traefik

Discuss the role of Traefik in routing and load balancing.
Explain how you configured Traefik to route traffic to different services based on hostnames.
Highlight any challenges you encountered and how you resolved them.

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