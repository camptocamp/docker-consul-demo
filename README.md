Automatic proxified docker setup demo
======================================


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

* the [consul dashboard](http://consul-admin.infra.127.0.0.1.xip.io/ui/#/dc1/services/www.hello) as a service node
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

* The `web_2` node will be shown in error in the [consul dashboard](http://consul-admin.infra.127.0.0.1.xip.io/ui/#/dc1/services/www.hello)
* The `web_2` node will not appear anymore in the [haproxy dashboard](http://proxy-admin.infra.127.0.0.1.xip.io/#hello_www_backend)
* The service will still run on the two other instances

To fix it, kill the container and scale again:

```
$ docker rm -f $(basename $PWD)_web_2
$ docker-compose scale web=3
```


## TODO

- [ ] Test with an overlay network and:

  - [ ] get rid of links
  - [ ] use consul DNS instead of links

- [ ] Test on swarm
- [ ] Port to an AWS ECS stack with CloudFormation

