# Django Development with Docker Compose and Machine

<div class="center-text">
  <img class="no-border" src="/images/blog_images/dockerizing-django/django-docker.png" style="max-width: 100%;" alt="django docker">
</div>

<br>

[Docker](https://www.docker.com/) is a containerization tool used for spinning up isolated, reproducible application environments. **This piece details how to containerize a Django Project, Postgres, and Redis for local development along with delivering the stack to the cloud via [Docker Compose](https://docs.docker.com/compose/) and [Docker Machine](http://docs.docker.com/machine/).**

<br>

In the end, the stack will include a separate container for each service:

- 1 web/Django container
- 1 nginx container
- 1 Postgres container
- 1 Redis container
- 1 data container

<br>

<div>
  <img class="no-border" src="/images/blog_images/dockerizing-django/container-stack.png" style="max-width: 100%;" alt="container stack diagram">
</div>

<br>

> Interested in creating a similar environment for Flask? Check out [this](https://realpython.com/blog/python/dockerizing-flask-with-compose-and-machine-from-localhost-to-the-cloud/) blog post.

## Local Setup

Along with Docker (v1.6.1) we will be using -

- *[Docker Compose](https://docs.docker.com/compose/)* (v1.2.0) for orchestrating a multi-container application into a single app, and
- *[Docker Machine](https://docs.docker.com/machine/)* (v0.2.0) for creating Docker hosts both locally and in the cloud.

Follow the directions [here](http://docs.docker.com/compose/install/) and [here](https://docs.docker.com/machine/#installation) to install Docker Compose and Machine, respectively. Then test out the installs:

```sh
$ docker-machine --version
docker-machine version 0.2.0 (8b9eaf2)
$ docker-compose --version
docker-compose 1.2.0
```

Next clone the project from the [repository](https://github.com/realpython/dockerizing-django) or create your own project based on the project structure found on the repo:

```sh
├── docker-compose.yml
├── nginx
│   ├── Dockerfile
│   └── sites-enabled
│       └── django_project
├── production.yml
└── web
    ├── Dockerfile
    ├── docker_django
    │   ├── __init__.py
    │   ├── apps
    │   │   ├── __init__.py
    │   │   └── todo
    │   │       ├── __init__.py
    │   │       ├── admin.py
    │   │       ├── models.py
    │   │       ├── templates
    │   │       │   ├── _base.html
    │   │       │   └── home.html
    │   │       ├── tests.py
    │   │       ├── urls.py
    │   │       └── views.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    ├── requirements.txt
    └── static
        └── main.css
```

We're now ready to get the containers up and running...

## Docker Machine

To start Docker Machine, simply run:

```sh
$ docker-machine create -d virtualbox dev;
INFO[0000] Creating CA: /Users/michael/.docker/machine/certs/ca.pem
INFO[0000] Creating client certificate: /Users/michael/.docker/machine/certs/cert.pem
INFO[0001] Downloading boot2docker.iso to /Users/michael/.docker/machine/cache/boot2docker.iso...
INFO[0035] Creating SSH key...
INFO[0035] Creating VirtualBox VM...
INFO[0043] Starting VirtualBox VM...
INFO[0044] Waiting for VM to start...
INFO[0094] "dev" has been created and is now the active machine.
INFO[0094] To point your Docker client at it, run this in your shell: eval "$(docker-machine env dev)"
```

The `create` command setup a new "Machine" (called *dev*) for Docker development. In essence, it downloaded [boot2docker](http://boot2docker.io/) and started a VM with Docker running. Now just point Docker at the *dev* machine:

```sh
$ eval "$(docker-machine env dev)"
```

Run the following command to view the currently running Machines:

```sh
$ docker-machine ls
NAME   ACTIVE   DRIVER       STATE     URL
dev    *        virtualbox   Running   tcp://192.168.99.100:2376
```

Next, let's fire up the containers with Docker Compose and get Django, Postgres, and Redis up and running.

## Docker Compose

Let's take a look at the *docker-compose.yml* file:

```yaml
web:
  restart: always
  build: ./web
  expose:
    - "8000"
  links:
    - postgres:postgres
    - redis:redis
  volumes:
    - /usr/src/app/static
  env_file: .env
  command: /usr/local/bin/gunicorn docker_django.wsgi:application -w 2 -b :8000

nginx:
  restart: always
  build: ./nginx/
  ports:
    - "80:80"
  volumes:
    - /www/static
  volumes_from:
    - web
  links:
    - web:web

postgres:
  restart: always
  image: postgres:latest
  volumes_from:
    - data
  ports:
    - "5432:5432"

redis:
  restart: always
  image: redis:latest
  ports:
    - "6379:6379"

data:
  restart: always
  image: postgres:latest
  volumes:
    - /var/lib/postgresql
  command: "true"
```

Here, we're defining five services - *web*, *nginx*, *postgres*, *redis*, and *data*.

1. First, the *web* service is built via the instructions in the *Dockerfile* within the "web" directory - where the Python environment is setup, requirements are installed, and the Django application is fired up on port 8000. That port is then forwarded to port 80 on the host environment - e.g., the Docker Machine. This service also adds environment variables to the container that are defined in the *.env* file.
1. The *nginx* service is used for reverse proxy to forward requests either to Django or the static file directory.
1. Next, the *postgres* service is built from the the official [PostgreSQL image](https://registry.hub.docker.com/_/postgres/) from [Docker Hub](https://hub.docker.com/), which installs Postgres and runs the server on the default port 5432.
1. Likewise, the *redis* service uses the official [Redis image](https://registry.hub.docker.com/u/library/redis/) to install, well, Redis and then the service is ran on port 6379.
1. Finally, notice how there is a separate [volume](https://docs.docker.com/userguide/dockervolumes/) container that's used to store the database data - called *data*. This helps ensure that the data persists even if the Postgres container is completely destroyed.

Now, to get the containers running, build the images and then start the services:

```sh
$ docker-compose build
$ docker-compose up -d
```

Grab a cup of coffee. Or go for a *long* walk. This will take a while the first time you run it. Subsequent builds run much quicker since Docker [caches](https://docs.docker.com/articles/dockerfile_best-practices/#build-cache) the results from the first build.

Once the services are running, we need to create the database migrations:

```sh
$ docker-compose run web /usr/local/bin/python manage.py migrate
```

Grab the IP associated with Docker Machine - `docker-machine ip` - and then navigate to that IP in your browser:

<div class="center-text">
  <img class="no-border" src="/images/blog_images/dockerizing-django/django-on-docker.png" style="max-width: 100%;" alt="flask running on docker">
</div>

<br>

Nice!

Try refreshing. You should see the counter update. Essentially, we're using the [Redis INCR](http://redis.io/commands/incr) to increment after each handled request. Check out the code in *web/docker_django/apps/todo/views.py* for more info.

Again, this created five services, all running in different containers:

```sh
$ docker-compose ps
            Name                          Command               State           Ports
----------------------------------------------------------------------------------------------
dockerizingdjango_data_1       /docker-entrypoint.sh true       Up      5432/tcp
dockerizingdjango_nginx_1      /usr/sbin/nginx                  Up      0.0.0.0:80->80/tcp
dockerizingdjango_postgres_1   /docker-entrypoint.sh postgres   Up      0.0.0.0:5432->5432/tcp
dockerizingdjango_redis_1      /entrypoint.sh redis-server      Up      0.0.0.0:6379->6379/tcp
dockerizingdjango_web_1        /usr/local/bin/gunicorn do ...   Up      8000/tcp
```

To see which environment variables are available to the *web* service, run:

```sh
$ docker-compose run web env
```

To view the logs:

```sh
$ docker-compose logs
```

You can also enter the Postgres Shell - since we forward the port to the host environment in the *docker-compose.yml* file - to add users/roles as well as databases via:

```sh
$ psql -h 192.168.99.100 -p 5432 -U postgres --password
```

Ready to deploy? Stop the processes via `docker-compose stop` and let's get the app up in the cloud!

## Deployment

So, with our app running locally, we can now push this exact same environment to a cloud hosting provider with Docker Machine. Let's deploy to a [Digital Ocean](https://www.digitalocean.com/?refcode=d8f211a4b4c2) box.

After you [sign up](https://www.digitalocean.com/?refcode=d8f211a4b4c2) for Digital Ocean, generate a [Personal Access Token](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2), and then run the following command:

```sh
$ docker-machine create \
-d digitalocean \
--digitalocean-access-token=ADD_YOUR_TOKEN_HERE \
production
```

This will take a few minutes to provision the droplet and setup a new Docker Machine called *production*:

```sh
INFO[0000] Creating SSH key...
INFO[0001] Creating Digital Ocean droplet...
INFO[0133] "production" has been created and is now the active machine.
INFO[0133] To point your Docker client at it, run this in your shell: eval "$(docker-machine env production)"
```

Now we have two Machines running, one locally and one on Digital Ocean:

```sh
$ docker-machine ls
NAME         ACTIVE   DRIVER         STATE     URL
dev          *        virtualbox     Running   tcp://192.168.99.100:2376
production            digitalocean   Running   tcp://104.131.107.8:2376
```

Set *production* as the active machine and load the Docker environment into the shell:

```sh
$ docker-machine active production
$ eval "$(docker-machine env production)"
```

Finally, let's build the Django app again in the cloud. This time we need to use a slightly different Docker Compose file that does not mount a [volume](https://docs.docker.com/userguide/dockervolumes/#mount-a-host-directory-as-a-data-volume) in the container. Why? Well, the volume is perfect for local development since we can update our local code in the "web" directory and the changes will immediately take affect in the container. In production, there's no need for this, obviously.

```sh
$ docker-compose build
$ docker-compose up -d -f production.yml
$ docker-compose run web /usr/local/bin/python manage.py migrate
```

Grab the IP address associated with that Digital Ocean account and view it in the browser. If all went well, you should see your app running, as it should.

## Conclusion

- Grab the code from the [repo](https://github.com/realpython/dockerizing-django) (star it too... my self-esteem depends on it!).
- Comment below with questions.
- Need a challenge? Try using [extends](https://docs.docker.com/compose/extends/) to clean up the receptive code in the two Docker Compose configuration files. Keep it DRY!
- Have a great day!
