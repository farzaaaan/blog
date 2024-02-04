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

