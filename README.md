Containers On Demand

For those who are using Traefik as a reverse-proxy solution - there is Ondemand Plugin available!
Link to plugin
https://plugins.traefik.io/.../628c9.../containers-on-demand

Short-ish instruction from me:

1. if you configure your reverse proxy using Lables in docker-compose files, you have a problem. In a regular docker environment, Traefik abandons a container and it's associated routers\middlewares\services once its scaled to 0. This can be avoided in several ways:
-- going Swarm - service scaled to 0 but its load balancer not stopped
-- going k8s - bit too complicated for this short
-- defining your apps in dynamic file configuration, which i will be expanding on further.

1.1 Install the plugin - "Install" button on Plugins.traefik.io has sufficient instructions about that - either file of dynamic configuration or lables will work!

1.2 Create a Middleware. For questions - middleware its a function used when traffic hits your entripoint and recognized as following one of your setup rules, and said function does something to said traffic. Adds TLS, redirects to HTTPS, etc.

ive added next to my Middlewares.toml
https://github.com/flargargoyl/traefik-ondemand-plugin/blob/main/middlewares.toml

    [http.middlewares.my-traefik-ondemand-plugin]
      [http.middlewares.my-traefik-ondemand-plugin.plugin]
        [http.middlewares.my-traefik-ondemand-plugin.plugin.traefik-ondemand-plugin]
          name = "TRAEFIK_HACKATHON_whoami"
          serviceUrl = "http://ondemand:10000"
          timeout = "1m"

1.3 you need ondemand service running - it will listen to the defined endpoints, and if a container you are trying to reach is stopped, it will show you a nice self-refreshing window while that's starting

https://github.com/.../traef.../blob/main/docker-compose.yml

2. Create your app's file. here's an example of whoemi.yml
https://github.com/flargargoyl/traefik-ondemand-plugin/blob/main/ondemand-whoami.yml

http:
  routers:
    whoami:
      rule: HostHeader(`whoami.$DOMAINNAME`)
      entryPoints:
        - "https"
      middlewares:
        - ondemand_whoami@docker
# Any other midlewares should work as well, as long as you have them in your middlewares or middlewares-chains files, apply as example below with your names:
#        - chain-oauth 
#        - middlewares-rate-limit
#        - middlewares-secure-headers
#        - middlewares-oauth
      service: "whoami"
      tls: {}

  services:
    whoami:
      loadBalancer:
        servers:
        - url: "http://whoami:80"


Obviously replace your domainname and watch for the naming convention.
As you may see, it supports other middlewares and even chains of them!

Now, in your docker-compose file:
https://github.com/flargargoyl/traefik-ondemand-plugin/blob/main/docker-compose.yml

  ondemand:
    image: ghcr.io/acouvreur/traefik-ondemand-service:1
    <<: *common-keys-core
    container_name: ondemand
    command: 
      - --swarmMode=false
    volumes: #I have not yet checked how it works with Socket-proxy but i will and higly suggest you would rather use that than just mount your docker socket directly, for security reasons!
      - '/var/run/docker.sock:/var/run/docker.sock' 
    labels:
      - traefik.enable=true
      - traefik.http.middlewares.ondemand_whoami.plugin.traefik-ondemand-plugin.name=whoami
      - traefik.http.middlewares.ondemand_whoami.plugin.traefik-ondemand-plugin.serviceUrl=http://ondemand:10000
      - traefik.http.middlewares.ondemand_whoami.plugin.traefik-ondemand-plugin.timeout=1m

whoami:
    image: containous/whoami
    networks:
      - t2_proxy
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped 
    container_name: whoami
    environment: #port 80
      PUID: $PUID
      PGID: $PGID
      TZ: $TZ


Make sure Traefik, Ondemand and the service you want to be ondemand are in the same docker network (and preferably the same file - i had some weird issues if a service was in a separate docker compose file :C )

Launch them all, and in your Traefik dashboard you should search for your entrypoint, and see 2 for each:

one would be named whoami-docker@docker (as it's defined in LABLES) and another one would be defined as whoam@file.
When the container stops, the route @docker will be removed, but the other one, from the file would still allow trafik to receive, identify, match the rule, and ask Ondemand - "Please, start whoami container" which it'll do.

Result? Bit confusing setup as for my liking, however you can have services you don't need online 24\7 be only available exactly when you need then - and stowed away when you don't use them for some time. Timeout of such is configurable, as well as lots of the other things as per documentation

https://github.com/acouvreur/traefik-ondemand-plugin

Additional cons - when you create all those services, they somewhy not always processed correctly from a first try. So what i do, is when i'm sure i can access the service, i'll stop the container and try to access it again. When container is started by Ondemand service\middleware, it'll be for sure capable of stopping them and starting them further on. As a result you may have less resource expenses for when you don't use it. Personally i use it for relatively lightweight services like whoami or hastebin, but also for kmvtoolnix and qdirstat - heavy one, but which i don't need always on.
Keep in mind you want to setup a reasonable timeout for those to be shut down after, as you may use mkv for an hour, packing some files, for example.
