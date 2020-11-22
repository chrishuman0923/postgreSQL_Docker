# MySQL Server via Docker

## Table of Contents

- [Overview & Purpose](#overview)
- [Pre-Requisites](#pre-reqs)
- [Installation](#install)
- [Start the Container](#start)
- [Login to the Container](#login-container)
- [Login to MySQL Server](#login-server)
- [Set User Permissions](#user-permissions)
- [Database Security](#security)
- [Connect MySQL Workbench](#workbench)
- [Persistent Data](#persistent)

<a name="overview" />

## Overview & Purpose

This repo provides a step-by-step guide for getting MySQL Server up and running on your development machine using Docker and Docker Compose.

You may be asking yourself:

> "Why would I want MySQL Server running in a container rather than directly on my machine?"

1. There is less clustter on your device.
1. You can spin up multiple MySQL Servers using slightly different verisons to cater to different projects or to experiment before making the leap to a new version.
1. You can reliably create identical / consistent MySQL Servers independent of your device's OS...as long as the OS supports Docker 🙂
1. Your development environment is "documented", automated, and reproducible.

<a name="pre-reqs"></a>

## Pre-Requisites

1. Have Docker and Docker Compose installed on your device.
1. Have access to the `docker` and `docker-compose` CLI commands.
   - You can test this by running `docker -v` & `docker-compose -v`
     - If either of these commands do no return a version or build number, do **NOT** continue until you have them working.
1. Run `docker ps` to confirm Docker engine is currently running.

<a name="install"></a>

## Installation

1. Clone this repo

   ```
   git clone https://github.com/chrishuman0923/mysql_server_docker.git
   ```

1. Create an .env file at the root of this directory for sensitive values using the following template:

   ```
   # The password for the root user
   DB_ROOT_PASS=

   # The username for a new user
   DB_USER=

   # The password for the new user defined above.
   DB_PASS=
   ```

1. Confirm that the MySQL Server version you want to use is defined in docker-compose.yml
   - If you are unsure which version of MySQL Server to use, consult the MySQL Docker page [here](https://hub.docker.com/_/mysql) or just use `mysql:latest` to get the latest version available.

<a name="start"></a>

## Start the Container

1. Open a terminal inside this directory and run the following command.

   ```
   docker-compose up -d
   ```

   - This creates and runs the container as defined by the docker-compose.yml file

   - The `-d` flag simply means to run the container in the background (aka "detached mode").

   - "Detached mode" allows you to regain use of your terminal after the container is started.

If you want to see the logs produced by the running container while in detached mode, you need to login to the running container with the terminal command `docker -it {CONTAINER_NAME} sh` and review the log files produced.

However, two easier ways to review the logs of a container are to print the live logs to the terminal with `docker logs {CONTAINER_NAME} --follow` or view them in the Docker Dashboard GUI.

<a name="login-container"></a>

## Login to the Container

Once the container is up and running, you should be able to view it using `docker ps` and login to it using `docker -it {CONTAINER_NAME} sh`.

- In this case, the container name is defined in the docker-compose.yml on line #6.

<a name="login-server"></a>

## Login to MySQL Server

1. Once logged into the container, you will need to then login to the MySQL Server using `mysql -u {USERNAME} -p`.

1. You will be prompted to enter the password for the user.

<a name='user-permissions'></a>

## Set User Permissions

At this point, the user you defined in the .env file has no permissions other than being able to connect to the server. It cannot inherently perform any CRUD (Create, Read, Update, Delete) operations within the server.

To correct this, we need to give the user permissions.

Login to the MySQL Server as the root user. Once logged in, you can grant any level of permissions you desire to your user that was defined in the .env file.

- If this server is soley for development and will never be in production (i.e. "public facing") then you can run the following:

  ```
  GRANT ALL ON *.* TO '{USERNAME}'@'%';
  ```

  Lets break that command down to understand what we are doing.

  - `GRANT` is the keyword used in MySQL to give permissions.
  - `ALL` are the permissions we are granting to the user.
    - You can get a complete list of permission options from the MySQL documentation.
  - `*.*` refers to the database name then a .(dot) followed by the table name.
    - In this case, we are using wildcards to grant the same permissions to all databases and all tables within each database.
  - `'{USERNAME}'@'%'` specifies the user and the connection IP address.
    - The `'%'` wildcard means we are permitting that user to connect to the server from any IP address.

  So, in effect, this statement will grant our user "super-admin" permissions to perform any action on the server when connecting from any IP address.

After running the above command, you should be able to connect to the server and perform CRUD operations with your created user.

<a name="security"></a>

## Database Security

**WARNING:** The permissions described above are **NOT** recommended for production environments!

Each user should be granted minimum permissions to minimum databases.

Ideally, you would implement user roles and, instead of granting permissions directly to a user, you would add users to a user role that contains the minimum permissions they require and audit the members of each role often.

Also, you would limit the IP addresses from which a user can connect from to a list of approved IP's.

This is by no means the only preventative measures you can take but it is at least a step in the right direction. You should always secure your data servers thoroughly and scrutinize access.

<a name="workbench"></a>

## Connect MySQL Workbench

Now, you may be thinking:

> That is all well and good that I can access my database via the terminal but I am a developer that still likes a GUI. How do I connect my other tools to the database now that it is inside a container? Am I out of luck?

**Of course not!!**

If you look back at our docker-compose.yml you will notice the `ports` definition on line #18. This is our ticket to connecting outside services to our database.

What line #18 and #19 is doing is mapping a port on our host machine (the device running docker) to a port within our container.

`3306:3306` is read as:

> Map all traffic from `localhost:3306` to `container:3306`.

This means that we can set tools like MySQL Workbench to connect to our database on `localhost:3306` and it will be re-routed into our container and reach our database.

Now, `3306` is the default listening port for MySQL Server and I made the ports match for simplicity.

However, lets say that `3306` on my machine was already occupied by something. I could have changed the host port to anything and still connect to my database.

E.G. If I had set the port definition in docker-compose.yml to `8080:3306` then I could still connect to my database by accessing `localhost:8080` and it would still be mapped to `container:3306`.

<a name="persistent"></a>

## Persistent Data

Now, a typical problem with containerizing applications is the non-persistent storage. This means that when the contain stops, the data is gone. Now, for a database, that is un-acceptable! We need our data to be retained regardless how many times we stop and start our container. How do we fix that?

The answer is a persistent volume.

In essence, we need to map the directory(ies) in our container that hold our data to a location on our host machine.

In our docker-compose.yml (line #25), you can see how we mapped or persisent volume. This reads the same as our port mapping.

> Map all the information to the `./db/data` directory on our host machine from the `/var/lib/mysql` directory in the container.

In MySQL Server, the `/var/lib/mysql` directory holds all of the data from the server.

This means that as long as we retain the `./db/data` directory, we can stop, start, and delete our container without fear of losing any information.
