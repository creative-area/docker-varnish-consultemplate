# Varnish 4 container (with Consul Template)

This image is intended to provide a caching and load balancing layer (with [Varnish Cache](https://www.varnish-cache.org/)) in front of web containers.

It should be used along with a [Consul](https://www.consul.io/) cluster and [Registrator](https://github.com/gliderlabs/registrator) in order to provide automatic container registration. With the help of [Consul Template](https://github.com/hashicorp/consul-template), it provide a way to do transparent load balancing of containers.

This image make use of [Supervisor](http://supervisord.org/) to manage multiple processes in our container. Using Supervisor allows us to run [Varnish](https://www.varnish-cache.org/docs/trunk/reference/varnishd.html), [Consul Template](https://github.com/hashicorp/consul-template), and [Varnishncsa](https://www.varnish-cache.org/docs/trunk/reference/varnishncsa.html).

##Usage

You have to run at least one **Consul** node and a container for **Registrator**:

```bash
$ docker run -d \
--hostname consul \
--name consul \
--publish 8400:8400 \
--publish 8500:8500 \
--publish 8600:53/udp \
progrium/consul -server -advertise $DOCKER_IP -bootstrap

$ docker run -d \
--hostname registrator \
--name registrator \
--volume /var/run/docker.sock:/tmp/docker.sock \
gliderlabs/registrator consul://$DOCKER_IP:8500
```

Then you can run as many load balanced instances of your web application as you want. For example, you can launch 2 `hello world` servers:

```bash
$ docker run -d -P --env SERVICE_NAME=varnished-app google/nodejs-hello
$ docker run -d -P --env SERVICE_NAME=varnished-app google/nodejs-hello
```

As you can see above, we set a `SERVICE_NAME` variable with a value of `varnished-app`. That is required for our Varnish load balancer to know which backend apps it has to register.

You can verify that your containers' instances are well registered in Consul by querying Consul API:

```bash
$ curl $DOCKER_IP:8500/v1/catalog/service/varnished-app

[
   {
      "Address" : "192.168.59.103",
      "ServiceName" : "varnished-app",
      "ServiceAddress" : "",
      "Node" : "consul",
      "ServicePort" : 32770,
      "ServiceID" : "registrator:silly_hoover:8080",
      "ServiceTags" : null
   },
   {
      "ServicePort" : 32771,
      "ServiceTags" : null,
      "ServiceID" : "registrator:compassionate_galileo:8080",
      "Node" : "consul",
      "ServiceAddress" : "",
      "ServiceName" : "varnished-app",
      "Address" : "192.168.59.103"
   }
]
```

Then run this container as follow:

```bash
$ docker run -d -P \
--link consul:consul \
creativearea/varnish-consul-template
```

You can adjust one of the following variables:

- `CONSUL_URL` consul:8500
- `VARNISH_PORT` 80
- `VARNISH_STORAGE_BACKEND` malloc,100M
- `VARNISHNCSA_LOGFORMAT` %h %l %u %t "%r" %s %b "%{Referer}i" "%{User-agent}i"
