---
published: true
title: Override docker-compose port numbers
tags:
  - docker
header:
  image: /images/dockercompose.jpg
---
When you are using a docker-compose yaml file, you basically describe how your containers need to run and interact with each other. Part of this are the ports that a container need to use. For example, take the below docker-compose file:

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

The *rabbitmq* service has 2 ports listed. If you now run `docker-compose up` it will map the requested ports to the services this container offers. However, if you already have some other service running on these ports, docker-compose is unable to provision as it cannot acquire access to the ports.

An obvious solution is to change it in this yaml file, however, it might be checked in into source control and used by others for which this setting makes sense or depend upon. Since you can specify multiple files as parameters there is another solution you might want to try.  

Create a new yaml file for your own settings (for example docker-compose.username.yaml) and only include the changes compared to the base file. Like this:

```yaml
version: '3'

services:

  rabbitmq:
    ports:
      - "15673:15672"
      - "5673:5672"
```

You can try to use `docker-compose -f docker-compose.yml -f docker-compose.username.yml up` but this won't give you the desired effect yet. Some elements of the yaml files cannot be overwritten and are actually concatenated to each other during the merge. Docker calls those [multi-value options](https://docs.docker.com/compose/extends/#adding-and-overriding-configuration) and these consist of ports, expose, external_links, dns, dns_search, and tmpfs.

The solution is to keep your base docker-compose.yml file simple and without the port numbers and add a **docker-compose.override.yml** file. Here you add the default values. The docker-compose.override.yml is automatically merged with the docker-compose.yml file when you call `docker-compose up`.

So your docker-compose.yml file consists of this:

```yaml
version: '3'

services:

  rabbitmq:
    image: "rabbitmq:3-management"
    hostname: "rabbit"
    labels:
      NAME: "rabbitmq"
    volumes:
      - ./rabbitmq-isolated.conf:/etc/rabbitmq/rabbitmq.config
```

And the docker-compose.override.yml adds the default ports:

```yaml
version: '3'

services:

  rabbitmq:
    ports:
      - "15672:15672"
      - "5672:5672"
```

For your own override, you use the first method again; have your settings in its own yaml file and include this with the `docker-compose -f docker-compose.yml -f docker-compose.username.yml up` command. In this case, the override file will be ignored and the port numbers from your file will be used.
