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

