---
published: false
title: Run RabbitMQ using Docker Compose with guest user outside localhost
tags:
  - docker
  - rabbitmq
header:
  image: /images/queue.jpg
---
In a recent project, we needed to run [RabbitMQ](https://www.rabbitmq.com/) as a replacement for our Azure service bus. The Azure SB works fine, but when we wanted to create a local end to end test environment we did not want to have external dependencies.

This E2E environment consists of a docker-compose file in which we have a RabbitMQ section:

```yaml
version: '3'

services:

  rabbitmq:
    image: "rabbitmq:3-management"
    hostname: "rabbit"
    ports:
      - "15672:15672"
      - "5672:5672"
    labels:
      NAME: "rabbitmq"
    volumes:
      - ./rabbitmq-isolated.conf:/etc/rabbitmq/rabbitmq.config

```  

This will launch a RabbitMQ docker container including the management website. Nothing exciting here, however, we had some more applications inside this docker compose file and they referenced this RabbitMQ instance by its host name. This means we cannot connect using the default guest username and password as that is only allowed when connecting from localhost.

So we need to pull in a config file to enable the guest user access from outside localhost. As you see in the above docker-compose file we map the `rabbitmq-isolated.conf` file from the local folder to the config location for the application.

The contents of the `rabbitmq-isolated.conf`:

```
[
 {rabbit,
  [
   %% The default "guest" user is only permitted to access the server
   %% via a loopback interface (e.g. localhost).
   %% {loopback_users, [<<"guest">>]},
   %%
   %% Uncomment the following line if you want to allow access to the
   %% guest user from anywhere on the network.
   {loopback_users, []},
   {default_vhost,       "/"},
   {default_user,        "guest"},
   {default_pass,        "guest"},
   {default_permissions, [".*", ".*", ".*"]}
  ]}
].
```

When you now run `docker-compose up` the RabbitMQ container with the right config will be started and guest access is possible from over the network. 

> NOTE this default setting to only allow guest access from localhost is there for a reason. Be very aware when you allow access from outside localhost. 