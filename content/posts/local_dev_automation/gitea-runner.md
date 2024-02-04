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