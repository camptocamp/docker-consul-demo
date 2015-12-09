Automatic proxified docker setup demo
======================================


## What is this?

This is a demo using:

* docker
* docker-compose for composition
* consul for service discovery
* registrator to automatically feed consul with new containers
* tutum/hello-world as the demo application

## How to run

```
$ docker-compose up -d
```

## What is there to see?

Once the stack is started, you can view:

* the consul dashboard at [http://localhost:8500](http://localhost:8500)
* the haproxy dashboard at [http://proxy-admin.127.0.0.1.xip.io](http://proxy-admin.127.0.0.1.xip.)
* the running application at [http://hello-dev.127.0.0.1.xip.io](http://hello-dev.127.0.0.1.xip.io)
