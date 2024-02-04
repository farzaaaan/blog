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