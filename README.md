# Embracing Kubernetes - Cloud Native Applications

*Getting started with containerization*

## Introduction

Nowadays cloud technologies have become the *State of the Art* in the IT industry. This is a natural change due to on-premise infrastructure ending up being cumbersome to manage and companies needing teams just involved in guaranteeing a certain level of stability of the production environment (rather than the applications).

The adoption of cloud services enables some key factors infrastructure-wise, from which we can highlight the following:
* Flexibility: __easily manage__ different services based on demand.
* Scalability: __easily deliver__ quality solutions to a flowing amount of users.

Now the question is, why don't we also apply this mindset to our applications? Thanks to container services such as Docker we can also grant our application the same benefits:
* **Flexibility**: converting it into smaller and isolated components that we can treat as independent building blocks.
* **Scalability**: as we are capable of adding more computing units in those parts that tend to become bottlenecks.

If, on top of that we add the capabilites of the **DevOps** philosophy, we will be creating an environment where we can continuosly deliver value as we speed up internal processes by closing the gap between Development and Operations.

## Hands on Docker

> First of all, make sure that you have Docker installed on your machine. You can follow the official [guide](https://docs.docker.com/install/linux/docker-ce/ubuntu/).

To get in touch with containerization we will start by introducing a possible use case. Suppose that we have an application that interacts with a Postgres DB. We will need to build some unit tests that also require this DB connection but we want to skip the hassle of locally installing Postgres. To do so, we can use Docker to create a container with a running database for us.

As we have online repositories such as [Maven](https://mvnrepository.com/) or [Pypi](https://pypi.org/) for both Jars and Python libraries, we also have a repository for Docker Images: [Docker Hub](https://hub.docker.com/).

We can think of an **Image** as a blueprint. By downloading an image to our machine, we then can instantiate any number of **containers** we want to run the contents of the image. Thus, as we want to work with postgres, we will some commands to get it running. We will use an Official Image for [Postgres](https://hub.docker.com/_/postgres):

```bash
# First, we pull the image
docker pull postgres

# Create a container and assign a name
docker run --name some-postgres postgres
```

> OBS: We could have just run `docker run` and if the image was not in the machine, it would have automatically been pulled.

We should be able to see some output from that command above. However, let's make sure that everything is running as expected:

```bash
docker ps -a                                                                                                                                                                                                                   
CONTAINER ID   IMAGE       COMMAND                  CREATED         STATUS         PORTS      NAMES
47690abc4882   postgres    "docker-entrypoint.s…"   3 seconds ago   Up 2 seconds   5432/tcp   some-postgres
```

Now this is giving some valuable information, as we can see in which port *of the container* the db is running. If we try to connect to that...

```bash
nc -vz localhost 5432
nc: connect to localhost port 5432 (tcp) failed: Connection refused
```

This happens because we need to open the ports of the container. Let's remove the current container with `docker rm -f <containerID>` and prepare a more involved `run` command:

```bash
docker run -p 5432:5432 --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```

Which will return the container. What we've done here is - apart from setting a pwd - specifying a link between `LOCAL_PORT:CONTAINER_PORT`. Now we should be able to really connect to that db:

```bash
nc -vz localhost 5432
Connection to localhost 5432 port [tcp/postgresql] succeeded!
```
