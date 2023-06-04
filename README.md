Django, Docker, and PostgreSQL Tutorial
=======================================

In this tutorial we will create a new Django project using Docker and PostgreSQL. Django ships with built-in SQLite support but even for local development you are better off using a "real" database like PostgreSQL that matches what is in production.

It's *possible* to run PostgreSQL locally using a tool like [Postgres.app](https://postgresapp.com/), however the preferred choice among many developers today is to use [Docker](https://www.docker.com/), a tool for creating isolated operating systems. The easiest way to think of it is as a large virtual environment that contains everything needed for our Django project: dependencies, database, caching services, and any other tools needed.

A big reason to use Docker is that it completely removes any issues around local development set up. Instead of worrying about which software packages are installed or running a local database alongside a project, you simply run a Docker image of the entire project. Best of all, this can be shared in groups and makes team development much simpler.

Install Docker
--------------

The first step is to install the desktop Docker app for your local machine:

- [Docker for Mac](https://docs.docker.com/docker-for-mac/install/)
- [Docker for Windows](https://docs.docker.com/docker-for-windows/install/)
- [Docker for Linux](https://docs.docker.com/install/)

The initial download of Docker might take some time to download. It is a big file. Feel free to stretch your legs at this point!

Once Docker is done installing we can confirm the correct version is running. In your terminal run the command `docker --version`.

$ docker --version
Docker version 20.10.14, build a224086

[Docker Compose](https://docs.docker.com/compose/) is an additional tool that is automatically included with Mac and Windows downloads of Docker. However if you are on Linux, you will need to add it manually. You can do this by running the command `sudo pip install docker-compose` after your Docker installation is complete.

Hopefully Docker is done installing by this point. To confirm the installation was successful quit the local server with `Control+c` and then type `docker run hello-world` on the command line. You should see a response like this:

$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
7050e35b49f5: Pull complete
Digest: sha256:10d7d58d5ebd2a652f4d93fdd86da8f265f5318c6a73cc5b6a9798ff6d2b2e67
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:

 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 <https://hub.docker.com/>

For more examples and ideas, visit:
 <https://docs.docker.com/get-started/>

Docker is properly installed. We can proceed to configuring a local Django set up and then switch over to Docker and PostgreSQL.

Django Set Up
-------------

The code for this project can live anywhere on your computer but the `Desktop` is an easy location for teaching purposes. On the command line navigate to the desktop and create a new directory called `django-docker`.

# Windows

$ cd onedrive\desktop
$ mkdir django-docker

# macOS

$ cd ~/desktop/code
$ mkdir django-docker

We will follow the standard steps for creating a new Django project: make a dedicated virtual environment, activate it, and install Django.

# Windows

$ python -m venv .venv
$ Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
$ .venv\Scripts\Activate.ps1
(.venv) $ python -m pip install django~=4.0.0

# macOS

$ python3 -m venv .venv
$ source .venv/bin/activate
(.venv) $ python3 -m pip install django~=4.0.0

Next we can create a new project called `django_project`, `migrate` our database to initialize it, and use `runserver` to start the local server. Normally I don't recommend running `migrate` on new projects until *after* a custom user model has been configured but in this tutorial we will ignore that advice.

(.venv) $ django-admin startproject django_project .
(.venv) $ python manage.py migrate
(.venv) $ python manage.py runserver

Confirm everything worked by navigating to `http://127.0.0.1:8000/` in your web browser. You may need to refresh the page but should see the familiar Django welcome page.

![Django welcome page](https://learndjango.com/static/images/django40_welcome.png)

The last step before switching over to Docker is creating a `requirements.txt` file with the contents of our current virtual environment. We can do this with a one-line command.

(.venv) $ pip freeze > requirements.txt

In your text editor inspect the newly-created `requirements.txt` file.

asgiref==3.5.0
Django==4.0.4
sqlparse==0.4.2

It should contain `Django` as well as the packages `asgiref` and `sqlparse` which are automatically included when Django is installed.

Now it's time to switch to Docker. Exit our virtual environment since we no longer need it by typing `deactivate` and `Return`.

(.venv) $ deactivate
$

How do we know the virtual environment is no longer active? There will no longer be parentheses around the directory name on the command line prompt. Any normal Django commands you try to run at this point will fail. For example, try `python manage.py runserver` to see what happens.

$ python manage.py runserver
File "/Users/wsv/Desktop/django-docker/manage.py", line 11, in main
  from django.core.management import execute_from_command_line
ModuleNotFoundError: No module named 'django'

This means we're fully out of the virtual environment and ready for Docker.

Docker Image
------------

A Docker *image* is a read-only template that describes how to create a Docker *container*. The image is the instructions while the container is the actual running instance of an image. To continue our apartment analogy from earlier in the chapter, an image is the blueprint or set of plans for building an apartment; the container is the actual, fully-built building.

Images are often based on another image with some additional customization. For example, there is a long list of officially supported images for [Python](https://hub.docker.com/_/python/) depending on the version and flavor of Python desired.

Dockerfile
----------

For our Django project we need to create a custom image that contains Python but also installs our code and has additional configuration details. To build our own image we create a special file known as a *Dockerfile* that defines the steps to create and run the custom image.

Use your text editor to create a new `Dockerfile` file in the project-level directory next to the `manage.py` file. Within it add the following code which we'll walk through line-by-line below.

# Pull base image

FROM python:3.10.2-slim-bullseye

# Set environment variables

ENV PIP_DISABLE_PIP_VERSION_CHECK 1
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set work directory

WORKDIR /code

# Install dependencies

COPY ./requirements.txt .
RUN pip install -r requirements.txt

# Copy project

COPY . .

`Dockerfile`s are read from top-to-bottom when an image is created. The first instruction is a `FROM` command that tells Docker what base image we would like to use for our application. Docker images can be inherited from other images so instead of creating our own base image, we'll use the official Python image that already has all the tools and packages that we need for our Django application. In this case we're using Python `3.10.2` and the much smaller in size `slim` variant that does not contain the common packages contained in the default tag. The tag `bullseye` refers to the latest stable Debian release. It is a good idea to set this explicitly to minimize potential breakage when there are new releases of Debian.

Then we use the `ENV` command to set three environment variables:

- `PIP_DISABLE_PIP_VERSION_CHECK` disables an automatic check for `pip` updates each time
- `PYTHONDONTWRITEBYTECODE` means Python will not try to write `.pyc` files
- `PYTHONUNBUFFERED` ensures our console output is not buffered by Docker

The command `WORKDIR` is used to set a default working directory when running the rest of our commands. This tells Docker to use this path as the default location for all subsequent commands. As a result, we can use relative paths based on the working directory rather than typing out the full file path each time. In our case the working directory is `/code` but it can often be much longer and something like `/app/src`, `/usr/src/app`, or similar variations depending upon the specific needs of a project.

The next step is to install our dependencies with `pip` and the `requirements.txt` file we already created. The `COPY` command takes two parameters: the first parameter tells Docker what file(s) to copy into the image and the second parameter tells Docker where you want the file(s) to be copied to. In this case we are copying the existing `requirements.txt` file from our local computer into the current working directory which is represented by `.`.

Once the `requirements.txt` file is inside the image we can use our last command, `RUN`, to execute `pip install`. This works exactly the same as if we were running `pip install` locally on our machine, but this time the modules are installed into the image. The `-r` flag tells `pip` to open a file--called `requirements.txt` here--and install its contents. If we did not include the `-r` flag `pip` would try and fail to install `requirements.txt` since it isn't itself an actual Python package.

At the moment we have a new image based on the `slim-bullseye` variant of `Python 3.10.2` and have installed our dependencies. The final step is to copy all the files in our current directory into the working directory on the image. We can do this by using the `COPY` command. Remember it takes two parameters so we'll copy the current directory on our local filesystem (`.`) into the working directory (`.`) of the image.

If you're confused right now don't worry. Docker is a lot to absorb but the good news is that the steps involved to "Dockerize" an existing project are very similar.

.dockerignore
-------------

A `.dockerignore` file is a best practice way to specify certain files and directories that should not be included in a Docker image. This can help reduce overall image size and improves security by keeping things that are meant to be secret out of Docker.

At the moment we can safely ignore the local virtual environment (`.venv`), a future `.git` directory, and a future `.gitignore` file. In your text editor create a new file called `.dockerignore` in the base directory next to the existing `manage.py` file.

.venv
.git
.gitignore

We now have our complete instructions for creating a custom image but we haven't actually built it yet. The command to do this is unsurprisingly `docker build` followed by the period, `.`, indicating the `Dockerfile` is located in the current directory. There will be a lot of output here. I've only included the first two lines and the last one.

$  docker  build  .
[+]  Building  9.1s  (10/10)  FINISHED
  =>  [internal]  load  build  definition  from  Dockerfile
...
=>  =>  writing  image  sha256:89ede1...

docker-compose.yml
------------------

Our fully-built custom image is now available to run as a container. In order to run the container we need a list of instructions in a file called `docker-compose.yml`. With your text editor create a `docker-compose.yml` file in the project-level directory next to the `Dockerfile`. It will contain the following code.

version: "3.9"
services:
 web:
 build: .
 ports:

- "8000:8000"
 command: python manage.py runserver 0.0.0.0:8000
 volumes:
- .:/code

On the top line we set the [most recent version](https://docs.docker.com/compose/compose-file/compose-versioning/) of Docker Compose which is currently `3.9`. Then we specify which `services` (or containers) we want to have running within our Docker host. It's possible to have multiple `services` running, but for now we just have one for `web`.

Within `web` we set `build` to look in the current directory for our `Dockerfile`. We'll use the Django default ports of `8000` and execute the `command` to run the local web server. Finally the [volumes](https://docs.docker.com/storage/volumes/) mount automatically syncs the Docker filesystem with our local computer's filesystem. This if we make a change to the code within Docker it will automatically be synced with the local filesystem.

The final step is to run our Docker container using the command `docker-compose up`. This command will result in another long stream of output code on the command line.

$  docker-compose  up
[+]  Building  4.2s  (10/10)  FINISHED
  =>  [internal]  load  build  definition  from  Dockerfile
Step  1/7  :  FROM  python:3.10
...
Attaching  to  docker-web-1
docker-web-1  |  Watching  for  file  changes  with  StatReloader
docker-web-1  |  Performing  system  checks...
docker-web-1  |
docker-web-1  |  System  check  identified  no  issues  (0  silenced).
docker-web-1  |  March  22,  2022  -  21:51:04
docker-web-1  |  Django  version  4.0.4,  using  settings  'django_project.settings'
docker-web-1  |  Starting  development  server  at  <http://0.0.0.0:8000/>
docker-web-1  |  Quit  the  server  with  CONTROL-C.

To confirm it actually worked, go back to `http://127.0.0.1:8000/` in your web browser. Refresh the page and the "Hello, World" page should still appear.

Django is now running purely within a Docker container. We are not working within a virtual environment locally. We did not execute the `runserver` command. All of our code now exists and our Django server is running within a self-contained Docker container. Success!

We will create multiple Docker images and containers over the course of this book and with practice the flow will start to make more sense.:

- create a `Dockerfile` with custom image instructions
- add a `.dockerignore` file
- build the image
- create a `docker-compose.yml` file
- spin up the container(s)

Stop the currently running container with `Control+c` (press the "Control" and "c" button at the same time) and additionally type `docker-compose down`. Docker containers take up a lot of memory so it's a good idea to stop them when you're done using them. Containers are meant to be stateless which is why we use `volumes` to copy our code over locally where it can be saved.

$ docker-compose down
[+] Running 2/2
 ⠿ Container docker-web-1  Removed
 ⠿ Network docker_default
 $

Whenever any new technology is introduced there are potential security concerns. In Docker's case, one example is that it sets the default user to `root`. The root user (also known as the "superuser" or "admin") is a special user account used in Linux for system administration. It is the most privileged user on a Linux system and has access to all commands and files.

The Docker docs contain a section a large section on Security and specifically on [rootless mode](https://docs.docker.com/engine/security/rootless/) to avoid this. We will not be covering it here since this is a book on Django, not Docker, but especially if your website stores sensitive information do review the entire Security section closely before going live.

psycopg2
--------

It's important to pause right now and think about what it means to install a package into Docker as opposed to a local virtual environment. In a traditional project we'd run the command `python -m pip install psycopg2-binary==2.9.3` from the command line to install Pyscopg2. But we're working with Docker now.

There are two options. The first is to install `psycopg2-binary` locally and then `pip freeze` our virtual environment to update `requirements.txt`. If we were going to use the local environment this might make sense. But since we are committed to Docker we can skip that step and instead just update `requirements.txt` with the `psycopg2-binary` package. We don't need to update the actual virtual environment further because it is unlikely we'll be using it. And if we ever did we can just update it based on `requirements.txt` anyway.

In your text editor open the existing `requirements.txt` file and add `psycopg2-binary==2.9.3` to the bottom.

asgiref==3.5.0
Django==4.0.4
sqlparse==0.4.2
psycopg2-binary==2.9.3

At the end of our PostgreSQL configuration changes we will build the new image and spin up our containers. But not yet.

PostgreSQL
----------

In the existing `docker-compose.yml` file add a new service called `db`. This means there will be two separate containers running within our Docker host: `web` for the Django local server and `db` for our PostgreSQL database.

The `web` service depends on the `db` service to run so we'll add a line called `depends_on` to `web` signifying this.

Within the `db` service we specify which version of PostgreSQL to use. As of this writing, Heroku supports version `13` as the latest release so that is what we will use. Docker containers are ephemeral meaning when the container stops running all information is lost. This would obviously be a problem for our database! The solution is to create a `volumes` mount called `postgres_data` and then bind it to a dedicated directory within the container at the location `/var/lib/postgresql/data/`. The final step is to add a [trust authentication](https://www.postgresql.org/docs/current/auth-trust.html) to the `environment` for the `db`. For large databases with many database users it is recommended to be more explicit with permissions, but this setting is a good choice when there is just one developer.

Here is what the updated file looks like:

```yaml
version: "3.9"

services:
  web:
    build: .
    image: '<yourregistryname>/django-docker:3.11-python3.18'
    command: python /code/manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/code
    ports:
      - 8000:8000
    depends_on:
      - db
  db:
    image: postgres:13
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - "POSTGRES_HOST_AUTH_METHOD=trust"

volumes:
  postgres_data:
```

DATABASES
---------

The third and final step is to update the `django_project/settings.py` file to use PostgreSQL and not SQLite. Within your text editor scroll down to the `DATABASES` config.

By default Django specifies `sqlite3` as the database engine, gives it the name `db.sqlite3`, and places it at `BASE_DIR` which means in our project-level directory.

# django_project/settings.py

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
```

To switch over to PostgreSQL we will update the [ENGINE](https://docs.djangoproject.com/en/4.0/ref/settings/#std:setting-DATABASE-ENGINE) configuration. PostgreSQL requires a `NAME`, `USER`, `PASSWORD`, `HOST`, and `PORT`. For convenience we'll set the first three to `postgres`, the `HOST` to `db` which is the name of our service set in `docker-compose.yml`, and the `PORT` to `5432` which is the default PostgreSQL [port](https://en.wikipedia.org/wiki/Port_%28computer_networking%29).

# django_project/settings.py

DATABASES = {
 "default": {
 "ENGINE": "django.db.backends.postgresql",
 "NAME": "postgres",
 "USER": "postgres",
 "PASSWORD": "postgres",
 "HOST": "db",  # set in docker-compose.yml
 "PORT": 5432,  # default postgres port
 }
}

And that's it! We can build our new image containing `psycopg2-binary` and spin up the two containers in detached mode with the following single command:

$ docker-compose up -d --build

If you refresh the Django welcome page at `http://127.0.0.1:8000/` it should work which means Django has successfully connected to PostgreSQL via Docker.

Running commands within Docker is a little different than in a traditional Django project. For example, to `migrate` the new PostgreSQL database running in Docker execute the following command:

$ docker-compose exec web python manage.py migrate

If you wanted to run `createsuperuser` you'd also prefix it with `docker-compose exec web...` so:

$ docker-compose exec web python manage.py createsuperuser

And so on. When you're done, don't forget to close down your Docker container since it can consume a lot of computer memory.

$ docker-compose down

Quick Review
------------

Here is a short version of the terms and concepts we've covered in this post:

- Image: the "definition" of your project
- Container: what your project actually runs in (an instance of the image)
- Dockerfile: defines what your image looks like
- docker-compose.yml: a [YAML](http://yaml.org/) file that takes the Dockerfile and adds additional instructions for how our Docker container should behave in production

We use the `Dockerfile` to tell Docker how to build our *image*. Then we run our actual project within a *container*. The `docker-compose.yml` file provides additional information for how our Docker container should behave in production.

Next Steps
----------

If you'd like to learn more about using Django, Docker, and PostgreSQL I've written an entire book on the subject, [Django for Professionals](https://djangoforprofessionals.com/). The first several chapters are free to read online.

Reference
---------

- [Django, Docker, and PostgreSQL Tutorial](https://learndjango.com/tutorials/django-docker-and-postgresql-tutorial)
