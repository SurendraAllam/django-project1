# Dockerizing Django with Postgres, Gunicorn, and Nginx

This project is used to configure Django to run on Docker with Postgres. For production environments, we'll add on Nginx and Gunicorn. We'll also take a look at how to serve Django static and media files via Nginx. Next we'all move this project to cloud (ec2 server).

Dependencies:
    1. Django v3.2.6
    2. Docker v20.10.8
    3. Python v3.9.6
    4. gunicorn v20.1.0
    5. Psycopg2-binary v2.9.1

## Project Setup

Create a new project directory along with a new Django project:

```bash
$ mkdir django-on-docker && cd django-on-docker
$ mkdir app && cd app
$ python3.9 -m venv env
$ env\Scripts\activate
(env)$
 
(env)$ pip install django==3.2.6
(env)$ django-admin.py startproject hello_django .
(env)$ python manage.py migrate
(env)$ python manage.py runserver
```
Navigate to http://localhost:8000/ to view the Django welcome screen. Kill the server once done. Then, exit from and remove the virtual environment. We now have a simple Django project to work with.

Create a requirements.txt file in the "app" directory and add Django as a dependency:
```bash

Django==3.2.6

```
Since we'll be moving to Postgres, go ahead and remove the db.sqlite3 file from the "app" directory.
Your project directory should look like:

```yml
    └── app
    ├── hello_django
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── requirements.txt
```
## Docker
Install Docker, if you don't already have it, then add a Dockerfile to the "app" directory:
```yml
# pull official base image
FROM python:3.9.6-alpine
 
# set work directory
WORKDIR /usr/src/app
 
# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1
 
# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt
 
# copy project
COPY . .
```
So, we started with an Alpine-based Docker image for Python 3.9.6. We then set a working directory along with two environment variables:
1. `PYTHONDONTWRITEBYTECODE`: Prevents Python from writing pyc files to disc (equivalent to `python -B` option)
2. `PYTHONUNBUFFERED`: Prevents Python from buffering stdout and stderr (equivalent to `python -u` option)

Finally, we updated Pip, copied over the requirements.txt file, installed the dependencies, and copied over the Django project itself.

Next, add a docker-compose.yml file to the project root:
```yml
version: '3.8'
 
services:
  web:
    build: ./app
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./app/:/usr/src/app/
    ports:
      - 8000:8000
    env_file:
      - ./.env.dev
```
Update the `SECRET_KEY`, `DEBUG`, and `ALLOWED_HOSTS` variables in settings.py:

```python
SECRET_KEY = os.environ.get("SECRET_KEY")
 
DEBUG = int(os.environ.get("DEBUG", default=0))
 
# 'DJANGO_ALLOWED_HOSTS' should be a single string of hosts with a space between each.
# For example: 'DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]'
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS").split(" ")

```
Make sure to add the import to the top:
```python
import os
```
Then, create a `.env.dev` file in the project root to store environment variables for development:
```yml
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
```

Build the image:
```bash
$ docker-compose build
```

Once the image is built, run the container:
```bash
$ docker-compose up -d
```
Navigate to http://localhost:5000/ to again view the welcome screen.

## Postgres
To configure Postgres, we'll need to add a new service to the docker-compose.yml file, update the Django settings, and install `Psycopg2`.
First, add a new service called `db` to docker-compose.yml:

```yml
version: '3.8'

services:
  web:
    build: ./app
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./app/:/usr/src/app/
    ports:
      - 8000:8000
    env_file:
      - ./.env.dev
    depends_on:
      - db
  db:
    image: postgres:13.0-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_USER=hello_django
      - POSTGRES_PASSWORD=hello_django
      - POSTGRES_DB=hello_django_dev

volumes:
  postgres_data:
```

To persist the data beyond the life of the container we configured a volume. This config will bind `postgres_data` to the "/var/lib/postgresql/data/" directory in the container.

We also added an environment key to define a name for the default database and set a username and password.

We'll need some new environment variables for the web service as well, so update .env.dev like so:
```bash
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
```
Update the `DATABASES` dict in settings.py:

