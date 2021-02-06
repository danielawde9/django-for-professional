
# Django

Now we will create a Django “Hello, World” project that runs locally on our computer
and then move it entirely within Docker so you can see how all the pieces fit together.

    $ cd ~/Desktop
    $ mkdir code && cd code

Then create a hello directory for this example and install Django using Pipenv which
creates both a Pipfile and a Pipfile.lock file. Activate the virtual environment with
the shell command.

    $ mkdir hello && cd hello
    $ pipenv install django==3.0.1
    $ pipenv shell
    (hello) $
    (hello) $ django-admin startproject hello_project .
    (hello) $ python manage.py migrate
    (hello) $ python manage.py runserver

## Pages App 

    (hello) $ python manage.py startapp pages

## Settings.py

    # hello_project/settings.py
    INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'pages.apps.PagesConfig', # new
    

## hello_project/urls.py

    from django.contrib import admin
    from django.urls import path, include # new
    urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('pages.urls')), # new
    ]

## pages/views.py
    from django.http import HttpResponse
    def home_page_view(request):
        return HttpResponse('Hello, World!')

create the urls.py 

    (hello) $ touch pages/urls.py

## pages/urls.py
    from django.urls import path
    from .views import home_page_view
    urlpatterns = [
        path('', home_page_view, name='home')
    ]

The full flow of our Django homepage is as follows:
* when a user goes to the home-page they will first be routed to hello_project/urls.py 
*  then routed to pages/urls.py
* and finally directed to the home_page_view which returns the string “Hello, World!”

# Docker

## What is Docker

Docker is a way to isolate an entire OS via Linux Containers, (type of virtulization) we virtualized from the Linux layer up instead that will provide a lightweight, faster way to duplicate much of the same functionality
For most applications–especially web applications–a virtual machine provides far more resources than are needed and a container is more than sufficient.
An analogy we can use here is that of homes and apartments. Virtual Machines are
like homes: stand-alone buildings with their own infrastructure including plumbing
and heating, as well as a kitchen, bathrooms, bedrooms, and so on. Docker containers
are like apartments: they share common infrastructure like plumbing and heating, but
come in various sizes that match the exact needs of an owner.

This, fundamentally, is what Docker is: a way to implement Linux containers!

## Containers vs. Virtual Environments

In python there's virtual environments, which are a way to isolate Python packages. Thanks to virtual environments, one computer can run multiple projects locally 

Linux containers go a step further and isolate the entire operating system, not just
the Python parts. In other words, we will install Python itself within Docker as well as
install and run a production-level database.

## Docker hello, world

Docker ships with its own “Hello, World” image that is a helpful first step to run. On
the command line type docker run hello-world . This will download an official Docker
image and then run it within a container.

The command docker info lets us inspect Docker. It will contain a lot of output but
focus on the top lines which show we now have 1 container which is stopped and 1
image.

## Images, Containers, and the Docker Host

**Docker Image:** a docker image is a snapshot in time of what a project contains. it is represented by a **Dockerfile** and is literally a list of instructions that must be built.

**Docker Container:** a docker container is a running instance of an image. To continue our apartment analogy, the actual fully-built building

**Docker Host:** a docker host is the underlying OS, its possible to have multiple containers running within a single Docker Host. When we refer to code or processes running within Docker, that means they are running in the Docker host.

let's create a Dockerfile

    $ touch Dockerfile

## Dockerfile

    # Pull base image
    FROM python:3.7
    # Set environment variables
    ENV PYTHONDONTWRITEBYTECODE 1
    ENV PYTHONUNBUFFERED 1
    # Set work directory
    WORKDIR /code
    # Install dependencies
    COPY Pipfile Pipfile.lock /code/
    RUN pip install pipenv && pipenv install --system
    # Copy project
    COPY . /code/

Dockerfiles are read from top-to-bottom when an image is created. The first instruc-
tion must be the FROM command which lets us import a base image to use for our image, this case `python 3.7`

Then we use the `ENV` command to set two environment variables:

- PYTHONUNBUFFERED ensures our console output looks familiar and is not buffered
by Docker, which we don’t want
- PYTHONDONTWRITEBYTECODE means Python will not try to write .pyc files which we
also do not desire

Next we use `WORKDIR` to set a default work directory path within our image called code
which is where we will store our code. If we didn’t do this then each time we wanted
to execute commands within our container we’d have to type in a long path. Instead
Docker will just assume we mean to execute all commands from this directory.

For our dependencies we are using Pipenv so we copy over both the `Pipfile` and
`Pipfile.lock` files into a `/code/` directory in Docker.

Moving along we use the `RUN` command to first install `Pipenv` and then `pipenv install`
to install the software packages listed in our `Pipfile.lock` , currently just Django. It’s
important to add the `--system` flag as well since by default `Pipenv` will look for a
virtual environment in which to install any package, but since we’re within Docker
now, technically there **isn’t any virtual environment**. In a way, the Docker container
is our virtual environment and more. So we must use the `--system` flag to ensure our
packages are available throughout all of Docker for us.

As the final step we copy over the rest of our local code into the `/code/` direc-
tory within Docker. Why do we copy local code over twice, first the `Pipfile` and
`Pipfile.lock` and then the rest? The reason is that images are created based on
instructions top-down so we want things that change often–like our local code–to
be last. That way we only have to regenerate that part of the image when a change
happens, **not reinstall** everything each time there is a change. And since the software
packages contained in our `Pipfile` and `Pipfile.lock` change infrequently, it makes
sense to copy them over and install them earlier.


Our image instructions are now done so let’s build the image using the command

    docker build .

The period, . , indicates the current directory is where to execute the
command.

Moving on we now need to create a `docker-compose.yml` file to control how to run the
container that will be built based upon our Dockerfile image.

    $ touch docker-compose.yml

## docker-compose.yml

    version: '3.7'

    services:
    web:
        build: .
        command: python /code/manage.py runserver 0.0.0.0:8000
        volumes:
        - .:/code
        ports:
        - 8000:8000

on top line we specify most recent verion of **Docker Compose** which is **different** from python 3.7, it's just coincidence

Then we specify which `services` or containers we want to have running within our **Docker Host**, its possible to have multiple servercies or containers running. 

The volumes mount automatically syncs the Docker filesystem with our local computer's filesystem. **this means that we don't have to rebuild the image each time we change a single file**

Lastly we specify the `ports` to expose within Docker which will be 8000 (django by default)

Run Docker container using the command 

    docker-compose up

### Troubleshoot

incase there was error try the following command

    docker-compose build
    docker-compose up


To confirm it actually worked, go back to http://127.0.0.1:8000/ in your web browser.
Refresh the page and the “Hello, World” page should still appear.
Django is now running purely within a Docker container. We are not working within
a virtual environment locally. We did not execute the runserver command. All of our
code now exists and our Django server is running within a self-contained Docker
container. Success!


Stop the container with Control+c (press the “Control” and “c” button at the same
time) and additionally type

    docker-compose down

Docker containers take up a lot of memory so it’s a good idea to stop them in this way when you’re done using them.
Containers are meant to be **stateless** which is why we use volumes to copy our code over locally where it can be saved.

Docker is a self-contained environment that includes everything we need for local
development: web services, databases, and more if we want. The general pattern will
always be the same when using it with Django:

- create a virtual environment locally and install Django
- create a new project
- exit the virtual environment
- write a Dockerfile and then build the initial image
- write a docker-compose.yml file and run the container with docker-compose up
