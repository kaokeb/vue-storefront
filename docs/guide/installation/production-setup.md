# Production setup

If you’d like to start developing sites using Vue Storefront, you should start with the [Installation guide](linux-mac.md). For development purposes, you'll likely use the `yarn install` / `npm run installer` sequence, which will set up Vue Storefront locally using the automated installer and prepared Docker images for having Elasticsearch and Redis support.

Development mode means you're using a node.js-based server as HTTP service and running the app on the `3000` TCP port. As it's great for local testing, it's not recommended to use the installer and direct-user access to node.js in production configurations.

## Production setup: Bare VPS

To run Vue Storefront in production mode without Docker/Kubernetes, you'll need the Virtual Private Server with `root` access (for setup purposes). We assume that you're using `Debian GNU Linux` in the following steps.

Assumptions for the rest of this tutorial:

- You have root access to a Debian Linux machine.
- We'll be using the default local ports `3000` for [`vue-storefront`](https://github.com/DivanteLtd/vue-storefront) and `8080` for [`vue-storefront-api`](https://github.com/DivanteLtd/vue-storefront-api); the ports **should not be exposed**, as they will be hidden behind **NGINX proxy**.
- We're using **prod.vuestorefront.io** as a domain name. Please replace it with your host URL address.
- We assume that you have an SSL certificate for **prod.vuestorefront.io** (or your domain, of course). SSL encryption is required for PWA and Service Workers.

General Solution Architecture:
_USER -> NGINX proxy -> vue-storefront / vue-storefront-api_

We'll be hiding the `vue-storefront` and `vue-storefront-api` services behind the NGINX proxy. You can use NGINX for caching proxy, but in our case, it will just forward the requests without cache (as VS is pretty fast and caching is not required). The key features we're using are: SSL encryption, gzip-encoding, and URL routing (to merge `vue-storefront` and `vue-storefront-api` services under one domain).

### Prerequisites

Vue Storefront requires **Elasticsearch** and the **Redis server** to be installed. By default, in the development mode, both dependencies are provided with the `docker-compose.yml` Docker images. However, for production purposes, we recommend installing the servers natively.

For the purpose of this tutorial, we will use default packages distributed along with Debian operating systems, without any security hardening, config hardening operations.

**Please make sure** that your security/devops team has taken a look at the configs you're using and do harden the server configuration before launching your app publicly!

First, let's create the user (as root user):

```bash
mkdir /home/www
useradd -m -d /home/www/vuestorefront vuestorefront
```

Then install the Elasticsearch and Redis (as root user):

```bash
apt-get update
apt-get install curl
apt-get install git

curl -sL https://deb.nodesource.com/setup_8.x | bash -
apt-get install -y nodejs
npm install -g yarn

apt-get install redis-server

apt-get install openjdk-8-jre
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.9.deb
dpkg -i elasticsearch-5.6.9.deb
/etc/init.d/elasticsearch start

apt-get install imagemagick
apt-get install nginx
```

### Nginx

We decided to use **NGINX** as an HTTP proxy, exposed in front of the users, handling the network traffic, and dealing with the `vue-storefront` and the `vue-storefront-api` apps as a backend.

This is a general rule of setting up a production node.js app, which gives you a lot of flexibility regarding the SSL, gzip compression, URL routing, and other techniques to be configured without additional hassle. You can use any other proxy server for this purpose, such as Varnish or Apache2 + mod_proxy.

Some additional materials:

