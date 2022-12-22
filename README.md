# I Google™ this way to often
A repo that has short snippets/commands that i google all the time since it's faster to google than to type it out.

## Linux setup

### base
```shell
sudo apt-get update && sudo apt-get install -y htop vim nload ca-certificates curl gnupg lsb-release wget
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
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

Swarm:  
`portainer-agent-stack.yml`
```yaml
version: '3.2'
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
      - "8000:8000"
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
volumes:
  portainer_data:
networks:
  agent_network:
    driver: overlay
    attachable: true
```
```shell
docker stack deploy -c portainer-agent-stack.yml portainer
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

Example service:   
```yaml
version: '3.8'
services:
  bdo-webtwo:
    image: nginx:latest
    restart: always
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
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