```python
DATABASES = {
    "default": {
        "ENGINE": os.environ.get("SQL_ENGINE", "django.db.backends.sqlite3"),
        "NAME": os.environ.get("SQL_DATABASE", BASE_DIR / "db.sqlite3"),
        "USER": os.environ.get("SQL_USER", "user"),
        "PASSWORD": os.environ.get("SQL_PASSWORD", "password"),
        "HOST": os.environ.get("SQL_HOST", "localhost"),
        "PORT": os.environ.get("SQL_PORT", "5432"),
    }
}
```

Here, the database is configured based on the environment variables that we just defined. Take note of the default values.

Update the Dockerfile to install the appropriate packages required for Psycopg2:

```yml
# pull official base image
FROM python:3.9.6-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy project
COPY . .
```

Add Psycopg2 to requirements.txt:

```bash
Django==3.2.6
psycopg2-binary==2.9.1
```

Build the new image and spin up the two containers:

```bash
$ docker-compose up -d --build
```
Run the migrations:
```bash
$ docker-compose exec web python manage.py migrate --noinput
```
Next, add an entrypoint.sh file to the "app" directory to verify that Postgres is healthy before applying the migrations and running the Django development server:
```bash
#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

python manage.py flush --no-input
python manage.py migrate

exec "$@"
```

Update the file permissions locally:
```bash
$ chmod +x app/entrypoint.sh
```
Then, update the Dockerfile to copy over the entrypoint.sh file and run it as the Docker entrypoint command:

```yml
# pull official base image
FROM python:3.9.6-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy entrypoint.sh
COPY ./entrypoint.sh .
RUN sed -i 's/\r$//g' /usr/src/app/entrypoint.sh
RUN chmod +x /usr/src/app/entrypoint.sh

# copy project
COPY . .

# run entrypoint.sh
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
```
Add the `DATABASE` environment variable to .env.dev:

```bash
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```
Test it out again:

1. Re-build the images
2. Run the containers
3. Try http://localhost:5000/

## Notes

First, despite adding Postgres, we can still create an independent Docker image for Django as long as the `DATABASE` environment variable is not set to `postgres`. To test, build a new image and then run a new container:
```bash
$ docker build -f ./app/Dockerfile -t hello_django:latest ./app
$ docker run -d \
    -p 8006:8000 \
    -e "SECRET_KEY=please_change_me" -e "DEBUG=1" -e "DJANGO_ALLOWED_HOSTS=*" \
    hello_django python /usr/src/app/manage.py runserver 0.0.0.0:8000

```

You should be able to view the welcome page at http://localhost:8006

Second, you may want to comment out the database flush and migrate commands in the entrypoint.sh script so they don't run on every container start or re-start:

```bash
#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

# python manage.py flush --no-input
# python manage.py migrate

exec "$@"
```

Instead, you can run them manually, after the containers spin up, like so:

```bash
$ docker-compose exec web python manage.py flush --no-input
$ docker-compose exec web python manage.py migrate
```

## Gunicorn
Moving along, for production environments, let's add Gunicorn, a production-grade WSGI server, to the requirements file:

```bash
Django==3.2.6
gunicorn==20.1.0
psycopg2-binary==2.9.1
```
Since we still want to use Django's built-in server in development, create a new compose file called docker-compose.prod.yml for production:
```yml
version: '3.8'

services:
  web:
    build: ./app
    command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
    ports:
      - 8000:8000
    env_file:
      - ./.env.prod
    depends_on:
      - db
  db:
    image: postgres:13.0-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db

volumes:
  postgres_data:
```

Take note of the default `command`. We're running Gunicorn rather than the Django development server. We also removed the volume from the `web` service since we don't need it in production. Finally, we're using separate environment variable files to define environment variables for both services that will be passed to the container at runtime.

.env.prod:

```bash
DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

.env.prod.db:

```bash
POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_prod
```
Add the two files to the project root. You'll probably want to keep them out of version control, so add them to a .gitignore file.

Bring down the development containers (and the associated volumes with the `-v` flag):

```bash
$ docker-compose down -v
```
Then, build the production images and spin up the containers:
```bash
$ docker-compose -f docker-compose.prod.yml up -d --build
```
Verify that the `hello_django_prod` database was created along with the default Django tables. Test out the admin page at http://localhost:8000/admin. The static files are not being loaded anymore. This is expected since Debug mode is off. We'll fix this shortly.

## Production Dockerfile

we're still running the database flush (which clears out the database) and migrate commands every time the container is run This is fine in development, but let's create a new entrypoint file for production.

entrypoint.prod.sh:

```bash
#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