- [How to setup production Node.js app in the Digital Ocean environment (Ubuntu 16)](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-16-04)
- [How to setup NGINX reverse proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

#### NGINX configuration

[Here is the complete `/etc/nginx/sites-enabled/prod.vuestorefront.io` file](https://github.com/DivanteLtd/vue-storefront/tree/master/docs/guide/installation/etc/nginx/sites-enabled/prod.vuestorefront.io).

Create NGINX config file from the template (please run as a root user):

```bash
curl https://raw.githubusercontent.com/DivanteLtd/vue-storefront/develop/docs/guide/installation/prod.vuestorefront.io > /etc/nginx/sites-available/prod.vuestorefront.io
ln -s /etc/nginx/sites-available/prod.vuestorefront.io /etc/nginx/sites-enabled/prod.vuestorefront.io
```

**Install the SSL certificate**

```bash
mkdir /etc/nginx/ssl
nano /etc/nginx/ssl/prod.vuestorefront.io.chained.crt
nano /etc/nginx/ssl/prod.vuestorefront.io.key
nano /etc/nginx/ssl/dhparam.pem
```

Now you can run the NGINX:

```bash
/etc/init.d/nginx restart
```

Please find the full comments on the following sections of the file below:

```js
server {
	listen 80;
	server_name prod.vuestorefront.io;
	return 301 https://prod.vuestorefront.io$request_uri;
}
```

This section runs the standard http://prod.vuestorefront.io and creates a wildcard redirect from http://prod.vuestorefront.io/* -> https://prod.vuestorefront.io/. SSL secured connection is a must to run PWA and use Service Workers.

```js
server {
	listen 443 ssl;
	server_name prod.vuestorefront.io http2;

	ssl on;
```

We're using `http2`, but it's not required. This section is for setting up the SSL secured virtual host of the Vue Storefront frontend.

```
ssl_certificate /etc/nginx/ssl/prod.vuestorefront.io.chained.crt;
ssl_certificate_key /etc/nginx/ssl/prod.vuestorefront.io.key;
```

We assume that certificate-related files are stored in the `/etc/nginx/ssl/`. Please point it to your certificate files.

```
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:DHE-RSA-AES256-SHA;
ssl_ecdh_curve secp384r1;
ssl_session_timeout 10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;

ssl_dhparam /etc/nginx/ssl/dhparam.pem;

add_header Strict-Transport-Security "max-age=31536000" always;
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
add_header X-Robots-Tag none;
```

Here we go with the SSL settings based on our best experiences from the past. Please read details in the
 [NGINX documentation](http://nginx.org/en/docs/http/configuring_https_servers.html) if you like ;)

```
gzip on;
gzip_proxied any;
gzip_types
  text/css
  text/javascript
  text/xml
  application/javascript
  application/json
  text/json
  text/html;
}
```

Vue Storefront SSR responses contain the full markup and JSON objects included for speeding up the first page view. Unfortunately, with significant JS bundle sizes, it can generate a significant network load. We're optimizing it by using gzip compression server-side.

```
location / {
  proxy_pass http://localhost:8000/;
}
```

We're using [`proxy_pass`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) from the `ngx_http_proxy_module` to pull content from the Vue Storefront node.js server. The site will be available under https://prod.vuestorefront.io/

```
location /assets/ {
  proxy_pass http://localhost:8000/assets/;
}
```

The same module is used for providing users with static assets. Assets will be available under: https://prod.vuestorefront.io/assets

```
location /api/ {
  proxy_pass http://localhost:8080/api/;
}
```

The next proxy section is used for serving the API. It's a proxy to [`vue-storefront-api`](https://github.com/DivanteLtd/vue-storefront-api) app running on `8080` port (default config). The API will be available under: https://prod.vuestorefront.io/api

```
location /img/ {
  proxy_pass http://localhost:8080/img/;
}
```

The last proxy is used for serving product images. It's a proxy to the [`vue-storefront-api`](https://github.com/DivanteLtd/vue-storefront-api) app running on `8080` port (default config). Images will be available under: https://prod.vuestorefront.io/img

#### Apache2 configuration

In case you are already using the apache2 web server in your environment as well and can't (or don't want) to use NGINX, you can also set up apache2 as a reverse proxy instead of NGINX. This is done by adding this block to your apache2 virtual host.

```
ProxyRequests off

ProxyPass /api/ http://localhost:8080/api/
ProxyPassReverse /api http://localhost:8080/api/

ProxyPass /img/ http://localhost:8080/img/
ProxyPassReverse /img http://localhost:8080/img/

ProxyPass /assets/ http://localhost:8000/assets/
ProxyPassReverse /assets http://localhost:8000/assets/

ProxyPass / http://localhost:8000/
ProxyPassReverse / http://localhost:8000/
```

You also need to enable [mod_proxy](https://httpd.apache.org/docs/current/mod/mod_proxy.html) for this.

### Vue Storefront and Vue Storefront API

After you have the NGINX set up, you should get a `502 error` when accessing the https://prod.vuestorefront.io. This is totally fine! We just missed the most important step, which is running backend services that will power up our installation. Now NGINX is trying to connect to `localhost:8000` for `vue-storefront` and `localhost:8080` for `vue-storefront-api` without any success.

We created a Linux user called `vuestorefront` and go to `/home/www/vuestorefront` which is our home directory.

You need to clone the `vue-storefront` and the `vue-storefront-api` repos accordingly with the following commands:

```bash
su vuestorefront
cd /home/www/vuestorefront
git clone https://github.com/DivanteLtd/vue-storefront.git
git clone https://github.com/DivanteLtd/vue-storefront-api.git
```

Then, you will need to install the required node packages:

```bash
cd /home/www/vuestorefront/vue-storefront-api
yarn install
```

... and ...

```bash
cd /home/www/vuestorefront/vue-storefront
yarn install
```

It may take a few minutes. Once the modules are installed, we can set configuration files for both services.

#### Vue Storefront configuration

The full configuration files are available here to download: [vue-storefront](https://github.com/DivanteLtd/vue-storefront/blob/develop/docs/guide/installation/vue-storefront/config) and [vue-storefront-api](https://github.com/DivanteLtd/vue-storefront/blob/develop/docs/guide/installation/vue-storefront-api/config).

Please create the `vue-storefront-api/config/local.json` and `vue-storefront/config/local.json` files accordingly.

```bash
curl https://raw.githubusercontent.com/DivanteLtd/vue-storefront/develop/docs/guide/installation/vue-storefront-api/config/local.json > /home/www/vuestorefront/vue-storefront-api/config/local.json
```

... and ...

```bash
curl https://raw.githubusercontent.com/DivanteLtd/vue-storefront/develop/docs/guide/installation/vue-storefront/config/local.json > /home/www/vuestorefront/vue-storefront/config/local.json
```

Please find the key sections of the `vue-storefront/config/local.json` file described in below:

```json
"elasticsearch": {
    "httpAuth": "",
    "host": "https://prod.vuestorefront.io/api/catalog",
    "index": "vue_storefront_catalog"
},
"storeViews": {
    "mapStoreUrlsFor": [
        "de",
        "it"
    ],
    "multistore": true,
    "de": {
        "disabled": false,
        "elasticsearch": {
            "httpAuth": "",
            "host": "https://prod.vuestorefront.io/api/catalog",
            "index": "vue_storefront_catalog_de"
        }
    },
    "it": {
        "disabled": false,
        "elasticsearch": {
            "httpAuth": "",
            "host": "https://prod.vuestorefront.io/api/catalog",
            "index": "vue_storefront_catalog_it"
        }
    }
},
```

We're setting up the product's endpoint to https://prod.vuestorefront.io/api/catalog (please use your domain accordingly of course). As you may notice, the `/api` url is proxied by the NGINX to `localhost:8080` - our `vue-storefront-api` instance.

```json
"cart": {
      "synchronize": true,
      "synchronize_totals": true,
      "create_endpoint": "https://prod.vuestorefront.io/api/cart/create?token={{token}}",
      "updateitem_endpoint": "https://prod.vuestorefront.io/api/cart/update?token={{token}}&cartId={{cartId}}",
      "deleteitem_endpoint": "https://prod.vuestorefront.io/api/cart/delete?token={{token}}&cartId={{cartId}}",
      "pull_endpoint": "https://prod.vuestorefront.io/api/cart/pull?token={{token}}&cartId={{cartId}}",
      "totals_endpoint": "https://prod.vuestorefront.io/api/cart/totals?token={{token}}&cartId={{cartId}}",
      "paymentmethods_endpoint": "https://prod.vuestorefront.io/api/cart/payment-methods?token={{token}}&cartId={{cartId}}",
      "shippingmethods_endpoint": "https://prod.vuestorefront.io/api/cart/shipping-methods?token={{token}}&cartId={{cartId}}",
      "shippinginfo_endpoint": "https://prod.vuestorefront.io/api/cart/shipping-information?token={{token}}&cartId={{cartId}}",
      "collecttotals_endpoint": "https://prod.vuestorefront.io/api/cart/collect-totals?token={{token}}&cartId={{cartId}}",
      "deletecoupon_endpoint": "https://prod.vuestorefront.io/api/cart/delete-coupon?token={{token}}&cartId={{cartId}}",
      "applycoupon_endpoint": "https://prod.vuestorefront.io/api/cart/apply-coupon?token={{token}}&cartId={{cartId}}&coupon={{coupon}}"
  },
```

There are 27 more instances of `prod.vuestorefront.io` to be replaced with your production URL address in this file. Please just do so :)

#### Vue Storefront API configuration

The [provided vue-storefront-api configuration](https://github.com/DivanteLtd/vue-storefront/blob/develop/docs/guide/installation/vue-storefront-api/config) requires almost no changes.

The only lines you need to alter are:

```json
"imageable": {
    "namespace": "",
    "maxListeners": 512,
    "imageSizeLimit": 1024,
    "timeouts": {
        "convert": 5000,
        "identify": 100,
        "download": 1000
    },
    "whitelist": {
        "allowedHosts": [
            ".*divante.pl",
            ".*vuestorefront.io"
        ],
        "trustedHosts": [
            ".*divante.pl",
            ".*vuestorefront.io"
        ]
    },
    "keepDownloads": true,
    "maxDownloadCacheSize": 1000,
    "tmpPathRoot": "/tmp"
},
"elasticsearch": {
    "host": "localhost",
    "port": "9200",
    "indices": [
        "vue_storefront_catalog",
        "vue_storefront_catalog_it",
        "vue_storefront_catalog_de"
    ]
}
```

You should put here the `allowedHosts` and `trustedHosts` for the Imageable to download the product images. The domain name points to the Magento 2 instance where images are sourced. In this example, Magento 2 is running under **http://demo-magento2.vuestorefront.io**.

 #### Using your own Magento 2 instance
 
 In this case, you'll have to update `magento2` config node with the correct hostname in the vue-storefront-api config file. To get all necessary Magento 2 API data for the `api` node, navigate to SYSTEM -> Extensions -> Integrations in the Magento 2 Admin.
node, navigate to SYSTEM -> Extensions -> Integrations in the Magento 2 Admin.

- Click Add New Integration
- Check the necessary permissions (check Catalog, Sales, My Account, and Carts on the API Permissions tab )
- Click Activate
- Copy necessary keys, secrets, and tokens into the api section of vue-storefront-api config
 
#### Build VS

Before we can run Vue Storefront and Vue Storefront API, we should build it in production mode. To do so, please execute the following commands:

```bash
cd /home/www/vuestorefront/vue-storefront/
yarn build
```

```bash
cd /home/www/vuestorefront/vue-storefront-api/
yarn build
```

#### Data import

Vue Storefront needs to have some data in the Elasticsearch to properly display products and categories. Of course, you can install [mage2vuestorefront](https://github.com/DivanteLtd/mage2vuestorefront) and configure the data pump to synchronize and update the Elasticsearch index whenever data is being changed in Magento. For the purpose of this tutorial, we'll just restore the data from the JSON file.

You can easily dump your current VS index using the following command (your local installation):

```bash
cd vue-storefront-api
rm var/catalog.json
npm run dump
```

Now in the `var/catalog.json` you have your current database dump. Please transfer this file to the server—for example, using the following ssh command:

```bash
ssh vuestorefront@prod.vuestorefront.io rm ~/vue-storefront-api/var/catalog.json
scp vue-storefront-api/var/catalog.json vuestorefront@prod.vuestorefront.io:~/vue-storefront-api/var/catalog.json
```

Then, after logging in to your `prod.vuestorefront.io` server as a `vuestorefront`, you can run the following command to import the data:

```bash
cd vue-storefront-api
npm run db new
npm run restore2main
npm run db rebuild
```

#### Running the Vue Storefront and Vue Storefront API

After everything set, you can just start the `vue-storefront` and `vue-storefront-api`:

```bash
cd vue-storefront-api
yarn start
cd vue-storefront
yarn start
```

Both applications use [`PM2` process manager](https://pm2.io/runtime) in production mode (`start` commands) to manage and respawn the node.js processes when needed.

## Production setup - using Docker / Kubernetes

To be prepared.
