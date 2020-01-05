# Traefik v2 </br>guide by examples

requirements

- have docker running somewhere
- have a domain `whateverblablabla.org`
- use cloudflare to manage DNS of the domain
- have 80/443 ports open

chapters

1. [traefik routing to docker containers](#1-traefik-routing-to-various-docker-containers)
2. [traefik routing to a local IP addresses](#2-traefik-routing-to-a-local-IP-addresses)
3. [middlewares](#3-middlewares)
4. [let's encrypt certificate html challange](#4-lets-encrypt-certificate-html-challange)
5. [let's encrypt certificate DNS challange](#5-lets-encrypt-certificate-DNS-challange-on-cloudflare)

# #1 traefik routing to various docker containers

![traefik-dashboard-pic](https://i.imgur.com/5jKHJmm.png)


- **create a new docker network** `docker network create traefik_net`.
Traefik and the containers need to be on the same network.
Compose creates one automaticly, but there is potential for fuckups later.
Better to just create own network and set it as default in every compose file.

   *extra info:* use `docker network inspect traefik_net` to see containers connected to that network

- **create traefik.yml**(traefik.toml) contains so called static traefik configuration.</br>
Dashboard is enabled in it.</br>
Entrypoints are defined, in this case one called `web` for port 80.</br>
Providers are defined , in this case docker.</br>
Since exposedbydefault is set to false, a label `"traefik.enable=true"` will be needed for containers that should be routed by traefik.</br>
This file will be passed to a docker container using bind mount, this will be done when we get to docker-compose.yml for traefik.

    `traefik.yml`
    ```
    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
    ```

   later on when traefik container is running, use command `docker logs traefik` and check if there is notice stating:
   `"Configuration loaded from file: /traefik.yml"`. Or run command `docker inspect -f '{{ .Mounts }}' traefik`
   to see active mounts on a running container.
   You don't want to be the moron who makes changes to traefik.yml
   and it does nothing because the file is not actually being used.


- **create `.env`** file that contains variables (usernames, passwords, ip addreses, apikeys, domain names,..)
that will be available for docker-compose
as enviroment variables when running the `docker-compose up` command.</br>
This alows compose files to be moved from system to system more freely and changes are done to the .env file,
so there's smaller possibility for a fuckup when forgeting to change some variable in a big compose file.

    `.env`
    ```
    MY_DOMAIN=whateverblablabla.org
    DEFAULT_NETWORK=traefik_net
    ```

    *extra info:* command `docker-compose config` shows how will compose look
    with the variables filled in.</br>
    Also getting in to the container with `docker container exec -it traefik sh`
    and then `printenv` can be useful.

- **create traefik-docker-compose.yml** file. It's a simple typical compose file.
Port 80 is mapped since we want traefik to be in charge of whot comes on it - use it as entrypoint.
Port 8080 is for dashboard where traefik shows info and stats.
Mount of docker.sock is needed so it can actually do its job communicating with docker.
Mount of traefik.yml is what gives the static traefik configuration.
The default network is set to the one created in the first step, as it will be set in all other compose files.

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **run traefik-docker-compose.yml**</br>
`docker-compose -f traefik-docker-compose.yml up -d` will start the traefik container.

    traefik is running, you can check it at the ip:8080 where you get the dashboard 

    *extra info:*</br>
    Typicly you see guides having just single compose file called `docker-compose.yml`
    with several services/containers and then it is just `docker-compose up -d` to start it all.
    You don't even need to bother defining networks with that.
    But this time I prefer small and separate steps when learning new shit.
    So that's why going with custom named docker-compose files as it allows separation.

    *extra info2:*</br>
    What you can also see in tutorials is no mention of traefik.yml(traefik.toml)
    and stuff is just passed from docker-compose file using docker's commands or labels.</br>
    like this: `command: --api.insecure=true --providers.docker`</br>
    But that way compose files look much more messy and you still can't do everything from there,
    so you still sometimes need that traefik.yml.
    So for now nicely structured yml file can be easier to digest.</br>

- **add labels to containers** that you want traefik to route.</br>
Here are examples of whoami, nginx, apache, portainer.</br>
As is obvious from the patern, all that is needed is adding few self-explanatory labels.</br>
But just in case
  - first label enables traefik
  - second label defines router named `whoami` that listens on entrypoint web(port 80)
  - third label defines a rule for this `whoami` router, specificly that when url
    equals `whoami.whateverblablabla.org` (the domain name comes from the .env file),
    that it means for router to do its job and route it.
  - fourth label does not exists, but pure absence of anything else is worth mentioning,
    nothing else is said but traefik knows to route trafik to that container and all the details.

    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=web"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=web"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `apache-docker-compose.yml`
    ```
    version: "3.7"

    services:
      apache:
        image: httpd:latest
        container_name: apache
        hostname: apache
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.apache.entrypoints=web"
          - "traefik.http.routers.apache.rule=Host(`apache.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `portainer-docker-compose.yml`
    ```
    version: "3.7"

    services:
      portainer:
        image: portainer/portainer
        container_name: portainer
        hostname: portainer
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - portainer_data:/data
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.portainer.entrypoints=web"
          - "traefik.http.routers.portainer.rule=Host(`portainer.$MY_DOMAIN`)"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK

    volumes:
      portainer_data:

    ```

- **run the damn containers**</br>
    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>
    `docker-compose -f apache-docker-compose.yml up -d`</br>
    `docker-compose -f portainer-docker-compose.yml up -d`


# #2 traefik routing to a local IP addresses

When url should aim at something other than a docker container.

![simple-network-diagram-pic](https://i.imgur.com/lTpUvWJ.png)

- **define a file provider, add required routing and service**

  This can't be defined using labels or commands in the compose file,
  so a file provider is needed.
  Somewhat common is to set traefik.yml itself as a file provider,
  or any new yml file can be created containing the dynamic config section.</br>
  Under providers theres a new `file` section and `traefik.yml` itself is set.</br>
  Then dynamic configuration stuff is added.</br>
  A router named `route-to-local-ip` with a simple subdomain hostname rule.
  What fits that rule, in this case exact url `test.whateverblablabla.org`,
  is send to the loadbalancer service which just routes it a specific IP and specific port.

    `traefik.yml`
    ```
    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
      file:
        filename: "traefik.yml"

    ## DYNAMIC CONFIGURATION
    http:
      routers:
        route-to-local-ip:
          rule: "Host(`test.whateverblablabla.org`)"
          service: route-to-local-ip-service
          priority: 1000
          entryPoints:
            - web

      services:
        route-to-local-ip-service:
          loadBalancer:
            servers:
              - url: "http://10.0.19.5:80"
    ```

    Priority of the router is set to 1000, a very high value,
    beating any possible other routers,
    like one we use later for doing global http -> https redirect.

- **run traefik-docker-compose** and it will work

    
# #3 middlewares

Example of an authentification middleware for any container.

![logic-pic](https://i.imgur.com/QkfPYel.png)

- **create a new file - `users_credentials`** containing username:passwords pairs,
 [htpasswd](https://www.htaccesstools.com/htpasswd-generator/) style</br>
 Bellow example has password `krakatoa` set to all 3 accounts

    `users_credentials`
    ```
    me:$apr1$L0RIz/oA$Fr7c.2.6R1JXIhCiUI1JF0
    admin:$apr1$ELgBQZx3$BFx7a9RIxh1Z0kiJG0juE/
    bastard:$apr1$gvhkVK.x$5rxoW.wkw1inm9ZIfB0zs1
    ```

- **mount users_credentials in traefik-docker-compose.yml**

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./users_credentials:/users_credentials:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add two labels to any container** that should have authentification</br>
  - The first label attaches new middleware called `auth-middleware`
    to an already existing `whoami` router.
  - The second label gives this middleware type basicauth,
    and tells it where is the file it should use to authenticate users.

    No need to mount the `users_credentials` here, it's traefik that needs that file
    and these lables are a way to pass info to traefik, what it should do
    in context of containers.

  `whoami-docker-compose.yml`
  ```
  version: "3.7"

  services:
    whoami:
      image: "containous/whoami"
      container_name: "whoami"
      hostname: "whoami"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.entrypoints=web"
        - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
        - "traefik.http.routers.whoami.middlewares=auth-middleware"
        - "traefik.http.middlewares.auth-middleware.basicauth.usersfile=/users_credentials"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `nginx-docker-compose.yml`
  ```
  version: "3.7"

  services:
    nginx:
      image: nginx:latest
      container_name: nginx
      hostname: nginx
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.entrypoints=web"
        - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
        - "traefik.http.routers.nginx.middlewares=auth-middleware"
        - "traefik.http.middlewares.auth-middleware.basicauth.usersfile=/users_credentials"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

- **run the damn containers** and now there is login and password needed

# #4 let's encrypt certificate, html challange

![letsencrypt-html-chalalnge-pic](https://i.imgur.com/yTshxC9.png)

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a file on the server containing cryptographic key
   that allows encrypted communication and can be also used to confirm the identity of the server</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiatie certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip addresse</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts at a specific place on the server.
  LE then asks DNS internet servers for `whateverblablabla.org` and that points to some IP address.
  LE looks at that IP address through ports 80/443 for the file containing that random text.

  If it's there then this proves that whoever asked for the certificate controls both
  the server and the domain, since it showed control over DNS records.
  Certificate is given and is valid for 3 months, traefik will automaticly try to renew
  when less than 30 days is remaining.

  Now how to actually get it done.


- **create an empty acme.json file with 600 permissions**

  This file will store the certificates and all the info about them.

  `touch acme.json && chmod 600 acme.json`

- **add 443 entrypoint and certificate resolver to traefik.yml**</br>

  In entrypoint section new entrypoint is added called websecure, port 443
  
  certificatesResolvers is a configuration section that tells traefik
  how to use acme or other resolvers to get certifaces.</br>
  - the name of the resolver is `lets-encr` and uses acme
  - commented out staging caServer makes LE issue a staging certificate,
    it is an invalid certificate and wont give green lock but has no limitations,
    if it's working it will say issued by let's encrypt.
  - Storage tells where to store given certificates - `acme.json`
  - The email is where LE sends notification about certificates expiring
  - httpChallenge is given entry point,
   so acme knows its doing http challange over port 80

  That is all that is needed for acme, rest has traefik pre-configured

    `traefik.yml`
    ```
    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"
      websecure:
        address: ":443"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false

    certificatesResolvers:
      lets-encr:
        acme:
          #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
          storage: acme.json
          email: whatever@gmail.com
          httpChallenge:
            entryPoint: web
    ```

- **expose/map port 443 and mount acme.json** in traefik-docker-compose.yml 

  Notice that acme.json is not :ro - read only

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add required labels to containers**</br>
compared to just plain http from first chapter,
it is just changing router's entryPoint from `web` to `websecure`
and assigning certificate resolver named `lets-encr` to the router named `whoami`

    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```
- **run the damn containers**</br>
give it a minute</br>
containers will now work only over https and have the greenlock</br>


- **reddirect http traffic to https** using middleware.</br>
http stops working with this https setup, no point in making it work,
better to just make http redirected to https.</br>
There are several places that this can be declared, in traefik.yml in dynamic section,
or using labels in any running container.</br>
This example goes with labels in traefik compose.

  - First label enables traefik for traefik container, otherwise other labels would be ignored.</br>
  - Second label creates new middleware called `redirect-to-https` and assigns it scheme `https`.</br>
  - Third label creates new router called `redirs`, with a rule that uses regex
    to catch any and every request entering traefik based on the url.
  - Forth label declares on which entrypoint this all is applied to.</br>
  - Fifth label assing the redirect middleware to that `redirs` router.</br>
    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"
        labels:
          - "traefik.enable=true"
          - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
          - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
          - "traefik.http.routers.redirs.entrypoints=web"
          - "traefik.http.routers.redirs.middlewares=redirect-to-https"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK

    ```

  You only want this declared once, not in every container.
 
# #5 let's encrypt certificate DNS challange on cloudflare

![letsencrypt-html-chalalnge-pic](https://i.imgur.com/dkgxFTR.png)

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a file on the server containing cryptographic key
   that allows encrypted communication and can be also used to confirm the identity of the server</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiatie certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip addresse</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts as a new DNS txt record.
  LE then checks `whateverblablabla.org` DNS records to see if the text is there.
  
  If it's there then this proves that whoever asked for the certificate controls the domain,
  since it showed control over DNS.
  Certificate is given and is valid for 3 months. Traefik will automaticly try to renew
  when less than 30 days is remaining.

  Benefit over httpChallenge is ability to have wildcard certificates.
  These are certificates that validate for example all subdomains `*.whateverblablabla.org`</br>
  Also no ports are needed to be open.

  But traefik needs to be able to make changes to DNS records,
  for this there needs to be support on the DNS control provider site.

  Now how to actually get it done.

- **create an empty acme.json file with 600 permissions**

  `touch acme.json && chmod 600 acme.json`

- **add 443 entrypoint and certificate resolver to traefik.yml**</br>

  In entrypoint section new entrypoint is added called websecure, port 443
  
  certificatesResolvers is a configuration section that tells traefik
  how to use acme or other resolvers to get certifaces.</br>
  - the name of the resolver is `lets-encr` and uses acme
  - Storage tells where to store given certificates - `acme.json`
  - The email is where LE sends notification about certificates expiring
  - dnsChallenge is specified with a [provider](https://docs.traefik.io/https/acme/#providers),
    in this case cloudflare, each provider needs differently named enviroment variable
    in the .env file, but thats later, here it just needs the name of the provider
  - resolvers are IP of well known DNS servers to use during challange
  - commented out staging caServer makes LE issue a staging certificate,
    it is an invalid certificate and wont give green lock but has no limitations,
    if it's working it will say issued by let's encrypt.

  `traefik.yml`
  ```
  api:
    insecure: true
    dashboard: true

  entryPoints:
    web:
      address: ":80"
    websecure:
      address: ":443"

  providers:
    docker:
      endpoint: "unix:///var/run/docker.sock"
      exposedByDefault: false

  certificatesResolvers:
    lets-encr:
      acme:
        #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
        email: whatever@gmail.com
        storage: acme.json
        dnsChallenge:
          provider: cloudflare
          resolvers:
            - "1.1.1.1:53"
            - "8.8.8.8:53"
  ```

- **to the `.env` file add required variables**</br>
We know what variables to add based on the [list of supported providers](https://docs.traefik.io/https/acme/#providers).</br>
For cloudflare variables are 
  - `CF_API_EMAIL` - cloudflare login
  - `CF_API_KEY` - global api key

  `.env`
  ```
  MY_DOMAIN=whateverblablabla.org
  DEFAULT_NETWORK=traefik_net
  CF_API_EMAIL=whateverbastard@gmail.com
  CF_API_KEY=8c08c85dadb0f8f0c63efe94fx155b6ve1abc
  ```

- **expose/map port 443 and mount acme.json** in traefik-docker-compose.yml 

  Notice that acme.json is not :ro - read only

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **add required labels to containers**</br>
  compared to just plain http from first chapter,
  it is changing router's entryPoint from `web` to `websecure`
  and assigning certificate resolver named `lets-encr` to the router named `whoami`.</br>
  Two domain labels define the main domain, and the subdomain we are trying to get certificate for.
  All domains with labels must also have dns record and it must be pointing to traefik.
  
    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"
          - "traefik.http.routers.whoami.tls.domains[0].main=whoami.$MY_DOMAIN"


    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"
          - "traefik.http.routers.nginx.tls.domains[0].main=nginx.$MY_DOMAIN"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```
- **run the damn containers**</br>
    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>

- **Fuck that, the whole point of DNS challange is to get wildcards!**</br>
  fair enough</br>
  so for wildcard run the relevant labels in traefik compose.
  - same `lets-encr` certificateresolver is used as before, the one defined in traefik.yml
  - the wildcard for subdomains(*.whateverblablabla.org) is set as the main domain to get certificate for
  - the naked domain(just plain whateverblablabla.org) is set as sans(Subject Alternative Name)
  - again, you do need `*.whateverblablabla.org` and `whateverblablabla.org` set in your DNS control panel, pointing to IP of traefik

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.traefik.tls.certresolver=lets-encr"
          - "traefik.http.routers.traefik.tls.domains[0].main=*.$MY_DOMAIN"
          - "traefik.http.routers.traefik.tls.domains[0].sans=$MY_DOMAIN"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    Now all other containers just need to have certificate resolver and websecure entry

    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    Here is apache but this time run on the naked domain `whateverblablabla.org`</br>
    `apache-docker-compose.yml`
    ```
    version: "3.7"

    services:
      apache:
        image: httpd:latest
        container_name: apache
        hostname: apache
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.apache.entrypoints=websecure"
          - "traefik.http.routers.apache.rule=Host(`$MY_DOMAIN`)"
          - "traefik.http.routers.apache.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **reddirect http traffic to https** using middleware.</br>
same as before with http challange, http stops working,
adding redirect middleware to the docker compose.

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"
        labels:
          - "traefik.enable=true"
          # DNS WILDCARD CHALLANGE
          - "traefik.http.routers.traefik.tls.certresolver=lets-encr"
          - "traefik.http.routers.traefik.tls.domains[0].main=*.$MY_DOMAIN"
          - "traefik.http.routers.traefik.tls.domains[0].sans=$MY_DOMAIN"
          # HTTP TO HTTPS REDDIRECT
          - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
          - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
          - "traefik.http.routers.redirs.entrypoints=web"
          - "traefik.http.routers.redirs.middlewares=redirect-to-https"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

# #stuff to checkout
  - [when file provider is used for managing docker containers](https://github.com/pamendoz/personalDockerCompose)
  - [traefik v2 forums](https://community.containo.us/c/traefik/traefik-v2)
  - ['traefik official site blog](https://containo.us/blog/)
  - 


  intersting setup to check out