exec "$@"
```
Update the file permissions locally:

```bash
$ chmod +x app/entrypoint.prod.sh
```

To use this file, create a new Dockerfile called Dockerfile.prod for use with production builds:

```yml
###########
# BUILDER #
###########

# pull official base image
FROM python:3.9.6-alpine as builder

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# lint
RUN pip install --upgrade pip
RUN pip install flake8==3.9.2
COPY . .
RUN flake8 --ignore=E501,F401 .

# install dependencies
COPY ./requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt


#########
# FINAL #
#########

# pull official base image
FROM python:3.9.6-alpine

# create directory for the app user
RUN mkdir -p /home/app

# create the app user
RUN addgroup -S app && adduser -S app -G app

# create the appropriate directories
ENV HOME=/home/app
ENV APP_HOME=/home/app/web
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

# install dependencies
RUN apk update && apk add libpq
COPY --from=builder /usr/src/app/wheels /wheels
COPY --from=builder /usr/src/app/requirements.txt .
RUN pip install --no-cache /wheels/*

# copy entrypoint.prod.sh
COPY ./entrypoint.prod.sh .
RUN sed -i 's/\r$//g'  $APP_HOME/entrypoint.prod.sh
RUN chmod +x  $APP_HOME/entrypoint.prod.sh

# copy project
COPY . $APP_HOME

# chown all the files to the app user
RUN chown -R app:app $APP_HOME

# change to the app user
USER app

# run entrypoint.prod.sh
ENTRYPOINT ["/home/app/web/entrypoint.prod.sh"]
```

Here, we used a Docker multi-stage build to reduce the final image size. Essentially, `builder` is a temporary image that's used for building the Python wheels. The wheels are then copied over to the final production image and the `builder` image is discarded.

By default, Docker runs container processes as root inside of a container. This is a bad practice since attackers can gain root access to the Docker host if they manage to break out of the container. If you're root in the container, you'll be root on the host.

Update the `web` service within the docker-compose.prod.yml file to build with Dockerfile.prod:

```yml

...
web:
  build:
    context: ./app
    dockerfile: Dockerfile.prod
  command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
  ports:
    - 8000:8000
  env_file:
    - ./.env.prod
  depends_on:
    - db
...

```

Try it out:

```bash
$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput

```

## Nginx

Next, let's add Nginx into the mix to act as a reverse proxy for Gunicorn to handle client requests as well as serve up static files.

Add the service to docker-compose.prod.yml:

```yml
nginx:
  build: ./nginx
  ports:
    - 1337:80
  depends_on:
    - web
```

Then, in the local project root, create the following files and folders:

```bash
└── nginx
    ├── Dockerfile
    └── nginx.conf
```
Dockerfile:

```yml
FROM nginx:1.21-alpine

RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d

```
nginx.conf:

```bash
upstream hello_django {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

}
```
Then, update the `web` service, in docker-compose.prod.yml, replacing `ports` with `expose`:

```yml
...
web:
  build:
    context: ./app
    dockerfile: Dockerfile.prod
  command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
  expose:
    - 8000
  env_file:
    - ./.env.prod
  depends_on:
    - db
...
```

Now, port 8000 is only exposed internally, to other Docker services. The port will no longer be published to the host machine.

Test it out again.

```bash
$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
```

Ensure the app is up and running at http://localhost:1337.

Your project structure should now look like:

```bash
├── .env.dev
├── .env.prod
├── .env.prod.db
├── .gitignore
├── app
│   ├── Dockerfile
│   ├── Dockerfile.prod
│   ├── entrypoint.prod.sh
│   ├── entrypoint.sh
│   ├── hello_django
│   │   ├── __init__.py
│   │   ├── asgi.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── manage.py
│   └── requirements.txt
├── docker-compose.prod.yml
├── docker-compose.yml
└── nginx
    ├── Dockerfile
    └── nginx.conf
```
Bring the containers down once done:

```bash
$ docker-compose -f docker-compose.prod.yml down -v
```

Since Gunicorn is an application server, it will not serve up static files. So, we should both static and media files be handled in this particular configuration.

## Static Files

To test out the handling of media files, start by creating a new Django app:

```bash
$ docker-compose up -d --build
$ docker-compose exec web python manage.py startapp webpage
```
Add the new app to the `INSTALLED_APPS` list in settings.py:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "webpage",
]
```
app/webpage/views.py:

```python
from django.shortcuts import render
from django.http import HttpResponse
# Create your views here.

def home(request):
    #return HttpResponse("<h1>Hello World</h1>")
    return render(request, 'index.html')
```

Add a "templates", directory to the "app/webpage" directory, and then add a new template called index.html file and static folder which contains css, images and webfonts folders.

app/hello_django/urls.py:

```python

from django.contrib import admin
from django.urls import path
from django.conf import settings
from django.conf.urls.static import static

from webpage.views import home

urlpatterns = [
    path("index/", home, name="home"),
    path('admin/', admin.site.urls),
]
```

Update settings.py:

```python
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
```

Development:

Now, any request to http://localhost:8000/static/* will be served from the "staticfiles" directory.

To test, first re-build the images and spin up the new containers per usual. Ensure static files are still being served correctly at http://localhost:8000/admin.

Production:

For production, add a volume to the web and nginx services in docker-compose.prod.yml so that each container will share a directory named "staticfiles":

```yml
version: '3.8'

services:
  web:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
    command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - static_volume:/home/app/web/staticfiles
    expose:
      - 8000
    env_file:
      - ./.env.prod
    depends_on:
      - db
  db:
    image: postgres:13.0-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db
  nginx:
    build: ./nginx
    volumes:
      - static_volume:/home/app/web/staticfiles
    ports:
      - 1337:80
    depends_on:
      - web

volumes:
  postgres_data:
  static_volume:
```
We need to also create the "/home/app/web/staticfiles" folder in Dockerfile.prod:

```yml
...

# create the appropriate directories
ENV HOME=/home/app
ENV APP_HOME=/home/app/web
RUN mkdir $APP_HOME
RUN mkdir $APP_HOME/staticfiles
WORKDIR $APP_HOME

...
```

Docker Compose normally mounts named volumes as root. And since we're using a non-root user, we'll get a permission denied error when the `collectstatic` command is run if the directory does not already exist

To get around this, you can either:

1. Create the folder in the Dockerfile
2. Change the permissions of the directory after it's mounted

We used the former.

Next, update the Nginx configuration to route static file requests to the "staticfiles" folder:

```bash
upstream hello_django {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/staticfiles/;
    }

}
```
Spin down the development containers:
```bash
$ docker-compose down -v
```
Test:

```bash
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
$ docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
```
Again, requests to http://localhost:1337/index will be geting our index.html file and with static files.

Navigate to http://localhost:1337/admin and ensure the static assets load correctly.

You can also verify in the logs -- via `docker-compose -f docker-compose.prod.yml logs -f` -- that requests to the static files are served up successfully via Nginx.

Bring the containers once done:
```bash
$ docker-compose -f docker-compose.prod.yml down -v
```

## Media Files

To test out the handling of media files, start by creating a new Django app:

```bash
$ docker-compose up -d --build
$ docker-compose exec web python manage.py startapp upload
```
Add the new app to the `INSTALLED_APPS` list in settings.py:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "webpage"
    "upload",
]
```

app/upload/views.py:

```python
from django.shortcuts import render
from django.core.files.storage import FileSystemStorage


def image_upload(request):
    if request.method == "POST" and request.FILES["image_file"]:
        image_file = request.FILES["image_file"]
        fs = FileSystemStorage()
        filename = fs.save(image_file.name, image_file)
        image_url = fs.url(filename)
        print(image_url)
        return render(request, "upload.html", {
            "image_url": image_url
        })
    return render(request, "upload.html")
```

Add a "templates", directory to the "app/upload" directory, and then add a new template called upload.html:

```python
{% block content %}

  <form action="{% url "upload" %}" method="post" enctype="multipart/form-data">
    {% csrf_token %}
    <input type="file" name="image_file">
    <input type="submit" value="submit" />
  </form>

  {% if image_url %}
    <p>File uploaded at: <a href="{{ image_url }}">{{ image_url }}</a></p>
  {% endif %}

{% endblock %}
```

app/hello_django/urls.py:

```python
from django.contrib import admin
from django.urls import path
from django.conf import settings
from django.conf.urls.static import static

from upload.views import image_upload
from webpage.views import home

urlpatterns = [
    path("", image_upload, name="upload"),
    path("index/", home, name="home"),
    path('admin/', admin.site.urls),
]

if bool(settings.DEBUG):
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```
app/hello_django/settings.py:

```python
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "mediafiles"
```

Development:

Test:
```bash
$ docker-compose up -d --build
```

You should be able to upload an image at http://localhost:8000/, and then view the image at http://localhost:8000/media/IMAGE_FILE_NAME.

Production
For production, add another volume to the `web` and `nginx` services:

```yml
version: '3.8'

services:
  web:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
    command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - static_volume:/home/app/web/staticfiles
      - media_volume:/home/app/web/mediafiles
    expose:
      - 8000
    env_file:
      - ./.env.prod
    depends_on:
      - db
  db:
    image: postgres:13.0-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - ./.env.prod.db
  nginx:
    build: ./nginx
    volumes:
      - static_volume:/home/app/web/staticfiles
      - media_volume:/home/app/web/mediafiles
    ports:
      - 1337:80
    depends_on:
      - web

volumes:
  postgres_data:
  static_volume:
  media_volume:

```
Create the "/home/app/web/mediafiles" folder in Dockerfile.prod:

```yml
...

# create the appropriate directories
ENV HOME=/home/app
ENV APP_HOME=/home/app/web
RUN mkdir $APP_HOME
RUN mkdir $APP_HOME/staticfiles
RUN mkdir $APP_HOME/mediafiles
WORKDIR $APP_HOME

...
```
Update the Nginx config again:

```bash
upstream hello_django {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/staticfiles/;
    }

    location /media/ {
        alias /home/app/web/mediafiles/;
    }

}
```
Re-build:
```bash
$ docker-compose down -v

$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
$ docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
```
Test it out one final time:

1. Upload an image at http://localhost:1337/.
2. Then, view the image at http://localhost:1337/media/IMAGE_FILE_NAME.

## GitHub

Upload this project to github
Create a repo in github and clone that repo and copy all the project files into that folder 
Add .gitignore file to ignore some of these files like env, .env.dev, .env.prod, .env.prod.db
commands to add this project to github:
```bash
git status
git add .
git commit -m "Commit message"
git push 
```
## Deploy this project on EC2 Server
Create an ec2 server in AWS account
And download the .pem file 
open command propt and change directory to .pem file location 
Next ssh into the EC2 server through base system

```bash 
 ssh -i file.pem username@ip-address
```
This is the explanation of the previous command:

1. ssh: Command to use SSH protocol
2. -i: Flag that specifies an alternate identification file to use for public key authentication.
3. username: Username that uses your instance
4. ip-address: IP address given to your instance

After pressing enter, a question will prompt to add the host to your known_hosts file. Type yes. This will help to recognize the host each time you’re trying to connect to your instance.

And that’s it! Now you’re logged in on your AWS instance

After that you can install some packages like docker .

```bash 
$ sudo apt update
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
$ sudo apt update
$ sudo apt install docker-ce
$ sudo usermod -aG docker ${USER}
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose

$ docker -v
Docker version 20.10.8, build 3967b7d

$ docker-compose -v
docker-compose version 1.29.2, build 5becea4c
```
Now docker installed successfully
Create a project folder and clone the project from github

```bash
$ git clone <project-url>
$ ls
$ cd <project-folder-name>
```
After that add .env.prod and .env.prod.db because we ignore these file while adding this project into github
Note: While adding .env.prod file please make sure that Allowed_Host=(public-ip-ec2Server)
```bash
DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=<public-ip-ec2Server>
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```  
Now run the following commands

```bash
$ sudo docker-compose -f docker-compose.prod.yml up -d --build
$ sudo docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
$ sudo docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear
```

## Note:
If you get any error like this

```bash
ERROR: for django-project1_web_1  Cannot start service web: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "/home/app/web/entrypoint.prod.sh": permission denied: unknown

ERROR: for web  Cannot start service web: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "/home/app/web/entrypoint.prod.sh": permission denied: unknown
ERROR: Encountered errors while bringing up the project.
```

lets change the permitions of `entrypoint.prod.sh`
```bash
sudo chmod +x entrypoint.prod.sh
```

Now you can test the output with public ip of our ec2 server.

1. Index file of Webpage at (public-ip):1337/index
2. Upload an image at (public-ip):1337/
3. Then, view the image at (publlic-ip):1337/media/IMAGE_FILE_NAME.
