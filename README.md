# rocket.chat-docker-reverse-proxy

This guide is intended to provide a succinct overview of getting [rocket.chat](https://www.rocket.chat/) running behind the behind the [jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy) reverse proxy container paired with the [jrcs/letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion).

## Prerequisites

Before beginning, you should already have an instance of Docker up and running, and should create an A record on your name server for the URL at which you wish to host rocket.chat.

> This guide will go over a basic setup of the `nginx-proxy` and `letsencrypt-nginx-proxy-companion` containers, but if you already have them setup, you can skip those sections.

## 1. Reverse Proxy Setup

To begin, let's get an instance of the `nginx-proxy` image up and running. This image can be configured to listen on ports 80 and 443, and will respond to HTTP and HTTPS requests. As new docker containers are started with the `VIRTUAL_HOST` environment variable, `nginx-proxy` will automatically adjust its internal config files to respond to "name-based" requests.

Use the following command to configure `nginx-proxy` for use with the LetsEncrypt companion:

```bash
docker run -d \
	--name nginx-proxy \
	--publish 80:80 \
	--publish 443:443 \
	--volume nginx-proxy_certs:/etc/nginx/certs \
	--volume nginx-proxy_vhost:/etc/nginx/vhost.d \
	--volume nginx-proxy_default:/usr/share/nginx/html \
	--volume nginx-proxy_dhparam:/etc/nginx/dhparam \
	--volume /var/run/docker.sock:/tmp/docker.sock:ro \
	jwilder/nginx-proxy
```

The container's additional volumes are setup to expose nginx's SSL and vhost directories to the LetsEncrypt companion. Once started, the companion will scan the vhost directory to determine which certificates it needs to generate, and then store the generated certificates in the certs directory.

> The `nginx-proxy_default` volume is setup to allow the operator to publish content to the reverse proxy's home page. For example, if your Docker host's FQDN is `docker01.example.com`, the content in in the `...default` volume will be what's served if a browser is pointed to that address.

## 2. LetsEncrypt Companion Setup

After we've setup our reverse proxy, we need to get an instance of `letsencrypt-nginx-reverse-proxy-companion` running to issue certificates and move them into nginx's store so that they can be served-up. Once configured, the `letsencrypt-nginx-reverse-proxy-companion` executes the task of creating and storing certificates automatically. As you add new containers with the `VIRTUAL_HOST`, `LETSENCRYPT_HOST`, and `LETSENCRYPT_EMAIL` environment variables, the companion will generate certificates and send to to nginx for use.

Use the following command to configure the LetsEncrypt companion (make sure to replace your e-mail address where specified):
```bash
docker run -d \
    --name nginx-proxy-letsencrypt \
    --volumes-from nginx-proxy \
    --volume /var/run/docker.sock:/var/run/docker.sock:ro \
    -e "DEFAULT_EMAIL=<your_email@example.com>" \
    jrcs/letsencrypt-nginx-proxy-companion

```
> Take note that the Docker socket volume is mounted differently than it was in the configuration for the reverse proxy. This is intentional, and necessary.

## 3. rocket.chat Setup
With both the reverse proxy and the companion configured, we're ready to get rocket.chat up and running. As stated before, you should have already created the A record on your nameserver before starting this endeavor. This next step is extremely similar to the instructions published on the [Docker Hub page for rocket.chat](https://hub.docker.com/_/rocket-chat), but includes considerations for using the reverse proxy and the LetsEncrypt companion.

Per the rocket.chat documentation, use the following command to create and configure an instance of the MongoDB (v4.0) image:

```bash
docker run --name rcdb -d mongo:4.0 --smallfiles --replSet rs0 --oplogSize 128
```
> I've named my image "rcdb," but you can choose whatever you want.

Next, run the following to initiate replicaSet on your newly created MongoDB image:
```bash
docker exec -ti rcdb mongo --eval "printjson(rs.initiate())"
```

Finally, use the following command to create a new instance of rocket.chat served-up behind our proxy and with SSL provided by the LetsEncrypt companion:

```bash
docker run -it --name rocketchat \
	--link rcdb:db \
	-e MONGO_OPLOG_URL=mongodb://rcdb:27017/local \
	-e ROOT_URL=https://chat.example.com \
	-e VIRTUAL_HOST=chat.example.com \
	-e VIRTUAL_PORT=3000 \
	-e LETSENCRYPT_HOST=chat.example.com \
	-e LETSENCRYPT_EMAIL=your_email@example.com \
	rocket.chat
```
> Take care to substitute the values of the various environment variables with your own e-mail, host name, and container name for your MongoDB instance.

> Note that we're not explicitly publishing any ports. The rocket.chat image is already configured to use port 3000, and in our `VIRTUAL_PORT` envionment variable, we're telling our reverse proxy to use 3000 when it responds to requests for the virtual host "chat.example.com."

## Conclusion
That's it. After running the last `docker run` command, wait about a minute or two for rocket.chat to initialize the database and for both the reverse proxy and LetsEncrypt companion to do their magic. Once finished, you should be able to setup rocket.chat by visiting the URL you specified.

Good luck!