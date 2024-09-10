---
notion_page: https://www.notion.so/wpmedia/Python-Repository-preparation-3e292f3b1ffc4ee3adfdf1aceefa3fe2?pvs=4
title: Python - Repository preparation
---

# Python - Repository preparation

## Requirements

Dependencies for a codebase usually differ based on the environment the code is running: linters and test tools are required for development and CI, but not for production for instance. Therefore, we recommend using separate requirement files to ease dependency management and keep the footprint of the app small.

In the app repository, we use a `requirements/` directory, containing the following files:
- `base.txt`: this file lists the dependencies needed in all contexts;
- `ci.txt`: this file lists the dependencies needed in a CI context, on top of `base.txt`. It usually begins with `-r base.txt`.
- `dev.txt`: this file lists the dependencies needed in a development environment. It usually begins with `-r ci.txt`.
- `prod.txt`: this file lists the dependencies only needed in production and deployed environments. It usually begins with `-r base.txt`.

Note that those files are related to each others as some requirements include other ones. To achieve this, we use the following syntax as first line of a file, so that it includes all dependencies from the referenced file:

```
`-r base.txt`
```

To automatically generate those files, you can use:

```
mkdir requirements
touch requirements/{base.txt,ci.txt,dev.txt,prod.txt}
```

## Configuration file

As we use various tools to enhance the developer experience, it is easier to gather all configuration within a single `.toml` file.
In the project’s root directory, create a `pyproject.toml` file. This will include all the configuration for third party packages.

```bash
touch pyproject.toml
```

## Setting up a local virtual environment

As developers often switch from one repository to another, and that the environment usually differ (different dependencies, versions, etc.), it is safer to use a virtual environment per project. This will allow you to install dependencies for this repository only, and maybe use different ones on another repository without interferences.
To create and run a virtual environment, you can use the following commands in the repository:

```bash
# create the virtual env
python3 -m venv .venv
# activate the virtualenv
. .venv/bin/activate
```

## Setting up Docker containers

Being able to run our apps locally is critical to ensure efficient development and debug processes. Ensuring that what happens locally is as close as possible to production is also very important to avoid wasting time on "local only" issues. To achieve this, all our Python projects must be ready to run locally within docker containers.
If you are not familiar with Docker, check out their [Getting started guide](https://docs.docker.com/get-started/).

Docker containers are also used on our live servers as we deploy them within kubernetes pods.

To prepare the app to run on Docker, a few files must be added and configured. The following chapter presents the baseline configuration we use at WP Media. Note that this can then evolve based on the needs of the project.

### Development container

In the project root, create a *docker* directory, and inside that create an *app* directory.

```bash
touch docker-compose.yaml
mkdir -p docker/app/
touch docker/app/dev.Dockerfile
```



In the `dev.Dockerfile`, paste the following code.

```docker
FROM python:3.11-bullseye

RUN set -xe;

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# The following configuration is only needed for Django apps.
ENV DJANGO_ALLOW_ASYNC_UNSAFE="true"
ENV DJANGO_SETTINGS_MODULE=ticket_ai.settings.dev

# Create a user named app, to run as that instead of root.
RUN groupadd -r app && useradd --create-home --no-log-init -u 1000 -r -g app app && mkdir /app && chown app:app /app
WORKDIR /app

RUN mkdir /app/requirements/
ADD ./requirements/* /app/requirements/
RUN pip install --no-cache-dir --upgrade pip && pip install --no-cache-dir -r requirements/dev.txt
ADD . /app

USER app

EXPOSE 8000/tcp
CMD python3 /app/manage.py migrate --noinput && python3 /app/manage.py runserver 0.0.0.0:8000
```

In the *./docker-compose.yaml* paste the following code.

```yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/app/dev.Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - .:/app

```

Run docker-compose and visit http://127.0.0.1:8000. You should see the json response hello world.

```bash
# to build the image.
docker-compose build

# to start the app again.
docker-compose up

```

At this point we already have the app running in the container.

### Production containers

The previous section describes the local setup. For production needs, we use a different Dockerfile. It typically defines an image with smaller footprint as we don't need debug tools in production, uses an uwsgi production server, and it also points to the production specific requirements.

```bash
touch docker/app/Dockerfile
```

```docker
FROM python:3.11-slim

RUN set -xe;

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# The following configuration is only needed for Django apps.
ENV DJANGO_ALLOW_ASYNC_UNSAFE="true"
ENV DJANGO_SETTINGS_MODULE=repository_name.settings.prod

# Create a user named app, to run as that instead of root.
RUN groupadd -r app && useradd --create-home --no-log-init -u 1000 -r -g app app && mkdir /app && chown app:app /app
WORKDIR /app

RUN mkdir /app/requirements/

COPY ./docker/app/entrypoint.sh /usr/local/bin/
ADD ./requirements/* /app/requirements/
RUN pip install --no-cache-dir --upgrade pip && pip install --no-cache-dir -r requirements/prod.txt
ADD . /app

USER app

# Add .local/bin to PATH (this is where python bins are put when installed through pip)
ENV PATH="${PATH}:/home/app/.local/bin"

EXPOSE 8000/tcp
ENTRYPOINT ["tini", "--", "entrypoint.sh"]
CMD ["tini", "--", "uwsgi", "--ini", "uwsgi.ini"]
```

In the production configuration, the app is started through a script `entrypoint.sh`. You must create it in `docker/app/` and use the following content:

```
#!/bin/sh
set -e

exec "$@"
```

Note that further instructions can be added to this file to be performed when starting the app. For instance, for Django apps, we usually add `python manage.py migrate` before `exec "$@"`.

#### uwsgi

For production, we rely on [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) as server suite.

uWSGI must be added to `requirements/prod.txt`:

```
`-r base.txt`

uWSGI==<latest_version>
```

Then, a `uwsgi.ini` configuration file must be added at the root of the repository:

```
touch uwsgi.ini
```

The following file content is a baseline we recommend for our Python apps. Note that this can evolve based on the needs of the app, and that some values there must be configured as environment variables as part of the kubernetes deployment.

```
[uwsgi]
strict = true
uid = app
gid = app
env = DJANGO_SETTINGS_MODULE=$(DJANGO_SETTINGS_MODULE)
env = LANG=en_US.UTF-8
chdir = /app
module = repository_name.wsgi:application
pidfile = /tmp/repository_name-master.pid
master = True
vacuum = True
max-requests = $(UWSGI_MAX_REQUESTS)
processes = $(UWSGI_PROCESSES)
harakiri = $(UWSGI_HARAKIRI)
lazy-apps = true
enable-threads = true
http-socket = :8000
http-enable-proxy-protocol = 1
log-x-forwarded-for = true
ignore-sigpipe = true
ignore-write-errors = true
disable-write-exception = true
```

The following articles are good references about the use of `uwsgi`, and provide insights about the above configuration:
- https://www.bloomberg.com/company/stories/configuring-uwsgi-production-deployment/
- https://blog.ionelmc.ro/2022/03/14/how-to-run-uwsgi/
