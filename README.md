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

## Hello Docker

> Make sure that you have Docker installed on your machine. You can follow the official [guide](https://docs.docker.com/install/linux/docker-ce/ubuntu/).

To get in touch with containerization we will start by introducing a possible use case. Suppose that we have an application that interacts with a Postgres DB but we want to skip the hassle of locally installing Postgres. To do so, we can use Docker to create a container with a running database for us.

As we have online repositories such as [Maven](https://mvnrepository.com/) or [Pypi](https://pypi.org/) for both Jars and Python libraries, we also have a repository for Docker Images: [Docker Hub](https://hub.docker.com/).

We can think of an **Image** as a blueprint. By downloading an image to our machine, we then can instantiate any number of **containers** we want to run the contents of the image. Thus, as we want to work with postgres, we will some commands to get it running. We will use an Official Image for [Postgres](https://hub.docker.com/_/postgres):

```bash
# First, we pull the image
docker pull postgres

# Create a container and assign a name
docker run --name some-postgres postgres
```

> OBS: We could have just run `docker run` and if the image was not in the machine, it would have automatically been pulled.

We should be able to see some output from that command above. However, let's make sure that everything is running as expected by [listing](https://docs.docker.com/engine/reference/commandline/ps/) all the containers in our machine:

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

We tried to connect to our localhost (where the container is running) in the specified port, but we can't! This happens because we need to open the ports of the container. Let's remove the current container with `docker rm -f <containerID>` and prepare a more involved `run` command:

```bash
docker run -p 5432:5432 --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```

Which will return the container. What we've done here - apart from setting a pwd as an environment variable - is specifying a link between `LOCAL_PORT:CONTAINER_PORT`. Now we should be able to really connect to that db:

```bash
nc -vz localhost 5432
Connection to localhost 5432 port [tcp/postgresql] succeeded!
```

> OBS: Note how we also used the `-d` option, which runs the container **detached** from the terminal session.

## Building an Application

After getting our db ready, it's time we interact with it. To do so, there is a Python flask application `app.py` that acts as a db client to which we'll run our queries to. But instead of directly running the application, we will build a docker container to host it. The files used to create Docker Images are called `Dockerfile`.

If we examine the one provided we can see the following steps:
1. `FROM` specifies the `Base Image`, i.e. another Docker image used as a foundation for ours.
2. `WORKDIR` sets the path on which the commands will run.
3. `COPY` puts our application into the `WORKDIR`. Note how we will need to build the Docker where this file is accessible and the path matches the one here.
4. `RUN` to trigger some commands.
5. We also set an `ARG` (argument) so that we can set the application port when building the image.
6. `EXPOSE` the application port IN the container.
7. `CMD` to run the application at container launch.

Now that this is clear, let's build the image:

```bash
docker build --build-arg APP_PORT=5000 --tag=pmbrull/python-flask-example .
```

Building an image is like compiling a piece of code, where we make sure that it is able to spin up working containers.

> OBS: `tag` specifies the name we want to set to our image. It usually starts with <userName>/<imageName>.

We can now check that the image is created with `docker image ls` and finally run a container with it:

```bash
docker run -e "PORT=5000" -p 5001:5000 pmbrull/python-flask-example
```

Note how we linked our 5001 port with container's 5000. Moreover, we built the Python app to be flexible enough so that the port used is an environment variable. We can set environment variables with the option `-e`. Building applications following these guidelines allow for better reusability.

## Docker Network

When building applications from scratch we need to make sure that all containers can understand each other, meaning that they all run in the same network. To do so, Docker has some built-in features regarding networks. What we could do is creating a network `docker network create my-net` and then run the container with the `--net my-net` flag.

```bash
docker run --name=some-postgres --net=my-net -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 -d postgres 
docker run -e "PORT=5000" -e "SERVICE_POSTGRES_SERVICE_HOST=some-postgres:5432" -p 5001:5000 --net my-net -d pmbrull/python-flask-example
```

## Testing the Manual Application

To make sure that everything is running as expected we will run the `post_query.sh` script, where the first argument specifies in which machine the flask app is running, and then we pass the port and a query.

```bash
sh scripts/post_query.sh localhost 5001 "create table account (id_user serial PRIMARY KEY, username VARCHAR(50) NOT NULL)"
# Should return OK
sh scripts/post_query.sh localhost 5001 "insert into account (username) values ('pmbrull')"
# Should return OK
sh scripts/post_query.sh localhost 5001 "select * from account"
# Should return a json with the result
```

We have been able to prepare a fully containerized working application. However, we have started with the manual style to understand the basics of Docker and have a quick view of what a containerized application actually represents. Luckily, there are better ways of handling these scenarios. One of the most common is **Docker Compose**.

## Docker Compose

With compose we still need to have the Docker images we want to use. However, it is useful in terms of setting up a multi-container application as it automatically handles aspects such as network or dependencies between containers.

> You can find further information on how to install Docker Compose in the [docs](https://docs.docker.com/compose/install/)

Let's address now the given compose file `docker-compose.yml` and discuss the different elements that we can find there:

1. `services` lists down all the docker containers that we need.
2. Then, for each service, we specify a name. In our case, those are `postgres` and `flask`.
3. `container_name` is useful when reaching out to other services. In our case, we call the postgres db from flask just by specifying the `some-postgres` host.
4. `volumes` are a big deal when working with databases. There we are mapping a local directory with another inside the container. This means that using volumes we can persist data during restarts!
5. `depends_on` let's us create a hyerarchy in the containers. We will not run the flask image until the postgres database has finished deploying.

Now we can just run `docker-compose up` and we will have everything working fine again. Let's inspect the networks:

```bash
docker network ls
```

We see that there is one called `k8s-workshop_default`, so let's run `docker inspect k8s-workshop_default`. There we will see our two containers created from the docker compose.

Moreover, with this method we've been able to set some environment variables by using the `.env` file. We can check what is the final version of the compose file after putting the variables by running `docker-compose config`.

We can now run the same tests as before to check that everything is running as expected.