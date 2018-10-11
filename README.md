# Downtimeless deployment process of a tiny application

This is a concept of the deployment procedure which guarantees the app will be available during its release process. Used technologies: Linux (CentOS7), Docker + Swarm, Ansible, Vagrant (to simplify the demo).

## Getting Started

With these instructions you will be able to use the copy of the project on your local machine for development and testing purposes. See **Deployment** section for notes on how to roll out the project on a live system.

### Prerequisites

In order to make everyone's life easier, the Vagrant VM image is appropriate to be installed on any system with VirtualBox and Vagrant. The mentioned box can be built using prepared Vagrantfile available by the link provided at **Installing** section. You may also take some profit from Vagrantfile placed right below this README file to create suitable VM using the Vagrantfile's content.

Anyway, wherever you want to try the suggested approach, Ansible package must be installed. Visit [Ansible Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for details.
In addition, the requirements below you will need on the host that executes kickstart/deployment playbooks:
- python >= 2.7
- docker (python module)
- git

**NOTE:** The playbooks are configured and tested on CentOS system. The (yum)[https://docs.ansible.com/ansible/latest/modules/yum_module.html] module and probably some others used in the playbooks, won't work on Debian-like and not RPM-based systems.

### Installing

1. Install [VirualBox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.vagrantup.com/downloads.html) on your system. Download [Vagrantfile](https://raw.githubusercontent.com/fesia/here_we_deploy/master/Vagrantfile) to some convenient place. Open up the terminal (Winkey+R -> cmd.exe on Windows systems), change directory to the one with downloaded Vagrantfile and build the VM:
```
> cd /path/to/vm_workdir
> vagrant up
```
2. In the same directory connect to the built VM by typing **vagrant ssh** and launch Ansible 'release_new_app_version.yml' playbook with the first app version:
```
> vagrant ssh
$ cd ./here_we_deploy
$ sudo ansible-playbook ansible-playbook ./ansible/plays/release_new_app_version.yml -e 'release_version=v0.1'
```

**NOTE:** 'v0.1' version is actually a tag at [test_app](https://github.com/fesia/test_app/tags) repo.

**That is it!** You may ascertain that your installation works by visiting page http://localhost:10080/ in your browser.

### What have we just done?
The built Vagrant box is based on CentOS 7 and all required dependencies were installed inside of it by `config.vm.provision` section at Vagrantfile.

The initial app version has been deployed in the VM by Ansible using the appropriate playbook. That playbook sets up the application calling 'deploy_app' Ansible role which contains all necessary instructions, files, and variables.

### Structure and relationship overview
#### Ansible part
'deploy_app' role directory tree:
```
.
├── defaults
│   └── main.yml
├── tasks
│   ├── deploy_docker_stack.yml
│   ├── main.yml
│   └── swarm_init.yml
├── templates
│   ├── docker_compose.yml
│   └── Dockerfile
└── vars
    ├── docker_compose_content.yml (wasn't currently used)
    └── main.yml
```

'deploy_app' role playbook calls sequence:
```
../../plays | release_new_app_version.yml <--.
            |              \                  \
defaults    |  main.yml    /             'release_version' (this extra var is passed by an engineer;
vars        |   /       main.yml                                               'latest' is the default value)
            |   \          \ 
tasks       |    `-----> main.yml
            |               ^
            |                \-- swarm_init.yml
            |                 `-- deploy_docker_stack.yml
```

#### Docker part
Docker images:
- afesenko/nginx_8080
- dockercloud/haproxy

HAProxy is used for load balancing between app containers with a web-server (in our case it's Nginx). Nginx is used for fetching app's content (index.html). This is the custom Nginx image with 8080 port configured to be listened instead of 80. These both images are pulled in the kick-off stage while provisioning the VM.

'mega_app' image is built at the deployment stage whatever the app version has been provided. It is created 'FROM' afesenko/nginx_8080 image with copying app content further. The containers built out of this image should always(!) be properly tagged.

Docker Swarm must be initialized in order to let us using `docker stack` feature. Our docker stack includes 12 replicas for providing app's fault tolerance and **downtimeless** deployment of its new versions. The replicas quantity is configured at docker-compose.yml file. As it was already mentioned, the requests from the clients are balanced between 'mega_app' replicas by HAProxy. All connections are processed by 'web' overlay-driven network.

## Deployment
If you've logged out of the VM, please log in back:
```
> vagrant ssh
```
Launch the deployment process with the new (v0.2) version of the application:
```
$ cd ./here_we_deploy
$ sudo ansible-playbook ./ansible/plays/release_new_app_version.yml -e 'release_version=v0.2'
```

That's all! Ansible will do the rest needed actions. You may want proceed with testing instantly.

**NOTE:** The important use case is that you can do the **rollback** in the same simple way - passing any of the previous app versions (try 'v0.1').

### Testing
#### First approach
Just start spamming 'F5' key being at the http://localhost:10080/ page in your browser.

#### The second one
The parallel terminal session is required. You may use our running VM or your host system.

In case you've decided to use our VM, please connect to it:
```
> cd /path/to/vm_workdir
> vagrant ssh
```
... and execute the following thing in the second terminal:
```
$ while true; do curl -s http://$(hostname -I | awk '{ print $1 }') | grep -o "This is app version [0-9]*\.[0-9]*"; sleep 2; done
```
Just put the correct IP/hostname and port in case you do this test on your workstation.

**NOTE:** The 'while' cycle above should be working simultaneously with the new app release deployment.

You will see the app's content (by what the app version is being displayed) every 2 seconds.

### How does it work?
From your workstation the application is available through the VirtualBox' NAT network: every request is proxied through 10080 host's port to the 80's on the VM.

We got rid of any downtime while deploying the new version of our application by using the 'parallelism' option. Once we built the new 'mega_app' image, the tag is applied to it automatically by Ansible (docker_image)[https://docs.ansible.com/ansible/latest/modules/docker_image_module.html] module. The updated docker-compose.yml file contains the new app tag as well. While deploying the new app version Docker (on Ansible's request) replaces old 'mega_app' containers with the new ones by batches of 4 one-by-one (here we have 'parallelism' set to 4 by default) **not** killing the whole set of the old containers before the new installation.

While the deployment process user(s) might sometimes get the old app's code and (probably) feel some degraded performance but the application will **never** become totally unreachable to them.


## Authors
**Andrii Fesenko** - *Initial work* - <andrii.iv.fesenko@gmail.com>
