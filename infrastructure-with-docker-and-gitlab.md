# Setup and maintain Infrastructure with Docker and GitLab

__Sven Haardiek, 2020-10-09__

Companies and Organization need different services for their daily work.
Things like chat systems to communicate or Wikis or very basic things like DNS or DHCP server.

In smaller companies and organization those services are often setup and maintained manually, since automation would mean an initial overhead.

I want to show a simple kind of automatic setup using [Docker](https://www.docker.com/) and [GitLab](https://about.gitlab.com/) as the base.
Both are Open Source and easy to setup.
Furthermore Docker is a [OS-level virtualization](https://en.wikipedia.org/wiki/OS-level_virtualization) with very little overhead and GitLab is even available as a [Software as a Service](https://en.wikipedia.org/wiki/Software_as_a_service) and for small amount of work also at no charge.

## Prerequities

Obviously you need a server with the [Docker Engine](https://docs.docker.com/engine/) installed.
This will be the workhorse were all the services will run.
I refer to the [Docker Installation Guide](https://docs.docker.com/engine/install/) for further instruction.
During this article we will use `docker.server` as the DNS name for this server.

Later we will using [SSH](https://en.wikipedia.org/wiki/Ssh_(Secure_Shell)) to the Docker server ([TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) is also possible).
So you need a user on the server who can connect via SSH with a key pair and is also in the `docker` group.
We will use the user `docker` in this guide.

Next to this server you will need a [GitLab](https://about.gitlab.com/) instance and a [GitLab Runner](https://docs.gitlab.com/runner/).
It is possible to use a self-hosted GitLab instance or use the [SaaS Version on gitlab.com](https://gitlab.com).
The same is true for the GitLab Runner.
You can either setup your own and connect it to GitLab or use the pool of free runner from gitlab.com.
It has to be possible for the runner to access the Docker server via SSH in some way.
They can run in the same network, can be both connected to the internet or they can use some kind of SSH jumphost.
This does have some security implication you always should be considering.

## The Setup

Since our imaginary organization wants to publish some services for their employees, we are using the reverse [nginx](https://www.nginx.com/) as an example service to setup.

So we are creating a new repository on our GitLab instance for our nginx service.

This repository will contain the nginx configuration `nginx.conf`, which we will not describe any further, a `Dockerfile` and `docker-compose.yml` for the service setup on the Docker server and a `.gitlab-ci.yml` for the automatic deployment and maintainance.

The generell workflow for the GitLab CI will be the following:

![GitLab CI Workflow](images/infrastructure-with-docker-and-gitlab/gitlab-ci-workflow.jpg)

### Docker

So first lets take a look a the `Dockerfile`:

```docker
FROM nginx:latest
COPY nginx.conf /etc/nginx/
```

Of course it is possible to add more that a single configuration file to the new image, e.g. splitted vhost configurations or TLS certificates, but the concept would be the same.

But why do we create a new image instead of mounting the configuration to the nginx Docker image?
Well, we will deploy this remote via SSH so we can not mount the configuration file, since it does not exist on the remote server.

Okay so after we know how to create the `Dockerfile`, let's take a look at the `docker-compose.yml`

```yml
---

version: '2'
services:
  nginx:
    build: .
    ports:
      - 80:80
      - 443:443
```

We stripped this confiration down to the essential things.
Nginx needs to be accessable from the outside, so we have to map the ports `80` and `443`.
Also we use the [build](https://docs.docker.com/compose/compose-file/compose-file-v2/#build) keyword to automatically build the Docker image.

For more complex services the `docker-compose.yml` can of course consist of multiple services, mount some volumes (remember relative path are not possible) and so on.

So now we have a docker service setup, which we can also test locally with the `docker` and `docker-compose` command line tools, but our ultimate goal is to automate the process of deploying new configuration and doing updates, so let's talk about the CI itself.

### GitLab CI

First we have to connect the runner to the Docker server.
For this we are using `SSH` which means that we need to know the [SSH host keys](https://www.ssh.com/ssh/host-key) from the Docker server and the private key from the key pair for the `docker` user we mentioned above.
We will store those information in GitLab as [CI/CD Variables](https://docs.gitlab.com/ce/ci/variables/) which you can add under `Settings->CI/CD` in your repository.
So lets add those information

First the host key

![Host Key](images/infrastructure-with-docker-and-gitlab/host-key.jpg)

and then the private ssh key

![Private Key](images/infrastructure-with-docker-and-gitlab/private-key.jpg)

Now we are ready to glue everything together.
Let's take a look at the `gitlab-ci.yml`

```yml
---

deploy:
  stage: deploy
  image: docker:stable
  # Only deploy new configuration from the master
  only:
    - master
  before_script:
    # Install prerequites.
    # We need the docker client, docker-compose and an ssh client
    - apk --no-cache add openssh-client python3 python3-dev py3-pip libffi-dev openssl-dev gcc libc-dev make
    - pip3 install --no-cache-dir docker-compose
    #############################################
    # Setup the connection to the Docker Server #
    #############################################
    # Prepare SSH
    - mkdir -p ~/.ssh
    - echo "$HOST_KEY" >> ~/.ssh/known_hosts
    - cp "$SSH_KEY" ssh_key
    - chmod 600 ssh_key
    # Tunnel the docker socket through a SSH connection
    - ssh -i ssh_key -nNT -L /docker.sock:/var/run/docker.sock docker@<docker.server> &
    # Configure the docker client to use the tunneled docker socket
    - export DOCKER_HOST=unix:///docker.sock
    # Test it (it can take some time for ssh client to connect)
    - sleep 2
    - docker info
  script:
    #########################################################################
    # Build an up-to-date Docker image with the configuration file included #
    #########################################################################
    - docker-compose build --no-cache --pull
    ##############################
    # Deploy the new built image #
    ##############################
    - docker-compose up --detach
  after_script:
    # Cleanup the runner
    - unset DOCKER_HOST
    - rm -rfv ssh_key /docker.sock ~/.ssh
```

This configuration should be self-explanatory.
For more information take a look at the [GitLab CI Documentation](https://docs.gitlab.com/ce/ci/README.html).

I would also like to point out some improvements, I have done to this in production.

* To speed up the whole setup, I created an image with the prerequisites installed and deployed it on [quay.io]()
* Sometime regular images are used in the `docker-compose.yml` and update them also, I use

  ```yml
  - docker-compose pull
  - docker-compose build --no-cache --pull
  ```

  at the beginning of `scipt:`.
* Configuration files sometimes contain sensible information. If that is the case, I substite them in the config with a placeholder, store the real value in the in the GitLab Variables and replace them in `script:` with something like

  ```yml
  - sed -i "s/SECRET/$SECRET/" configuration-file
  ```

So now everytime we push something to the master, it is automatically deployed on our Docker server.

### Keep everything up to date

It is great to be able to publish changes quickly, but another big problem in small organizations is to keep everything up to date.
Once a service is running nobody cares anymore and it becomes outdated.
You will never get new features and have a potential security problem.

We can circumvent this by using the [GitLab CI/CD Schedules](https://docs.gitlab.com/ce/ci/pipelines/schedules.html), which you can find in your repository under `CI/CD->Schedules`.

So let's create a new schedule

![Schedule](images/infrastructure-with-docker-and-gitlab/schedule.png)


This will run our pipeline once a week and since we pull the latest image everytime the image will be up to date.

Of course this creates a new problem, since the service can break during the update, but in a small organization, I personally rather use up to date software and have to fix it sometimes, than working with old legacy software for a long time.
Also some upstream docker image have some kind of patch level tagging in place with which you can minimize the risc.


## Conclusion

Personally, I think automation is great and reduces workload, especially boring workload, in companies and organization and using Docker and GitLab is pretty simple to use and do not require a lot of overhead.
I uses it for nearly 3 years now and I am pretty happy with it.
