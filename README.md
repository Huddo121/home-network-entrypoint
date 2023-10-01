# Home Network Entrypoint

Simple reverse proxy setup for exposing services to the internet, expected to be deployed to a server that is also directly hosting those service (e.g. you have a NUC, HTPC, or Raspberry Pi sitting around doing things).

Created by mixing together the steps from the following blog posts;

- https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71
- https://mindsers.blog/post/https-using-nginx-certbot-docker/

## Setup and first run

### Prerequisites

- [Docker](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/standalone/) installed
  - You may need to install Docker Compose for the `root` user in addition to your own personal user if you have set it up previously
- Systemd based OS
- Admin access to the server
- Port `80` and `443` on the server are free

### Create a new user to run the proxy

Rather than running the proxy as your user, or worse, `root`, I usually create a specific user for running some particular task.

- Create the user and give them a home directory
  - `sudo useradd -m -r -d /home/nginx nginx`
  - **NB:** if you want to change the name of the service account or the location of its home directory then you need to change the systemd unit described later.
- Give that user the ability to run docker containers
  - `usermod -aG docker nginx`

### Get a copy of this repo on to the server

- Start a shell with administrative access
  - `sudo /bin/bash`
- Jump in to the service user's home directory
  - `cd /home/nginx`
- Clone the repo
  - `git clone https://github.com/Huddo121/home-network-entrypoint.git proxy`

### Get the service running

- Add the systemd unit
  - `cp proxy/nginx.service /lib/systemd/system/`
- Have the nginx service start up on boot
  - `sudo systemctl enable nginx`
- Start up the service
  - `sudo systemctl start nginx`

## Exposing a new service

### Set up DNS

There's little point in following through with process unless you're exposing the computer to the wider internet, so I am going to assume you have a domain you'd like to use, how to point that domain at your server's IP address, and how to update the settings in your router to forward port `80` and `443` to the server.

### Getting certificates

At this point NGINX will be running and attached to port `80`, but there are no certificates for SSL set up. The NGINX configuration doesn't do anything but expose some folders that [Certbot](https://hub.docker.com/r/certbot/certbot) can write to in order to complete the [ACME challenges](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment) for certificate creation.

Decide on the domain that you'd like to expose. For this setup you need to get certificates for each domain or subdomain you'd like to expose as getting renewals working with wildcard domains requires integration with your DNS provider.

- Update the `conf.d/app.conf` file to listen on your chosen domain(s), and then tell the NGINX container to reload the config
  - `docker exec -it nginx-nginx-1 nginx -s reload`
- Generate the certificate for your chosen domain or subdomain
  - `docker compose run --rm certbot certonly --webroot --webroot-path /var/www/certbot -d my.chosen.domain.com`

### Proxying to your desired service

Now that you have certificates for your chosen domain, it's time to direct traffic to your chosen service. Because the `docker-compose.yml` file attaches NGINX to the `host` network, it can redirect traffic to any service that's exposed on the server.

You can add a block similar to the following in order to start proxying requests to your chosen service;

```
server {
    listen 443 ssl;
    server_name my.chosen.domain.com;

    ssl_certificate /etc/letsencrypt/live/my.chosen.domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/my.chosen.domain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;

    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    location / {

        proxy_pass http://localhost:8080;
    }
}
```

Make sure you update all of the appropriate places within the `server` directive with your chosen domain, forgetting one will result in strange errors otherwise.

Now you can tell NGINX to reload its configuration to put the proxying in to effect;

- `docker exec -it nginx-nginx-1 nginx -s reload`
