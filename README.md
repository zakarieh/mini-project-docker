# Student List Application

This repository contains a simple application to list students with their ages using a web server (PHP) and an API (Flask).

![Project Image](https://user-images.githubusercontent.com/18481009/84582395-ba230b00-adeb-11ea-9453-22ed1be7e268.jpg)

------------

### Objectives

The goal of this practice exam is to demonstrate your ability to manage Docker infrastructure. You will be evaluated on the following aspects:

### Key Themes:
- Improving an existing application deployment process
- Versioning your infrastructure releases
- Applying best practices when implementing Docker-based infrastructure
- Implementing Infrastructure as Code (IaC)

### Context
  

This guide will help you set up a virtual machine (VM) with CentOS 7.6, install Docker, Docker Compose, and deploy a simple API and PHP website using Docker.

### Prerequisites

- **Vagrant**: A tool for managing virtual machine environments in a consistent workflow.
- **VirtualBox**: A free and open-source hosted hypervisor for running virtual machines.
- **Docker**: A platform to develop, ship, and run applications inside containers.
- **Docker Compose**: A tool for defining and running multi-container Docker applications.

## Application deployment steps with docker

## 1. Create a Virtual Machine with CentOS 7.6 using Vagrant

First, ensure you have both **Vagrant** and **VirtualBox** installed.

### Vagrantfile Setup

The `Vagrantfile` below will configure a virtual machine with CentOS 7.6 and the file [install_docker.sh](https://github.com/diranetafen/cursus-devops/blob/master/vagrant/docker/docker_centos7/install_docker.sh) to install Docker. 
In total, we'll need to forward 4 ports: 2 for the Pozos application and 2 for the private registry.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :
# To enable zsh, please set ENABLE_ZSH env var to "true" before launching vagrant up 
#   + On Windows => $env:ENABLE_ZSH="true"
#   + On Linux  => export ENABLE_ZSH="true"

Vagrant.configure("2") do |config|
  config.vm.define "docker" do |docker|
    docker.vm.box = "eazytrainingfr/centos7"
    docker.vm.box_version = "1.0"

    # Forwarded port for pozos app
    docker.vm.network "forwarded_port", guest: 5000, host: 5000
    docker.vm.network "forwarded_port", guest: 8080, host: 8080

    # Forwarded port for private registry
    docker.vm.network "forwarded_port", guest: 8081, host: 8081
    docker.vm.network "forwarded_port", guest: 8088, host: 8088

    docker.vm.hostname = "docker"
    docker.vm.provider "virtualbox" do |v|
      v.name = "docker"
      v.memory = 1024
      v.cpus = 2
    end
    docker.vm.provision :shell do |shell|
      shell.path = "install_docker.sh"
      shell.env = { 'ENABLE_ZSH' => ENV['ENABLE_ZSH'] }
    end
  end
end
```


### Running Vagrant

1. Initialize the VM:
   ```bash
   vagrant up
   ```
2. SSH into the VM:
   ```bash
   vagrant ssh
   ```


### Install Docker & Docker Compose

*Once everything has gone according to plan, you can connect to the configured VM.* **_if  Docker and Docker Compose is not installed_**, run the following commands to install :

```bash
# Install Docker
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

---

## 2. Clone and Set Up the Project
 

###  Clone the project and set-up the index.php file 

1. Clone the repository containing the PHP website and the simple API:

   ```bash
   git clone https://github.com/diranetafen/student-list/
   cd student-list
   ```

2. Update the PHP `index.php` inside the website file to point to the correct API URL by modifying the `url` and `port` variable:

   ```php
   // In index.php
   $url = 'http://api:5000/pozos/api/v1.0/get_student_ages';
   ```

 
&nbsp;


### **Create Dockerfile for the API**

in the simple_api folder, create a file `Dockerfile` if this is not the case, and format it as follows:

```Dockerfile
# Dockerfile for the API
FROM python:3.8-buster
LABEL maintainer="zakarieh"

# Copy Python script and install dependencies
COPY student_age.py /
RUN DEBIAN_FRONTEND=noninteractive apt update -y && apt install python-dev python3-dev libsasl2-dev python-dev libldap2-dev libssl-dev -y
COPY requirements.txt /requirements.txt
RUN pip3 install -r /requirements.txt

# Create volume and expose port
RUN mkdir /data
VOLUME ["/data"]
EXPOSE 5000

# Run the Python script
CMD ["python3", "./student_age.py"]
```

&nbsp;

### Build and Run the API Container

1. Go to the folder where the Dockerfile is located and Build the Docker image for the API :

   ```bash
   docker build -t api:0.1 . 
   ```

2. Run the API container:

   ```bash
   docker run -d -p 5000:5000 -v ./student_age.json:/data/student_age.json --name api1 api:0.1
   ```

3. Verify the API is running by executing:

   ```bash
   curl -u toto:python -X GET http://localhost:5000/pozos/api/v1.0/get_student_ages
   ```

   Result:

   ```json
   {
     "student_ages": {
       "alice": "12", 
       "bob": "13"
     }
   }
   ```
    ![Capture d’écran du 2024-11-23 22-08-05](https://github.com/user-attachments/assets/44ac240a-aee6-4c8d-9483-b5137d663b8f)

 

&nbsp;


## 3. Set Up Docker Compose 
 

To manage both the API and the website containers, use Docker Compose. Below is the `docker-compose.yml` file:

```yaml
version: "3.8"

services:
  api:
    build:
      context: ./simple_api
      dockerfile: Dockerfile
    container_name: api
    restart: unless-stopped
    ports:
      - "5000:5000"
    volumes:
      - ./simple_api:/data
    networks:
      - student_network
    healthcheck:
      test: ["CMD", "curl", "-u", "toto:python", "-X", "GET", "http://localhost:5000/pozos/api/v1.0/get_student_ages"]
      interval: 30s
      timeout: 10s
      retries: 3
    command: ["python3", "./student_age.py"]

  website:
    image: php:apache
    container_name: website
    restart: unless-stopped
    environment:
      - USERNAME=toto
      - PASSWORD=python
    ports:
      - "8080:80" # Note that Apache uses port 80 (Apache default), so you need to map it to the port of your choice, here 8080 on the host.
    volumes:
      - ./website:/var/www/html
    networks:
      - student_network
    depends_on:
      api:
        condition: service_healthy

networks:
  student_network:
    driver: bridge

```

In these lines in the docker-compose, you can use the image built in the previous steps but i choose to create with the Dockerfile. For example : 

```ruby 
    build:
      context: ./simple_api  # We need to specify the context, which is the directory that contains the Dockerfile
      dockerfile: Dockerfile  #  And mention the Dockerfile here (it's very important)
```

&nbsp; 

### Start the Application with Docker Compose

1. Start all services using Docker Compose:

   ```bash
   docker-compose up -d
   ```

   ![image](https://github.com/user-attachments/assets/739ca01f-fbb3-4e1d-a7e4-5da2fc4776fe)


2. The PHP website should now be accessible at `http://localhost:8080`, and it will communicate with the API at `http://localhost:5000`.

   ![Capture d’écran du 2024-11-23 22-20-56](https://github.com/user-attachments/assets/3d2bf9b1-3c34-4c30-aca6-57ac8248163d) ![Capture d’écran du 2024-11-23 22-20-39](https://github.com/user-attachments/assets/dcc215c3-d080-4f95-bb64-56d6082bf24b)


&nbsp;

## 4. Private registry  

### create a docker-compose to set-up the private registry

In the docker-compose.yml file below, we define the setup for two services: registry-server (Docker Registry) and registry-ui (UI for Docker Registry). Volumes are used to manage persistent data, configurations, and authentication securely.

```yml
version: "3.8"

services:
  # Docker Registry Server - The registry service that stores the Docker images
  registry-server:
    image: registry:2.8.2
    restart: always
    ports:
      - "8081:5000"
    environment:
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin: "[http://registry.example.com]"
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Methods: "[HEAD,GET,OPTIONS,DELETE]"
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Credentials: "[true]"
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Headers: "[Authorization,Accept,Cache-Control]"
      REGISTRY_HTTP_HEADERS_Access-Control-Expose-Headers: "[Docker-Content-Digest]"
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
    volumes:
      - ./registry/data:/var/lib/registry # Data directory
      - ./config.yml:/etc/docker/registry/config.yml # Mount the config file
      - ./htpasswd:/etc/docker/registry/htpasswd # Mount the config file
    container_name: registry-server

    # Docker Registry UI - Frontend web interface for registry
  registry-ui:
    image: joxit/docker-registry-ui:main
    restart: always
    ports:
      - "8088:80" # Exposing port 8088 for the UI to be accessible on the host
    environment:
      - SINGLE_REGISTRY=true # If true, it assumes only one registry is configured
      - REGISTRY_TITLE=Docker Registry UI # The title for the UI
      - DELETE_IMAGES=true # Allows image deletion via UI
      - SHOW_CONTENT_DIGEST=true # Shows digest of content in the UI
      - NGINX_PROXY_PASS_URL=http://registry-server:8081 # The URL of the registry server
      - SHOW_CATALOG_NB_TAGS=true # Show number of tags in the catalog
      - CATALOG_MIN_BRANCHES=1 # Minimum branches for catalog display
      - CATALOG_MAX_BRANCHES=1 # Maximum branches for catalog display
      - TAGLIST_PAGE_SIZE=100 # Number of tags per page in the tag list
      - REGISTRY_SECURED=false # If true, enforces secure HTTPS access
      - CATALOG_ELEMENTS_LIMIT=1000 # Limit for catalog elements
    container_name: registry-ui # Name for the container

```
 
***Volumes in the Docker Registry Setup***

- **`./registry/data:/var/lib/registry`**  
  This volume stores the Docker images and data to persistent.  

- **`./config.yml:/etc/docker/registry/config.yml`**  
  Mounts the custom configuration file for the Docker Registry. This file contains registry-specific settings like of storage registry-server, security, and access controls.

- **`./htpasswd:/etc/docker/registry/htpasswd`**  
   This file contains  usernames and bcrypt-encrypted passwords to authenticate users who need access to the registry. You can create a new one from this website https://bcrypt-generator.com/

&nbsp; 

### Start the Application with Docker Compose

1. Run the two containers using Docker Compose:

   ```bash
   docker-compose up -d
   ```
   
   ![Capture d’écran du 2024-11-23 23-11-47](https://github.com/user-attachments/assets/f98e9b82-b636-4608-abad-d972e157c04f)

2. Tag and push the existing images :

`docker push ip:8088/php:apache`

`docker tag id_image ip:8088/php:apache`

3. Visualation of Registry UI 
 
 ![Capture d’écran du 2024-11-23 23-18-32](https://github.com/user-attachments/assets/77102bda-ba96-462f-9cc6-849b387a9db4)


## **Conclusion**

This project was an excellent opportunity to apply Docker in a real-world deployment scenario. By containerizing a student listing application and creating a private Docker registry, I was able to enhance my knowledge of Docker Compose, container management, and infrastructure best practices. The challenges in setting up multiple services, managing dependencies, and versioning the infrastructure through Docker have significantly improved my understanding of Docker-based application deployment.

**Author**

[@zakarieh](https://github.com/zakarieh)
