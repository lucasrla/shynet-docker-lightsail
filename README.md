# Shynet on Lightsail + Cloudflare with `docker-compose`

Run [Shynet](https://github.com/milesmcc/shynet) on Amazon Lightsail and Cloudflare paying just $3.50 per month. As far as I can tell, that is the cheapest way to run a privacy-conscious analytics on your personal website.


# Overview

### Basic Setup
1. [Have a subdomain available (or register a new one)](#1-have-a-subdomain-available-or-register-a-new-one)
2. [Create an Amazon Lightsail instance running Ubuntu 18.04](#2-create-an-amazon-lightsail-instance-running-ubuntu-1804)
3. [Install Shynet via `docker-compose`](#3-install-shynet-via-docker-compose)
4. [Setup Cloudflare as reverse proxy (with SSL/TLS)](#4-setup-cloudflare-as-reverse-proxy-with-ssltls)
5. [Use Shynet](#5-use-shynet)

Keep in mind that this guide is for the most basic setup possible. For a deep dive, please follow [Shynet's own installation guides](https://github.com/milesmcc/shynet/blob/master/GUIDE.md#installation).

### Maintenance
- [Upgrading Shynet](#upgrading-shynet)
- [Cleaning up exited/unused containers and dangling images](#cleaning-up-exitedunused-containers-and-dangling-images)


# Setup

## 1. Have a subdomain available (or register a new one)

We need a domain (or subdomain) to host both the Shynet dashboard and the script it provides. You most likely already have one, otherwise you wouldn't be looking for an analytics service.

In this guide I will assume you already have your own `YOUR_WEBSITE.COM` domain and that we are going to run Shynet from a subdomain, `shynet.YOUR_WEBSITE.COM`.

In case you don't have any domain available, get one from a registrar first. There are many to choose from: [Cloudflare](https://www.cloudflare.com/products/registrar/), [Google Domains](https://domains.google), [AWS Route 53](https://aws.amazon.com/route53/), [GoDaddy](https://www.godaddy.com), etc.

## 2. Create an Amazon Lightsail instance running Ubuntu 18.04

If you don't have an AWS account, [create one first](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html). They offer a [generous free tier that includes one month of Lightsail for free](https://aws.amazon.com/free/).

1. Sign in to [Lightsail](https://aws.amazon.com/lightsail/) and click on `Create Instance`
2. Choose whatever region you prefer
3. Under `Select a blueprint` click on `OS Only` and choose the `Ubuntu 18.04 LTS` image
4. Click on `+ Add launch script`
5. Enter the following code:
   
   ```sh
   curl -o lightsail-startup-script.sh https://raw.githubusercontent.com/lucasrla/shynet-docker-lightsail/master/lightsail-startup-script.sh

   chmod +x ./lightsail-startup-script.sh

   ./lightsail-startup-script.sh
   ```
6. Check `Enable Automatic Snapshots` for automatic backups on a daily basis
7. Choose an instance size. Assuming our site is a small and personal website, let's go with the $3.50 option
8. Rename the instance, it could be something like `Shynet-Ubuntu`
9. Click on `Create`

Great! Our instance should be up and running in a few moments. 

Next, make its IP static:

1. Click on the instance and navigate to `Networking`
2. Make its `IP` static
3. Navigate to `Connect` and take note of the `public IP` and `user name` (which should be `ubuntu`)

Connect to our instance using our own SSH client:

1. Go to your Lightsail [account page](https://lightsail.aws.amazon.com/ls/webapp/account/keys) and download your `Default` SSH key (it is a `.pem` file)
2. If you don't have SSH ready on your computer, do it first. These guides by [DigitalOcean](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys) and [GitHub](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh) might be useful for you
3. Open the terminal in your computer (like, say, [iTerm2 for macOS](https://iterm2.com))
4. Enter the following commands:

    ```sh
    mv ~/Downloads/LightsailDefaultKey-XX-XXXX-X.pem ~/.ssh/
    chmod 400 ~/.ssh/LightsailDefaultKey-XX-XXXX-X.pem
    ssh -i ~/.ssh/LightsailDefaultKey-XX-XXXX-X.pem ubuntu@YOUR_INSTANCE_PUBLIC_IP_ADDRESS
    ```

    Change `LightsailDefaultKey-XX-XXXX-X.pem` to the `.pem` file name you just downloaded and `YOUR_INSTANCE_PUBLIC_IP_ADDRESS` to the `public IP` you took note.

You are now logged in to your instance.

Let's update its software and then restart it:

```sh
sudo apt update

sudo apt full-upgrade

# during the upgrade, whenever prompted
# select the `keep the local version currently installed` option

sudo shutdown -r 0
# you will be disconnected
```

## 3. Install Shynet via `docker-compose`

Start by setting the environment variables:

1. Download `TEMPLATE.env` from https://github.com/milesmcc/shynet/blob/master/TEMPLATE.env
2. Use `openssl rand -base64 48` on your terminal to create `A_LONG_AND_RANDOM_SECRET_KEY` and paste it as `DJANGO_SECRET_KEY`
3. Repeat the step above to create `A_LONG_AND_RANDOM_PASSWORD` and set it as `DB_PASSWORD`
4. Set your preferred `TIME_ZONE` using https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
5. Set `ALLOWED_HOSTS` to `YOUR_INSTANCE_PUBLIC_IP_ADDRESS,shynet.YOUR_WEBSITE.COM,0.0.0.0`
6. Save the edited file as `.env`

Copy our `.env` file to the server and then install Shynet:

```sh
scp -i ~/.ssh/LightsailDefaultKey-XX-XXXX-X.pem .env ubuntu@YOUR_INSTANCE_PUBLIC_IP_ADDRESS:~/

ssh -i ~/.ssh/LightsailDefaultKey-XX-XXXX-X.pem ubuntu@YOUR_INSTANCE_PUBLIC_IP_ADDRESS

sudo mv ~/.env /srv/docker/

cd /srv/docker/

# check if our .env were recognized by docker-compose
docker-compose config
  networks:
  internal: {}
  services:
  db:
    environment:
        POSTGRES_DB: shynet_db
        POSTGRES_PASSWORD: # A_LONG_AND_RANDOM_PASSWORD
        POSTGRES_USER: shynet_db_user
    image: postgres:12
    networks:
        internal: null
    restart: always
    volumes:
    - shynet_db:/var/lib/postgresql/data:rw
  shynet:
    depends_on:
    - db
    environment:
        ALLOWED_HOSTS: # YOUR_INSTANCE_PUBLIC_IP_ADDRESS,shynet.YOUR_WEBSITE.COM,0.0.0.0
        DB_HOST: db
        DB_NAME: shynet_db
        DB_PASSWORD: # A_LONG_AND_RANDOM_PASSWORD
        DB_PORT: '5432'
        DB_USER: shynet_db_user
        DJANGO_SECRET_KEY: # A_LONG_AND_RANDOM_SECRET_KEY
        ONLY_SUPERUSERS_CREATE: "True"
        PERFORM_CHECKS_AND_SETUP: "True"
        PORT: '8080'
        SCRIPT_HEARTBEAT_FREQUENCY: '5000'
        SCRIPT_USE_HTTPS: "True"
        SESSION_MEMORY_TIMEOUT: '1800'
        ACCOUNT_SIGNUPS_ENABLED: "False"
        TIME_ZONE: # YOUR_PREFERRED_TIMEZONE
    expose:
    - 8080
    image: milesmcc/shynet:latest
    networks:
        internal: null
    ports:
    - 80:8080/tcp        
    restart: unless-stopped
  version: '3.0'
  volumes:
  shynet_db: {}

docker-compose -f docker-compose.yml up -d
  Creating network "docker_internal" with the default driver
  Creating volume "docker_shynet_db" with default driver
  Pulling db (postgres:)...
  latest: Pulling from library/postgres
  ...
  Status: Downloaded newer image for milesmcc/shynet:latest
  Creating docker_db_1 ... done
  Creating docker_shynet_1 ... done

# you should see the two containers up and running
docker ps

# now, check if the database container is running as intented
docker exec -t -i POSTGRES_CONTAINER_ID bash

psql --host=db --username=shynet_db_user --dbname=shynet_db
  Password for user shynet_db_user: # enter your A_LONG_AND_RANDOM_PASSWORD

# double check the configurations
\conninfo
  You are connected to database "shynet_db" as user "shynet_db_user" on host "db" (address "172.18.0.2") at port "5432".
  # if you see something similar, you are good to go!

# quit psql
\q 

# and then leave the container
exit

# take note of the shynet container ID
docker ps

# configure it with your email
# and take note of the temporary password that will be printed
docker exec -t -i SHYNET_CONTAINER_ID ./manage.py registeradmin YOUR_EMAIL@DOMAIN.COM
  Successfully created a Shynet superuser
  Email address: # YOUR_EMAIL@DOMAIN.COM
  Password: # YOUR_TEMP_PASSWORD

docker exec -t -i SHYNET_CONTAINER_ID ./manage.py hostname shynet.YOUR_WEBSITE.COM
  Successfully set the hostname to 'shynet.YOUR_WEBSITE.COM'

docker exec -t -i SHYNET_CONTAINER_ID ./manage.py whitelabel "shynet @ YOUR_WEBSITE.COM"
  Successfully set the whitelabel to 'shynet @ YOUR_WEBSITE.COM'
```

Open `YOUR_INSTANCE_PUBLIC_IP_ADDRESS` in your browser. Shynet should be up and running there. Login with the credentials from the `registeradmin` step above. Then, under `Account > Emails` click on `Resend Verification`.

Next, go back to your terminal to check out the "confirmation email" we should have just received:

```sh
# assuming you are still logged in to your lightsail instance

docker ps

# you should be able to read the "confirmation email" in our logs
docker logs SHYNET_CONTAINER_ID --tail 20
  ...
  Content-Type: text/plain; charset="utf-8"
  MIME-Version: 1.0
  Content-Transfer-Encoding: 7bit
  Subject: Confirm Email Address
  From: Shynet <noreply@shynet.example.com>
  To: # YOUR_EMAIL
  Date: # ...
  Message-ID: # ...

  Hi there,

  You are receiving this email because YOUR_EMAIL has listed this email as a valid contact address for their account.

  To confirm this is correct, go to # A_CONFIRMATION_URL

  Thank you,
  # shynet @ YOUR_WEBSITE.COM
  -------------------------------------------------------------------------------
```

Copy and paste the `A_CONFIRMATION_URL` on your browser to verify your email.

## 4. Setup Cloudflare as reverse proxy (with SSL/TLS)

Cloudflare is an excellent service that includes CDN, attack/DDoS protection and SSL/TLS certificates for HTTPS. Simply put, your website will be faster and more secure. And you can have it all for free! 

The caveat: you will need to migrate the DNS of your root domain (i.e. `YOUR_WEBSITE.COM`) to Cloudflare. More specifically, you will need to move your current name servers to theirs. Fear not, this migration should be very easy, and take ~30 minutes to happen. Simply follow [their instructions](https://support.cloudflare.com/hc/en-us/articles/201720164-Creating-a-Cloudflare-account-and-adding-a-website).

After you have finished the DNS migration, do the following:

1. Under `DNS`, click on `+ Add record` and enter `A` type, `shynet` for name and `YOUR_INSTANCE_PUBLIC_IP_ADDRESS` for `IPv4 address`. Keep `TTL` set to `Auto` and `Proxy status` as `Proxied`
2. Click `Save` and wait 10-20 minutes
3. Navigate to `shynet.YOUR_WEBSITE.COM`

Voilà! Shynet should be accessible there.

Then tweak it to add `HTTPS`:

- Under `SSL/TLS`, select your `SSL/TLS Encryption Mode` to `Flexible`

> Strictly speaking it is better to have end-to-end encryption everywhere. But recall that this is a basic guide intended only for hobbyist websites that do not collect anything from their users. 

Wait more 10-20 minutes and visit `shynet.MY_WEBSITE_URL.com` again. You should have been redirected to `HTTPS` automatically. Check it yourself by clicking on the lock in your browser address bar. There should be something like "Connection is secure/encrypted" and a certificate issued by Cloudflare.

To strengthen the security of your Shynet dashboard, follow the instructions in the [Shynet's own usage guide](https://github.com/milesmcc/shynet/blob/master/GUIDE.md#cloudflare).


## 5. Use Shynet

Login to `shynet.YOUR_WEBSITE.com`. Click on `+ New Service` and create a analytics service for `YOUR_WEBSITE.com`. 

Finally, install your tracking script. Place the following snippet at the end of the `<body>` tag on any `html` page you'd like to track:

```html
<noscript><img src="https://shynet.YOUR_WEBSITE.com/ingress/XXXXXXXXXXXXX/pixel.gif"></noscript>
<script src="https://shynet.YOUR_WEBSITE.com/ingress/XXXXXXXXXXXXX/script.js"></script>
```

Add the tracking script to your code, deploy it and visit `MY_WEBSITE_URL.com`. Your hit should appear at `shynet.MY_WEBSITE_URL.com` a few moments later.


# Maintenance

## Upgrading Shynet

Whenever there is a [new version](https://github.com/milesmcc/shynet/releases) of Shynet released on [docker hub](https://hub.docker.com/r/milesmcc/shynet), you should be able to update it (most of the time) with:

```sh
# login to your virtual server
ssh -i ~/.ssh/LightsailDefaultKey-XX-XXXX-X.pem ubuntu@YOUR_INSTANCE_PUBLIC_IP_ADDRESS

# list your current images
docker images

cd /srv/docker/

docker-compose pull
  Pulling db     ... done
  Pulling shynet ... done

# check our new images
docker images

# rebuild our containers and run them on detached mode
docker-compose -f docker-compose.yml up -d --remove-orphans

# check the results
docker ps

# leave the server
exit
```

## Cleaning up exited/unused containers and dangling images

```sh
# login to your server
ssh -i ~/.ssh/LightsailDefaultKey-XX-XXXX-X.pem ubuntu@YOUR_INSTANCE_PUBLIC_IP_ADDRESS

# list all containers, including exited/unused ones
docker container list -a
# which is the same as: docker ps -a

# remove exited/unused ones (by each id)
docker rm CONTAINER_ID

# check that they are gone
docker container list -a

# remove obsolete images
# (i.e. images that are not used by any existing container)
docker image prune
```

## Troubleshooting docker issues

```sh
# login to your server
ssh -i ~/.ssh/LightsailDefaultKey-XX-XXXX-X.pem ubuntu@YOUR_INSTANCE_PUBLIC_IP_ADDRESS

# check what is running
docker ps

# if there is anything different from "Up ..." in the STATUS column of a container,
# check the logs
docker logs --tail 20 --follow --timestamps CONTAINER_ID
```


# Other References

- [Deploying Docker Containers on Amazon Lightsail](https://www.youtube.com/watch?v=z525kfneC6E) by Mike Coleman
- [A couple ways to deploy an app on Amazon Lightsail](https://github.com/mikegcoleman/todo/blob/master/readme.md) also by Mike Coleman