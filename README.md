# I Google™ this way to often
A repo that has short snippets/commands that i google all the time since it's faster to google than to type it out.

## Linux setup

### base
```shell
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install -y htop vim nload ca-certificates curl gnupg lsb-release wget
sudo useradd -m -G sudo -s /bin/bash nezu
echo "nezu ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/nezu
sudo -i -u nezu
mkdir -p ~/.ssh && curl -L -o ~/.ssh/authorized_keys "https://github.com/dumbasPL/dumbasPL/raw/master/id_rsa.pub" && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

### Docker

#### Install
Taken from: https://docs.docker.com/engine/install

Ubuntu:
```shell
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
```

Debian:
```shell
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
```

#### Log driver
`/etc/docker/daemon.json`
```json
{
  "log-driver": "local"
}
```

### Portainer
Taken from: https://docs.portainer.io/start/install/server

Standalone:  
```shell
docker volume create portainer_data
docker run -d -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
echo "https://$(curl ip.nezu.cc 2> /dev/null):9443/"
```

Swarm:  
```shell
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add portainer.portainer-data=true $NODE_ID
```

`portainer.yml`
```yaml
version: '3.8'
services:
  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]
  portainer:
    image: portainer/portainer-ce:latest
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9443:9443"
      - "9000:9000"
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
      #- proxy
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
          - node.labels.portainer.portainer-data == true
      #labels:
      #  - "traefik.enable=true"
      #  - "traefik.docker.network=proxy"
      #  - "traefik.http.routers.portainer.entrypoints=https"
      #  - "traefik.http.routers.portainer.rule=Host(`portainer.<DOMAIN>`)"
      #  - "traefik.http.routers.portainer.tls=true"
      #  - "traefik.http.routers.portainer.tls.certresolver=http"
      #  - "traefik.http.routers.portainer.service=portainer"
      #  - "traefik.http.services.portainer.loadbalancer.server.port=9000"
volumes:
  portainer_data:
networks:
  agent_network:
    driver: overlay
    attachable: true
  #proxy:
  #  external: true
```
```shell
docker stack deploy -c portainer.yml portainer
echo "https://$(curl ip.nezu.cc 2> /dev/null):9443/"
```

### Traefik

Standalone:  
```shell
docker network create proxy
sudo mkdir -p /srv/traefik
echo "{}" | sudo tee /srv/traefik/acme.json > /dev/null && sudo chmod 600 /srv/traefik/acme.json
```

`/srv/traefik/traefik.yml`
```yaml
entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: ":443"
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /config.yml
certificatesResolvers:
  http:
    acme:
      email: <EMAIL>
      storage: acme.json
      caServer: https://acme-staging-v02.api.letsencrypt.org/directory
      #caServer: https://acme-v02.api.letsencrypt.org/directory
      httpChallenge:
        entryPoint: http
```

`/srv/traefik/config.yml`
```yaml
http:
  routers:
    portainer:
      entryPoints:
        - "https"
      rule: "Host(`portainer.<DOMAIN>`)"
      tls:
        certResolver: http
      service: portainer
  services:
    portainer:
      loadBalancer:
        serversTransport: self-sign
        servers:
          - url: "https://host.docker.internal:9443"
  serversTransports:
    self-sign:
      insecureSkipVerify: true
```

`traefik.yml`
```yaml
version: '3.8'
services:
  traefik:
    image: traefik:v2.9
    container_name: traefik
    restart: always
    security_opt:
      - no-new-privileges:true
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - proxy
    ports:
      - 443:443
      - 80:80
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /srv/traefik/traefik.yml:/traefik.yml:ro
      - /srv/traefik/acme.json:/acme.json:rw
      - /srv/traefik/config.yml:/config.yml:ro
networks:
  proxy:
    external: true
```

Swarm:  
```shell
docker network create --driver=overlay proxy
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add traefik.traefik-certificates=true $NODE_ID
openssl passwd -apr1
```

`traefik.yml`
```yaml
version: '3.8'
services:
  traefik:
    image: traefik:v2.9
    networks:
      - proxy
    ports:
      - 80:80
      - 443:443
    deploy:
      placement:
        constraints:
          - node.labels.traefik.traefik-certificates == true
      labels:
        - traefik.enable=true
        - traefik.docker.network=traefik
        - traefik.http.routers.traefik.rule=Host(`traefik.iamagayperson.ml`)
        - traefik.http.routers.traefik.entrypoints=https
        - traefik.http.routers.traefik.tls=true
        - traefik.http.routers.traefik.service=api@internal
        - traefik.http.routers.traefik.tls.certresolver=http
        #- traefik.http.routers.traefik.middlewares=admin-auth
        #- traefik.http.middlewares.admin-auth.basicauth.users=nezu:$(openssl passwd -apr1)
        - traefik.http.services.dummy-traefik.loadbalancer.server.port=8080 # dummy service
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-certificates:/certificates
    command:
      - --providers.docker
      - --providers.docker.exposedbydefault=false
      - --providers.docker.swarmmode
      - --entrypoints.http.address=:80
      - --entrypoints.http.http.redirections.entryPoint.to=https
      - --entrypoints.http.http.redirections.entryPoint.scheme=https
      - --entrypoints.https.address=:443
      - --certificatesresolvers.http.acme.email=<EMAIL>
      - --certificatesresolvers.http.acme.storage=/certificates/acme.json
      #- --certificatesresolvers.http.acme.caServer=https://acme-staging-v02.api.letsencrypt.org/directory
      - --certificatesresolvers.http.acme.tlschallenge=true
      #- --accesslog
      - --log
      - --api
volumes:
  traefik-certificates:
networks:
  proxy:
    external: true
```

Example service:   
```yaml
version: '3.8'
services:
  test:
    image: nginx:latest
    #restart: always
    #security_opt:
    #  - no-new-privileges:true
    networks:
      - proxy
    #deploy:
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.test.entrypoints=https"
      - "traefik.http.routers.test.rule=Host(`test.<DOMAIN>`)"
      - "traefik.http.routers.test.tls=true"
      - "traefik.http.routers.test.tls.certresolver=http"
      - "traefik.http.routers.test.service=test"
      - "traefik.http.services.test.loadbalancer.server.port=80"
networks:
  proxy:
    external: true
```