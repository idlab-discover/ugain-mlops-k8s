# Docker containers introduction

In this introduction you will learn some basic docker commands that are good to master. They will help you complete the coming lab sessions. They will also give you a primer into this technology, which will certainly be useful in the future.

## Terminology

A docker image is the template from which a container can be created. A container is the thing you start and stop and that runs your (or someone else's) program. You can download and use (read: pull) docker images that are hosted on repositories, both private and public. The [Docker Hub](https://hub.docker.com) is the public repository hosted by Docker (the company).

One image can have multiple versions, they are marked with tags. Images are usually defined in the format `[repository*]/[ownerPath*]/[imageName]:[tag]`. (repository is optional and defaults to Docker Hub; some images do not have an actual owner thus the path turns into `_`). Here are some examples:
- [`python:3`](https://hub.docker.com/_/python)
- [`ubuntu:20.04`](https://hub.docker.com/_/ubuntu)
- [`grafana/grafana:10.1.4`](https://hub.docker.com/r/grafana/grafana)
- `gcr.io/google-containers/cassandra:v7`

Some images on Docker Hub don't have an ownerPath (or just `_`), this is because they are [Docker Official Images](https://docs.docker.com/docker-hub/official_images/), and are hosted on the root level.

Tags are optional; when no tag is present, it defaults to requesting the `latest` tag. However, the repository must host an explicit `latest` tag for that to work.

> :information_source: It is considered good practice to always pinpoint a specific version by using the tag. This way you are sure you always work with the exact same version of the image in between deployments, regardless of any (upstream) updates to the `latest` tag.

It is also possible to build your own image and publish it on Docker Hub. For that you need to create a Dockerfile which is basically a text file that contains instructions on how to create the image. Images in Docker are a layer-based system. Layers are overlaid/superimposed on top of other layers and are also internally stored that way. This allows for instance to create five different images, that all use the same base layer (e.g. Ubuntu 20.04). This in turn would mean we only need to download this base layer part once for all five images.

*Dockerfile* example:
```docker
FROM python:3

# set a directory for the app
WORKDIR /usr/src/app

# copy all the files from the current directory to the container
COPY . .

# install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# define the port number the container should expose
EXPOSE 5000

# run the command
CMD ["python", "./app.py"]
```

While Docker containers typically run services, a Docker volume can be viewed as a storage container. If you start a container without a volume attached to it, all written data will be gone once you restart that container. If it is linked with a volume, that data is persisted throughout restarts. Volumes allow you to bind a local directory to an internal path of the container, thus writing and reading from your local system path instead.

## Docker daemon and CLI commands

To execute Docker containers, build images or just interact with running containers, a service is required. This is the Docker Daemon. You will first have to install the docker daemon on your pc/laptop to start working. To do this please download and install the latest version of [Docker Desktop](https://www.docker.com/get-started) (Windows/Mac/Linux).

We will also need a terminal to issue CLI commands. On Windows you can either use PowerShell or the Windows Subsystem for Linux (WSL). When using WSL be mindful that you are operating in a virtualized environment and that tooling has to be either installed in WSL or be linked to the WSL instance somehow. Docker Desktop has a setting to enable WSL integration, which will allow you to use the `docker` commands in WSL. 

Instructions to setup WSL and an Ubuntu distribution [can be found here](https://documentation.ubuntu.com/wsl/en/latest/guides/install-ubuntu-wsl2/).

> :warning: On Windows: do not use `cmd.exe` as this terminal is very limiting in its capabilities and some commands listed here will not work. If you so prefer, you can also use `powershell`, but just remember that you will have to use `bash` later on in the course, as it is the default shell for the development workspace.

If you want to do anything with Docker, you will have to issue CLI commands to the Docker daemon. These commands are prefixed with `docker`. Below are some basic commands, but you can get a full listing by issuing the `docker help` command.

| Command      | Description |
| ----------- | ----------- |
| ps      | List containers |
| info   | Display system-wide information |
| help | Displays extensive information on all possible commands |
| start | Start an existing container |
| stop | Stop an existing container |
| run | Run a command in a new container |
| pull | Pull an image from a registry |
| push | Push an image to a registry |
| rm | Remove a container from your local daemon |
| rmi | Remove an image from your local daemon |

For more information on the use of these commands, type `docker COMMAND --help`.

> :warning: Windows: if you can't access the `docker` commands on WSL, then be sure to check Docker Desktop settings! Go to `Settings > Resources > WSL integration` and make sure `Enable integration with my default distro` is turned on. If you have multiple distros you can also select for which you want to enable the integration.

## Monitoring system

*From here on this will be a guided experience that we highly suggest you try to duplicate on your own system. It will give you a basic but good understanding of some of the aspects of docker.*

### Preparation

> :warning: We highly recommend working in a directory that is not administrator-protected. For instance create the folder C:\containers and cd into that folder before continuing with this lab!

Once you've picked a directory to work in (we will refer to that as the root directory), you will have to create a single configuration file. This configuration is used for Prometheus, a time series database used mostly in monitoring systems, which we will set up later.

Create a file named `prometheus.yml` with the following content in your root directory:
```yaml
# my global config
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  # scrape_timeout is set to the global default (10s).

scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

Now you are all set to start with the next section, keep working in this root directory.

> :heavy_exclamation_mark: For this lab you should be located in your root directory. This is the folder which contains the prometheus.yml file!

### Overview

To showcase the power of Docker we will set up a quick monitoring system. It will use 4 services:
- **Prometheus Node exporter**: collects and exposes hardware and OS metrics
- **Prometheus Container exporter**: collects and exposes container metrics
- **Prometheus**: a database that stores time series metrics
- **Grafana**: a dashboard solution that allows us to quickly visualize all captured metrics

**Prometheus Node exporter** will give us a lot of information about the underlying system. **Prometheus container exporter** will give us more information about running Docker containers. All these metrics are stored in the **Prometheus** database. **Grafana** will be our tool to visualize all of this.

Installing and configuring these 4 services would take quite some time if we were to do this by following the classic installation instructions for each service. In addition our local machines would be polluted with config files, install directories and log files from said applications. By deploying these applications using containers and Docker, setup and cleanup are both quicker and more efficient.

### Setup

We will now set up this system by bringing up 4 individual Docker containers and a virtual network. This will help you understand how these Docker containers work individually and communicate with each other.

1. **User-defined network**

First, we will create a user-defined network in Docker. This is a virtual network that will allow us to communicate between connected containers by name. It will also allow access to all ports of these containers, without having to expose them to other docker services, not part of the virtual network. The default network type is a *bridge network*, this will make sure the host computer is always part of any user-defined (bridge) network. Create a network with this command:

```
docker network create my_network
```
If you want to know more about the different network types: [docker network drivers](https://docs.docker.com/network/#network-drivers)

2. **Prometheus Node exporter**

Use the following command to start a Node exporter container from its image. The Prometheus Node Exporter Docker image can be found on [Docker Hub](https://hub.docker.com/r/prom/node-exporter). We will use the Docker CLI command to tell the Docker Daemon that we want to run a container made from that image. The Docker Daemon will *pull* (read: download) that image onto our system, if we don't have it yet.
```
docker run -d --name node-exporter --network my_network prom/node-exporter:v1.8.2
```
- `-d` Makes it run in detached mode, meaning it runs in the background.
- `--name [name]` Give the container a name that can be used as a handle.
- `--network [name]` Connect to the network with the given name.

> :information_source: Remember the syntax of a docker image: prom is the ownerPath, node-exporter the imageName, and v1.8.2 the tag. The repository part is empty, thus defaults to Docker Hub.

3. **Google cAdvisor**

To scrape some docker container metrics, we will also install cAdvisor. This docker image requires more complicated arguments to run, but you can just copy paste the command. The cAdvisor image can also be found on [GitHub](https://github.com/google/cadvisor).
```
docker run -v /:/rootfs:ro -v /var/run:/var/run:rw -v /sys:/sys:ro -v /var/lib/docker/:/var/lib/docker:ro --network my_network -d --name=cadvisor gcr.io/cadvisor/cadvisor:v0.50.0
```
- `-d` Makes it run in detached mode, meaning it runs in the background.
- `--name [name]` Give the container a name that can be used as a handle.
- `--network [name]` Connect to the network with the given name.
- `-v [localPath]:[remotePath]:[mode]` Mount a local path onto a remote path (as explained above with *docker volumes*).

cAdvisor collects metrics of running containers. We mount several system files (see `-v` options) from which cAdvisor gathers the required container metrics.

4. **Prometheus**

Next, we will set up the Prometheus instance. The Prometheus [Docker image](https://hub.docker.com/r/prom/prometheus) can be found on Docker Hub. Again, use the following Docker cli command:
```
docker run -d --name prometheus -p 9090:9090 --network my_network -v "$(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml" prom/prometheus:v2.54.1
```
- `-d` Makes it run in detached mode, meaning it runs in the background.
- `--name [name]` Give the container a name that can be used as a handle.
- `-p [hostPort]:[containerPort]` Binds the hostPort to the containerPort (making it accessible outside the container, e.g. on localhost:hostPort). You can check this by browsing to [port 9090 on your localhost](http://localhost:9090/) (it should respond with a small webpage hosted by Prometheus)
- `--network` Connect to the network with the given name.
- `-v [localPath]:[remotePath]:[mode]` Mount a local path into a remote path (as explained above with docker volumes). Here we use this to mount our prometheus config file onto the expected config path of the container.

5. **Grafana**

Our final piece of the setup is Grafana. This is a dashboard solution that allows us to connect with many different data sources, Prometheus being one of them. We will use it to graph our data in a simple monitoring dashboard.

```
docker run -d --name grafana -p 3000:3000 --network my_network grafana/grafana:11.2.1
```
- `-d` Makes it run in detached mode, meaning it runs in the background.
- `--name [name]` Give the container a name that can be used as a handle.
- `-p [hostPort]:[containerPort]` Binds the hostPort to the containerPort (making it accessible outside the container).
- `--network [name]` Connect to the network with the given name.

**Configuring Grafana**
- Browse to [port 3000 on your localhost](http://localhost:3000/).
- Use username `admin` and password `admin` to log in.
- Optionally change the admin password now or press `skip` and do it later

**Adding Prometheus as a Data source**
- Toggle the sidebar Menu > **Connections** > **Data Sources**
- Click the **Add data source** button.
- Select Prometheus from the list.
- Under **Connection**, fill in the **Prometheus server URL** as follows: `http://prometheus:9090` (this works thanks to the created my_network and the given container name).
- Leave all other settings at default values.
- Click the **Save & Test** button below. You should get a green alert that the *Data source* is working.

**Importing the monitoring dashboard**
- Open from the sidebar Menu: **Dashboards**.
- Click the dropdown button **New** > **Import**.
- In the input box Grafana.com dashboard URL or ID fill in the following id: `17041` and click **Load**. *This will download our dashboard from [grafana.com](https://grafana.com/grafana/dashboards/17041-system-docker-monitoring/)*.
- Now just add our Prometheus instance in the Prometheus selection box and click **Import**.

That's it, you now have a dashboard connected to our metrics in Prometheus.

## Docker Compose

In the previous step you've manually set up 4 containers and a network, and made them work together. While this is definitely an improvement over regular OS installs of these services, it still is quite a bit of work setting these up (and tearing them down).

This is where **Docker Compose** comes in. It allows us to write a simple *docker-compose.yml* (YAML) file that describes each of our services. Once this file is in place, you can just execute `docker compose up` and all the services will start. If you want to bring them down, a simple `docker compose down` is all it takes. We will now do this for our 4 services, but first make sure all running containers are stopped.

> :information_source: Use `docker ps -a` to list all existing containers and `docker rm -f [container]` to force delete the containers (even if they are running). If you do not delete them, Docker will complain that containers with the same name already exist. We can also remove the created network with `docker network rm [network]`.

**Setup: creating docker-compose.yml**

> :heavy_exclamation_mark: Again, make sure you are located in the root directory (the one containing the prometheus.yml file) of the repository!

Add an empty `docker-compose.yml` file next to the `prometheus.yml` file. We will add to it section by section and explain what each line does.

> :warning: Indentation is important in yaml files. Like python, wrong indentation changes the meaning of the file.

First add the services section where we will define our 4 services.

```yaml
services:
```

> :information_source: In the previous setup we defined a docker network called `my_network` to be used by our applications. When using docker-compose we don't need to define this network explicitly. By default Docker Compose sets up a single network for your app. Each container for a service joins the default network and is both reachable by other containers on that network, and discoverable by them at a hostname identical to the service name.

**Service 1: Node Exporter**

Now let's define the first service: node_exporter.

```yaml
services:
  node-exporter:
    image: prom/node-exporter:v1.8.2
```

- `image`: the Docker image to use.

**Service 2: cAdvisor**

Adding cAdvisor will be similar, but now we have one more command to express in the docker-compose file: volume mounts.
```yaml
services:
  node-exporter:
    image: prom/node-exporter:v1.8.2

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.50.0
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
```

- `image`: the Docker image to use.
- `volumes`: array defining all volume mounts.

**Service 3: Prometheus**

Adding Prometheus is very similar, but it adds the concept of port bindings. This is needed to be able to bind the [9090 port to our localhost](http://localhost:9090/) (for when you want to interact with Prometheus from its interface).

```yaml
services:
  node-exporter:
    image: prom/node-exporter:v1.8.2

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.50.0
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  prometheus:
    image: prom/prometheus:v2.54.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
```

- `image`: the Docker image to use.
- `volumes`: array defining all volume mounts.
- `ports`: array of port mappings. (`[hostPort]:[containerPort]`)

**Service 4: Grafana**

The last service to add is Grafana, and this should speak for itself by now.

```yaml
 services: 
  node-exporter:
    image: prom/node-exporter:v1.8.2

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.50.0
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  prometheus:
    image: prom/prometheus:v2.54.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090

  grafana:
    image: grafana/grafana:11.2.1
    ports:
      - 3000:3000
```

**Starting and stopping**

So now we got all services defined, this is how you start everything:

```
docker compose up -d
```

This will start all services and create the network if it did not exist. If the images are not downloaded yet, they will be pulled first. The `-d` trigger makes them all run detached.

If you want to inspect the log output of a service, for instance Prometheus, try this:

```
docker compose logs prometheus
```

Use the optional `-f` trigger to tail the log. (`docker-compose logs -f prometheus`, `ctrl+c` to break out again).

If you want to bring everything down, use:

```
docker compose down
```

**Data Persistence with Volumes**

You may notice that when you tear down the monitoring setup, we lose all data: gathered metrics, datasource setup, custom admin password, dashboard import, etc.

In order to preserve the data between restarts we can make one final addition to the compose file: [volumes](https://docs.docker.com/storage/volumes/).

At the bottom of your compose file add this entry

```yaml
volumes:
  prometheus-data:
  grafana-data:
```

Here we define two new volumes which we can bind to our grafana and prometheus container by adding an entry to their service volumes. We are going:

- Bind `prometheus-data` to `/prometheus` path of the Prometheus container.
- Bind `grafana-data` to `/var/lib/grafana` of the Grafana container.

```yaml
  prometheus:
    image: prom/prometheus:v2.54.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - 9090:9090

  grafana:
    image: grafana/grafana:11.2.1
    ports:
      - 3000:3000
    volumes:
      - grafana-data:/var/lib/grafana
```

Now when we bring our services up, we can see them running with `docker ps` and we can also see our volumes through `docker volume ls` (we can also inspect all this through the Docker Desktop GUI).

Now when we bring our services down or restart them, you'll see that data gets persisted. You can now easily start building out your docker monitoring deployment and start and stop it when needed, without having to re-import dashboards/settings/etc.

*Optionally if you do want to wipe your data, you can use `docker compose down -v` , this will delete all named volumes (and thus clear your Prometheus database and delete your Grafana configuration and dashboards). If you are curious to learn more about (named) volumes: [docker volumes](https://docs.docker.com/storage/volumes/)*

**Summary**
With that you should now have a general understanding of Docker and Docker Compose. You can easily see how you can quickly set up different services on your development machine using Docker Compose and bring them down without any effort. This is very useful in many cases:

- Faster installation of services when trying out new technologies.
- Clean OS after uninstalling services.
- Easy packaging and sharing of applications/services that you've made.
- Quick spin up of services for testing in a build pipeline.

Final Docker Compose file

```yaml
services: 
  node-exporter:
    image: prom/node-exporter:v1.8.2

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.50.0
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  prometheus:
    image: prom/prometheus:v2.54.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - 9090:9090

  grafana:
    image: grafana/grafana:11.2.1
    ports:
      - 3000:3000
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  prometheus-data:
  grafana-data:
```

## Documentation links

- [Docker CLI](https://docs.docker.com/engine/reference/commandline/cli/)
- [Docker Compose CLI](https://docs.docker.com/reference/cli/docker/compose/)
- [Dockerfile syntax](https://docs.docker.com/engine/reference/builder/)
- [Docker Compose syntax](https://docs.docker.com/compose/compose-file/)
