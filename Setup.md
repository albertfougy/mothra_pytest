This is an attempt to document how this was built, step by step, as a way for those new to web development in python to see an example of one way to built apps. Keep in mind there are many ways to structure apps and processes you can follow. This is just the way I do it. 

Each step in the tutorial links to the commit in the repo history where that code was pushed. 

# Creating an app

## The repo

The first thing I do is create an empty repo in github and make sure to add a README and a .gitignore with the default python ignore rules. If I know how I'm going to be using the code, I will also add the LICENSE at that time. 

Then locally, I run `git clone` with the github url for the repo. Before I go any further, I create a feature branch in my local repo with the intention of keeping the master branch free of any commits that haven't been merged by any method other than an approved pull request. 

```bash
$ git checkout -b setup
```

To that end, I will also create a branch protection rule in github to require pull requests before merging and that all pull requests require at least one approver. 

## [Python dependencies](https://github.com/dchess/mothra/commit/13195aee1a4c9e267c9279c0756c47d3564dd5eb)

I use [pipenv](https://pipenv-fork.readthedocs.io/en/latest/) for dependency and environment management, so the first thing I will do is use it to install the dependencies I will need. In addition to [Django](https://www.djangoproject.com/), I want to include [Django REST Framework](https://www.django-rest-framework.org/). I will want to have some easy functionality for filtering my API calls as well, so I will install [django-filter](https://django-filter.readthedocs.io/en/master/). Lastly, I know I'm going to want a more robust database than the default sqlite, so I also install the [Postgres](https://pypi.org/project/psycopg2/) adapter. 

```bash
$ pipenv install django djangorestframework django-filter psycopg2-binary
```

## [Creating the project](https://github.com/dchess/mothra/commit/4f09f76fcae717866be02c08b2eaa57901ffd3e6)

Django has a command line tool called django-admin than can generate a blank project to get you started. You can do this manually if you want it configured differently or even use something more advanced like [cookiecutter](https://cookiecutter-django.readthedocs.io/en/latest/). The latter tends to bit more than I need so I borrow concepts from it's architecture but don't use it as is. The former also requires writing some boilerplate from scratch, so I tend to just use the django-admin and then make a few quick tweaks. 

```bash
$ cd mothra/
$ pipenv run django-admin startproject mothra .
```

Before I commit these changes, I will remove the secret key that Django creates in the settings.py file. I could change it later before deploying, but I find that if I do it immediately, I am less likely to accidentally expose anything and it saves me time having to regenerate my own secret key later. 

⚠️ I will also move that key to a `.env` file and modify the settings.py file to pull that key from the environment instead. 

At this point, I will do a quick [smoke test](https://en.wikipedia.org/wiki/Smoke_testing_(software)) to ensure everything worked by just running the django server and confirm the default homepage loads correctly. 

```bash
$ pipenv run python manage.py runserver
```

If it works, I will see the following message in the terminal:

```bash
Loading .env environment variables…
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 17 unapplied migration(s). Your project may not work properly until 
you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.

October 23, 2019 - 00:30:02
Django version 2.2.6, using settings 'mothra.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

And if I open a browser and go to localhost:8000, I should see the default Django page.

[[images/django_default_homepage.png]]

### [First tweak: change the project folder name](https://github.com/dchess/mothra/commit/7daf9ae8c30f91a0612f52042816ec31ff22c9a0)

I like to rename the project folder itself to config because it mostly contains the project settings, wsgi server config, and the main url router. I find it a little confusing when my repo is the projectname and there is also a folder with the same name inside it and anything that reduces my cognitive load gives me more mental bandwidth to dedicate to writing the code. 

This name change requires also making a few changes in settings.py, wsgi.py, and manage.py. Once again, a simple smoke test after those changes will prevent heart-ache later. 

```bash
$ pipenv run python manage.py runserver
```

### [Second Tweak: change the database](https://github.com/dchess/mothra/commit/96b0cae91e725849a33d04a3d40b2d96503871b9)

By default, Django uses sqlite as its database, but this will not stand up to a production deployment with data volumes of any substantial size (although to be fair some folks have been able to squeeze quite a bit of performance out of sqlite). Postgres plays really nicely with Django and has native support in Heroku which is where I intend to deploy this initially. 

Replace this section of settings.py:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": os.path.join(BASE_DIR, "db.sqlite3"),
    }
}
```

With this snippet (which will look for database credentials in the environment):

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.getenv("POSTGRES_DB"),
        "USER": os.getenv("POSTGRES_USER"),
        "PASSWORD": os.getenv("POSTGRES_PASSWORD"),
        "HOST": os.getenv("POSTGRES_HOST", "db"),
        "PORT": os.getenv("POSTGRES_PORT", 5432),
    }
}
```

This serves two purposes: (1) It makes sure I don't expose my database credentials in the repo by accidentally hard-coding them into my settings and (2) makes it easy to change them easily between local development and production deployment. 

At this point I think it's important to step back and get our Docker environment setup and working to handle our local database and web server.

## [Docker setup](https://github.com/dchess/mothra/commit/25e14bc92787338ac4511fbda0d80466e3331903)

For the web app server, I use the [python 3 docker image](https://hub.docker.com/_/python) and add some additional config to leverage pipenv. I create a `Dockerfile` to configure the image for the app. 

### What is a Dockerfile?
From the [Docker Documentation](https://docs.docker.com/engine/reference/builder/):

>Docker can build images automatically by reading the instructions from a `Dockerfile`. A `Dockerfile` is a text document that contains all the commands a user could call on the command line to assemble an image. Using `docker build` users can create an automated build that executes several command-line instructions in succession.

### Explanation of the Directives

```docker
FROM python:3
ENV PYTHONUNBUFFERED 1
WORKDIR /code
RUN pip install pipenv
COPY Pipfile .
RUN pipenv install --system --skip-lock
COPY ./ .
```

Let's go through what each line is doing here.

The first directive `FROM python:3` creates the docker image based on the python 3 image from dockerhub. 

The next directive `ENV PYTHONUNBUFFERED 1` sets an environment variable. This prevents Docker from buffering the output from Python in the standard output (stdout). This is a good practice in terms of memory usage and becomes relevant once we start using logging.

The directive `WORKDIR /code` creates the working directory and all the directives that follow will be executed in that directory. I called the directory `code` to identify that this is where I will store all of the code powering the web app server.

The next few commands install the pipenv python library for managing the app's dependencies, copy the `Pipfile` into the docker image from our repo, then installs the dependencies from the `Pipfile`. The install step is passed an additional parameter to bypass installing the dependencies in an virtual env and instead [installing them system wide](https://pipenv.readthedocs.io/en/latest/advanced/#deploying-system-dependencies).

Given that the code is already run in a containerized environment, it is an additional layer of abstraction that I think is unnecessary. It also allows running python with standard commands instead of needing to prefix all commands with `pipenv run` or needing to open the `pipenv shell`. I don't know if this is strictly a best practice or not, but it has made some of the config easier to read and understand.

The final directive copies all files and directories in the project repo into the image. Notice that this is run after the pipenv directives. This is intentional. We first copy in the `Pipfile` alone in order to run the dependency installation step, but if we copy all project files in prior, if even a single python file changes (as it likely will during development) a new build will kick off the dependencies installs all over again, even if nothing has changed in them. 

This is due to [the way Docker handles build caching](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#add-or-copy):

>If you have multiple `Dockerfile` steps that use different files from your context, `COPY` them individually, rather than all at once. This ensures that each step’s build cache is only invalidated (forcing the step to be re-run) if the specifically required files change.

## [Using Docker-Compose](https://github.com/dchess/mothra/commit/34ae581bbc5d6e2ea0e7b0a7e7fe2150b62bf5bf)

### What is Docker-Compose?

From the [Docker Compose Documentation](https://docs.docker.com/compose/):

>Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.

### Why do we need this?

For this web app, we'll have two containers (Django web app and PostgreSQL database) that will need to work in tandem and compose will make this much more manageable to coordinate.

### Explanation of the YAML Config

The Compose config has three main sections. The first two define the database and web app services and the last defines a named volume to persist the data from the database.

```yaml
services:
  db:
    restart: always
    image: postgres:11
    volumes:
      - mothra_data:/var/lib/postgresql/data/
    env_file:
      - .env
    ports:
      - 5432
  web:
    restart: always
    build: .
    env_file:
      - .env
    command: python manage.py runserver 0.0.0.0:80
    volumes:
      - .:/code
    ports:
      - "80:80"

volumes:
  mothra_data:
```

What's going on here?

```yaml
 web:
    restart: always
    build: .
    env_file:
      - .env
    command: python manage.py runserver 0.0.0.0:80
    volumes:
      - .:/code
    ports:
      - "80:80"
```

For the web app the config does a few basic things. The command `restart: always` ensures the container will restart automatically as needed. This is helpful because the database needs to up for the web app to connect and this will force the web app to keep checking for the database and retry running until it works. 

Next `build: .` tells Compose to build the web app image from the `Dockerfile` in the current directory. The `env-file` directive then tells Compose where to find the environment variables file.

We then definte what command we want to execute when the container runs with `command: python manage.py runserver 0.0.0.0:80`. This runs Django's local development webserver and maps it to port 80 which will make it available as `http://localhost` in our web browser. 

The `volumes` command then maps our entire project repo to the working directory inside the container. This is so we can do hot reloads while developing. This means we can have the app running, make changes to our codebase, and see them change in the browser with a simple page refresh instead of needing to spin the container down and do a fresh build or restart the containers.

Lastly, it maps port 80 inside the container (where the `runserver` command is serving our app) to our host environment port so we can access the app outside of Docker. 

This all could be configured as a run command but imagine having to remember all those things everytime you wanted to run the app. What a nightmare! This way we define it once and then can forget it until we need to make changes.

```yaml
db:
    restart: always
    image: postgres:11
    volumes:
      - mothra_data:/var/lib/postgresql/data/
    env_file:
      - .env
    ports:
      - 5432
```

For the database, we do some similar things. I'll focus on what's different here.

We aren't going to modify the published image for postgres so we don't need our own `Dockerfile`. We can just point to a public image and version. We'll want version 11 of postgres so we define the command `image: postgres:11`.

We map the location of the database storage files inside the containter to a named volume in our host system called `mothra_data` that we define separately. 

We define what port postgres will accept connection on with the `ports` command, but don't open it externally to the host system. Only the other Docker container needs access and Compose will create a network between them internally.

Finally, we define the named volume for our database with the following section:

```yaml
volumes:
  mothra_data:
```

### Updating the env vars

At this point we need to add our database connection credentials to our `.env` file before spinning up our containers so our app will be able to talk to the database.

We add the following to our `.env`:

```bash
POSTGRES_DB=mothra
POSTGRES_USER=mothra_admin
POSTGRES_PASSWORD=mysecretpassword
```

Replace the password with your own secure password.

### Bring up your container network

Now we're ready to spin up our containers and do another smoke test to confirm all this Docker configuration hasn't broken anything.

```bash
$ docker-compose build
$ docker-compose up
```

We should see some logging to stdout and a similar success message. Likewise navigating to `http://localhost` should produce the default Django homepage. You can abort the services with `CTRL+C` in the terminal when you're ready to move on.

We will normally run Compose detached (as a background worker) and can even run both the build and the up commands in a single command:

```bash
$ docker-compose up -d --build
```

When we want the services to stop, we run:

```bash
$ docker-compose down
```

**Congrats! You've successfully spun up a multi-container network ready for development. We'll dig into some actual Django development in the next section of the tutorial.**


