Deploying OpenWhisk using Ansible
=========


### Getting started

If you want to deploy OpenWhisk locally using Ansible, you first need to install Ansible on your development environment:

#### Ubuntu users
```
sudo apt-get install python-pip
sudo pip install ansible==2.4.2.0
sudo pip install jinja2==2.9.6
```

#### Vagrant users
Nothing to be done, Ansible is already installed during vagrant provisioning.
You can skip setup and prereq steps as those have been done by vagrant for you.  
You may jump directly to [Deploying Using CouchDB](#deploying-using-couchdb)

#### Docker for Mac users
```
sudo easy_install pip
sudo pip install ansible==2.4.2.0
pip install jinja2==2.9.6
```
Docker for Mac does not provide any official ways to meet some requirements for OpenWhisk.
You need to depend on the workarounds until Docker provides official methods.

If you prefer [Docker-machine](https://docs.docker.com/machine/) to [Docker for mac](https://docs.docker.com/docker-for-mac/), you can follow instructions in [docker-machine/README.md](../tools/macos/docker-machine/README.md).

##### Enable Docker remote API
The remote Docker API is required for collecting logs using the Ansible playbook [logs.yml](logs.yml).

##### Activate docker0 network
This is an optional step for local deployment.
The OpenWhisk deployment via Ansible uses the `docker0` network interface to deploy OpenWhisk and it does not exist on Docker for Mac environment.

An expedient workaround is to add alias for `docker0` network to loopback interface.

```
sudo ifconfig lo0 alias 172.17.0.1/24
```

### Using Ansible
**Caveat:** All Ansible commands are meant to be executed from the `ansible` directory.
This is important because that's where `ansible.cfg` is located which contains generic settings that are needed for the remaining steps.

In all instructions, replace `<environment>` with your target environment. The default environment is `local` which works for Ubuntu and
Docker for Mac. To use the default environment, you may omit the `-i` parameter entirely. For older Mac installation using Docker Machine,
use `-i environments/docker-machine`.



In all instructions, replace `<openwhisk_home>` with the base directory of your OpenWhisk source tree. e.g. `openwhisk`

#### Setup

The following step must be executed once per development environment.
It will generate the `hosts` configuration file based on your environment settings.

```
ansible-playbook -i environments/<environment> setup.yml
```

The default configuration does not run multiple instances of core components (e.g., controller, invoker, kafka).
You may elect to enable high-availability (HA) mode by passing tne ansible option `-e mode=HA` when executing this playbook.
This will configure your deployment with multiple instances (e.g., two kafka instancess, and two invokers).

In addition to the host file generation, you need to configure the database for your deployment. This is done
by creating a file `ansible/db_local.ini` to provide the following properties.

```bash
[db_creds]
db_provider=
db_username=
db_password=
db_protocol=
db_host=
db_port=
```

This file is generated automatically if you are using an ephermeral CouchDB instance. Otherwise, you must create it explicitly.
For convenience, you can use shell environment variables that are read by the playbook to generate the required `db_local.ini` file as shown below.

```
export OW_DB=CouchDB
export OW_DB_USERNAME=<your couchdb user>
export OW_DB_PASSWORD=<your couchdb password>
export OW_DB_PROTOCOL=<your couchdb protocol>
export OW_DB_HOST=<your couchdb host>
export OW_DB_PORT=<your couchdb port>

ansible-playbook -i environments/<environment> couchdb.yml --tags ini
```

Alternatively, if you want to use Cloudant as your datastore:

```
export OW_DB=Cloudant
export OW_DB_USERNAME=<your cloudant user>
export OW_DB_PASSWORD=<your cloudant password>
export OW_DB_PROTOCOL=https
export OW_DB_HOST=<your cloudant user>.cloudant.com
export OW_DB_PORT=443

ansible-playbook -i environments/<environment> couchdb.yml --tags ini
```

#### Install Prerequisites
This step is not required for local environments since all prerequisites are already installed, and therefore may be skipped.`

This step needs to be done only once per target environment. It will install necessary prerequisites on all target hosts in the environment.

```
ansible-playbook -i environments/<environment> prereq.yml
```

**Hint:** During playbook execution the `TASK [prereq : check for pip]` can show as failed. This is normal if no pip is installed. The playbook will then move on and install pip on the target machines.

### Deploying Using CouchDB
- Make sure your `db_local.ini` file is [setup for](#setup) CouchDB then execute:

```
cd <openwhisk_home>
./gradlew distDocker
cd ansible
ansible-playbook -i environments/<environment> couchdb.yml
ansible-playbook -i environments/<environment> initdb.yml
ansible-playbook -i environments/<environment> wipe.yml
ansible-playbook -i environments/<environment> apigateway.yml
ansible-playbook -i environments/<environment> openwhisk.yml
ansible-playbook -i environments/<environment> postdeploy.yml
```

You need to run `initdb.yml` **every time** you do a fresh deploy CouchDB to initialize the subjects database.
The playbooks `wipe.yml` and `postdeploy.yml` should be run on a fresh deployment only, otherwise all transient
data that include actions and activations are lost.

#### Limitation

You can not run multiple CouchDB nodes on a single machine.
This limitation comes from Erlang EPMD.
When CouchDB forms a cluster, it counts on EPMD to find other nodes.
If we want to run multiple nodes on a single machine, we must differentiate EPMD port(`4369`) for each nodes. But if this port is different on each nodes, they cannot find each other.
So if you want to deploy multiple CouchDB nodes, all nodes should be placed on different machines respectively.


### Deploying Using Cloudant
- Make sure your `db_local.ini` file is set up for Cloudant. See [Setup](#setup)
- Then execute

```
cd <openwhisk_home>
./gradlew distDocker
cd ansible
ansible-playbook -i environments/<environment> initdb.yml
ansible-playbook -i environments/<environment> wipe.yml
ansible-playbook -i environments/<environment> apigateway.yml
ansible-playbook -i environments/<environment> openwhisk.yml
ansible-playbook -i environments/<environment> postdeploy.yml
```

You need to run `initdb` on Cloudant **only once** per Cloudant database to initialize the subjects database.
The `initdb.yml` playbook will only initialize your database if it is not initialized already, else it will skip initialization steps.

The playbooks `wipe.yml` and `postdeploy.yml` should be run on a fresh deployment only, otherwise all transient
data that include actions and activations are lost.

Use `ansible-playbook -i environments/<environment> openwhisk.yml` to avoid wiping the data store. This is useful to start OpenWhisk after restarting your Operating System.

### Configuring the installation of `wsk` CLI
There are two installation modes to install `wsk` CLI: remote and local.

The mode "remote" means to download the `wsk` binaries from available web links. By default, OpenWhisk sets
the installation mode to remote and downloads the binaries from the CLI [release page](https://github.com/apache/incubator-openwhisk-cli/releases), where OpenWhisk publishes the official `wsk` binaries.

The mode "local" means to build and install the `wsk` binaries from local CLI project. You can download the source code
of OpenWhisk CLI via [this link](https://github.com/apache/incubator-openwhisk-cli). Let's assume your OpenWhisk CLI home directory is <openwhisk_cli_home>. After you download the source code, use the gradle command to build the binaries:

```
cd <openwhisk_cli_home>
./gradlew buildBinaries
```

All the binaries are generated and put under the folder of <openwhisk_cli_home>/bin. Then, use the following ansible command to configure the CLI installation mode:

```
ansible-playbook -i environments/<environment> openwhisk.yml -e cli_installation_mode=local -e openwhisk_cli_home=<openwhisk_cli_home>
```

The parameter cli_installation_mode specifies the CLI installation mode and the parameter openwhisk_cli_home specifies the home directory of your local OpenWhisk CLI.

### Hot-swapping a Single Component
The playbook structure allows you to clean, deploy or re-deploy a single component as well as the entire OpenWhisk stack. Let's assume you have deployed the entire stack using the `openwhisk.yml` playbook. You then make a change to a single component, for example the invoker. You will probably want a new tag on the invoker image so you first build it using:

```
cd <openwhisk_home>
gradle :core:invoker:distDocker -PdockerImageTag=myNewInvoker
```
Then all you need to do is re-deploy the invoker using the new image:

```
cd ansible
ansible-playbook -i environments/<environment> invoker.yml -e docker_image_tag=myNewInvoker
```

**Hint:** You can omit the Docker image tag parameters in which case `latest` will be used implicitly.

### Cleaning a Single Component
You can remove a single component just as you would remove the entire deployment stack.
For example, if you wanted to remove only the controller you would run:

```
cd ansible
ansible-playbook -i environments/<environment> controller.yml -e mode=clean
```

**Caveat:** In distributed environments some components (e.g. Invoker, etc.) exist on multiple machines. So if you run a playbook to clean or deploy those components, it will run on **all** of the hosts targeted by the component's playbook.


### Cleaning an OpenWhisk Deployment
Once you are done with the deployment you can clean it from the target environment.

```
ansible-playbook -i environments/<environment> openwhisk.yml -e mode=clean
```

### Removing all prereqs from an environment
This is usually not necessary, however in case you want to uninstall all prereqs from a target environment, execute:

```
ansible-playbook -i environments/<environment> prereq.yml -e mode=clean
```


### Troubleshooting
Some of the more common problems and their solution are listed here.

#### Setuptools Version Mismatch
If you encounter the following error message during `ansible` execution

```
ERROR! Unexpected Exception: ... Requirement.parse('setuptools>=11.3'))
```

your `setuptools` package is likely out of date. You can upgrade the package using this command:

```
pip install --upgrade setuptools --user python
```


#### Mac Setup - Python Interpreter
The MacOS environment assumes Python is installed in `/usr/local/bin` which is the default location when using `brew`.
The following error will occur if Python is located elsewhere:

```
ansible all -i environments/mac -m ping
ansible | FAILED! => {
    "changed": false,
    "failed": true,
    "module_stderr": "/bin/sh: /usr/local/bin/python: No such file or directory\n",
    "module_stdout": "",
    "msg": "MODULE FAILURE",
    "parsed": false
}
```

An expedient workaround is to create a link to the expected location:

```
ln -s $(which python) /usr/local/bin/python
```

Alternatively, you can also configure the location of Python interpreter in `environments/<environment>/group_vars`.

```
ansible_python_interpreter: "/usr/local/bin/python"
```

#### Failed to import docker-py

After `brew install ansible`, the following lines are printed out:

```
==> Caveats
If you need Python to find the installed site-packages:
  mkdir -p ~/Library/Python/2.7/lib/python/site-packages
  echo '/usr/local/lib/python2.7/site-packages' > ~/Library/Python/2.7/lib/python/site-packages/homebrew.pth
```
  
Just run the two commands to fix this issue.

#### Spaces in Paths
Ansible 2.1.0.0 and earlier versions do not support a space in file paths.
Many file imports and roles will not work correctly when included from a path that contains spaces.
If you encounter this error message during Ansible execution

```
fatal: [ansible]: FAILED! => {"failed": true, "msg": "need more than 1 value to unpack"}
```

the path to your OpenWhisk `ansible` directory contains spaces. To fix this, please copy the source tree to a path
without spaces as there is no current fix available to this problem.

#### Changing limits
The default system throttling limits are configured in this file [./group_vars/all](./group_vars/all).
```
limits:
  invocationsPerMinute: "{{ limit_invocations_per_minute | default(120) }}"
  concurrentInvocations: "{{ limit_invocations_concurrent | default(100) }}"
  concurrentInvocationsSystem:  "{{ limit_invocations_concurrent_system | default(5000) }}"
  firesPerMinute: "{{ limit_fires_per_minute | default(60) }}"
  sequenceMaxLength: "{{ limit_sequence_max_length | default(50) }}"
```
These values may be changed by modifying the `group_vars` for your environment. For example,
mac users will find the limits in this file [./environments/mac/group_vars/all](./environments/mac/group_vars/all):
```
limit_invocations_per_minute: 60
limit_invocations_concurrent: 30
limit_invocations_concurrent_system: 5000
limit_fires_per_minute: 60
```
- The `limit_invocations_per_minute` represents the allowed namespace action invocations per minute.
- The `limit_invocations_concurrent` represents the maximum concurrent invocations allowed per namespace.
- The `limit_invocations_concurrent_system` represents the maximum concurrent invocations the system will allow across all namespaces.
- The `limit_fires_per_minute` represents the allowed namespace trigger firings per minute.
