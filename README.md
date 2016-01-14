Automatic proxified docker setup demo
======================================

[![By Camptocamp](https://img.shields.io/badge/by-camptocamp-fb7047.svg)](http://www.camptocamp.com)


## What is this?

This is a demo using:

* docker
* docker-compose for composition
* consul for service discovery
* registrator to automatically feed consul with new containers
* tutum/hello-world as the demo application


![docker_consul_haproxy](https://camo.githubusercontent.com/0f9ce22895adb7538c0a641d3efcba1ec1ca9b39/687474703a2f2f63646e2d616b2e662e73742d686174656e612e636f6d2f696d616765732f666f746f6c6966652f662f666f6f7374616e2f32303134313032362f32303134313032363032313831342e706e67)



## How to run


### Start the consul/registrator/haproxy stack

First, launch the consul/registrator/haproxy base infrastructure with:

```
$ docker-compose -f consul.yml up -d
```

You will then be able to see:

* the consul dashboard at [http://consul-admin.infra.127.0.0.1.xip.io](http://consul-admin.infra.127.0.0.1.xip.io)
* the haproxy dashboard at [http://proxy-admin.infra.127.0.0.1.xip.io](http://proxy-admin.infra.127.0.0.1.xip.io)


### Start the application stack

Then launch the hello-world application with:

```
$ docker-compose up -d
```

which will let you access:

* the running application at [http://www.hello.127.0.0.1.xip.io](http://www.hello.127.0.0.1.xip.io)

the `web_1` node was also added to:

* the [consul dashboard](http://consul-admin.infra.127.0.0.1.xip.io/ui/#/dc1/services/www) as a service node
* the [haproxy dashboard](http://proxy-admin.infra.127.0.0.1.xip.io/#hello_www_backend) as a backend


## Scale the app

If you type:

```
$ docker-compose scale web=2
```

You will see:

* A new node added to the [consul dashboard](http://consul-admin.infra.127.0.0.1.xip.io)
* A new backend added to the [haproxy dashboard](http://proxy-admin.infra.127.0.0.1.xip.io)
* A changing ID when refreshing the [application page](http://www.hello.127.0.0.1.xip.io)


## Health checking

Health checking is implemented at two levels in this stack:

* [haproxy](http://proxy-admin.infra.127.0.0.1.xip.io) checks that port 80 of each member is up
* [consul](http://consul-admin.infra.127.0.0.1.xip.io) checks that the node is up and that `/` returns a 200 status

If either check fails, the node will not be proxified.


You can test the HTTP check with:

```
$ docker-compose scale web=3
$ docker exec -ti $(basename $PWD)_web_2 pkill php-fpm
```

This will kill the `php-fpm` master process of the second container:

* The `web_2` node will be shown in error in the [consul dashboard](http://consul-admin.infra.127.0.0.1.xip.io/ui/#/dc1/services/www)
* The `web_2` node will not appear anymore in the [haproxy dashboard](http://proxy-admin.infra.127.0.0.1.xip.io/#hello_www_backend)
* The service will still run on the two other instances

To fix it, kill the container and scale again:

```
$ docker rm -f $(basename $PWD)_web_2
$ docker-compose scale web=3
```

## Use service tags

Options of the haproxy are managed using service tags. An example is provided in [hello2.yml](hello2.yml), which provides the same `www` service as `docker-compose.yml`, but using a `world.dev` domain name, as well as `hello.home`, using a `server_aliases=` tag. The container is also named differently (`web2`) to avoid compose thinking it's the same containers.

If you launch:

```
$ docker-compose -f hello2.yml up -d
```

you should see:

* A new `world.dev` tag for the `www` service in the [consul dashboard](http://consul-admin.infra.127.0.0.1.xip.io/ui/#/dc1/services/www)
* A new node in that service
* A new `world.dev_www` backend in the [haproxy dashboard](http://proxy-admin.infra.127.0.0.1.xip.io/#world.dev_www_backend)

You can access the new application at [http://www.world.dev.127.0.0.1.xip.io](http://www.world.dev.127.0.0.1.xip.io) as well as [http://hello.home.127.0.0.1.xip.io](http://hello.home.127.0.0.1.xip.io).


## Notes

### Application internal load-balancing

The current setup using haproxy, consul and registrator makes it easy to load-balance applications by their service names (and optionally server aliases).

However, it does not solve out of the box the problem of internal load-balancing (i.e. when component A of the application needs to connect to component B, in a load-balanced way). One way to achieve that is demonstrated in `hello2.yml`, using `external_links`. In our case, this would be:

```yaml
- "load-balancer:B"
```

This will insert and entry for `B` in `/etc/hosts`, resolving to the IP of the haproxy container. Since haproxy-consul will accept connections with the simple service name `B`, this will effectively load-balance to all `B` containers behind the proxy, without using the full address.

This saves us from using an ambassador or connectable for this purpose, since haproxy can load-balance on dynamic IP/port combinations.


### Multiple versions of the same service

Since there is currently no namespace system in consul, deploying multiple versions of the same service can be tricky. The official recommendation seems to use tags, ACLs and prepared queries. This is complex and quite confusing (since all nodes are listed under the same service name with different labels).

What we are advocating with this setup is to use different service names, which will map to different domain names behind the proxies. One way to do that would be to have an environment variable set up when launching your `docker-compose` stack. For example, if that variable is called `$PROD_LINE`, your composition could be something like:

```yaml
serviceA:
  image: foobar/a
  external_links:
    - "load-balancer:serviceB.${PROD_LINE}"
  environment:
    SERVICE_B: "serviceB.${PROD_LINE}:80"
    SERVICE_NAME: "serviceA.${PROD_LINE}"
    SERVICE_TAGS: "server_alias=serviceA.${PROD_LINE}.example.com"
  ports:
    - 80

serviceB:
  image: foobar/b
  environment:
    SERVICE_NAME: "serviceB.${PROD_LINE}"
  ports:
    - 8080
```

In this setup, provided the composition is started with `PROD_LINE=prod`, `serviceA` will be accessible by users as `serviceA.prod.example.com` (provided the DNS entry is made to point to the proxy). ServiceA will be able to communicate with serviceB by directly using the `serviceB.prod` name. The `$SERVICE_B` environment variable is passed to the serviceA container for that purpose.


In a CI/CD setup, it would be rather simple to use for example the commit ID or the branch name as the `PROD_LINE` value, allowing to point to domains such as `serviceA.2a5bf183.example.com` or `serviceA.feat42.example.com`.



## TODO

- [ ] Test with an overlay network and:

  - [ ] get rid of links
  - [ ] use consul DNS instead of links

- [ ] Test on swarm
- [ ] Port to an AWS ECS stack with CloudFormation
- [ ] Add a demo of internal load-balancing using the haproxy container with the short service name

