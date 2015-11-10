
This repository contains:

* Sync Gateway + Couchbase Server Cluster setup scripts suitable for:
    * Functional Tests
    * Performance Tests
* Functional Test Suite (python)

## Setup Controller

The "controller" is the machine that runs ansible, which is typically:

* Your developer workstation
* A virtual machine / docker container

The instructions below are docker specific, but if you look in `docker/controller/Dockerfile` it should give you an idea of the required dependencies if you want to make this work directly on your workstation.

### Start a container

```shell
$ docker run -ti tleyden5iwx/sync-gateway-tests /bin/bash
```

The rest of the commands should be run **inside** the container created in the previous step.

### Clone Repo

```
$ cd /opt
$ git clone https://github.com/couchbaselabs/sync-gateway-testcluster.git
```

### Setup Global Ansible Config

```
$ cd sync-gateway-testcluster/provision/ansible/playbooks
$ cp ansible.cfg.example ansible.cfg
$ vi ansible.cfg  # edit to your liking
```

By default, the user is set to `root`, which works for VM clusters.  If you are on AWS, you will need to change that to `centos`


### Copy SSH key from Host -> Container

In order to be able to ssh from your container into any of the hosts on AWS using public key auth, which is required to run most of the ansible commands, you will need to have an ssh keypair in your container that corresponds to your registered AWS key.


## Setup Cluster

### Spin up VM's or Bare Metal machines

Requirements:

* Should have a centos user with full root access in /etc/sudoers

### Spin up Machines on AWS

**Add boto configuration**

```
$ cat >> ~/.boto
[Credentials]
aws_access_key_id = CDABGHEFCDABGHEFCDAB
aws_secret_access_key = ABGHEFCDABGHEFCDABGHEFCDABGHEFCDABGHEFCDAB
^D
```

(and replace fake credentials above with your real credentials)

**Add AWS env variables**

```
$ export AWS_ACCESS_KEY_ID=CDABGHEFCDABGHEFCDAB
$ export AWS_SECRET_ACCESS_KEY=ABGHEFCDABGHEFCDABGHEFCDABGHEFCDABGHEFCDAB
$ export AWS_KEY=<your-aws-keypair-name>
```

**To run tests or ansible scripts**

```
$ export KEYNAME=key_<your-aws-keypair-name>
```

**To gather data in Splunk you will want to set variable below**

```
$ export SPLUNK_SERVER="<url_of_splunk_server>:<port>"
$ export SPLUNK_SERVER_AUTH="<username>:<password>"
```

**To kick off cluster**

```
$ cd provision
$ python create_and_instantiate_cluster.py \
    --stackname="YourCloudFormationStack" \
    --num-servers=1 \
    --server-type="m3.large" \
    --num-sync-gateways=1 \
    --sync-gateway-type="m3.medium" \
    --num-gatlings=1 \
    --gatling-type="m3.medium" 
```

This script performs a series of steps for you

1) It uses [troposphere](https://github.com/cloudtools/troposphere) to generate the Cloudformation template (a json file). The Cloudformation config is declared via a Python DSL, which then generates the Cloudformation Json.

2) The generated template is uploaded to AWS with ssh access to the AWS_KEY name you specified (assuming that you have set up that keypair in AWS prior to this)

## Configure conf/hosts.ini 

Copy conf/hosts.ini.example to conf/hosts.ini

**AWS**

If you are running on AWS, you will need to manually generate a conf/hosts.ini file so that the provisioning scripts have a working Ansible Inventory to use.  Eventually, this step will be automated.


* Open conf/hosts.ini
* Go to the AWS console and find the public hostnames of your servers
* Update conf/hosts.ini with these hostnames, depending on their respective roles
* Save the conf/hosts.ini file

**VMs**

* Open conf/hosts.ini and edit


## Setup hosts / deploy shared key

This will generate a 'temp_ansible_hosts' file from a conf/host-file-name.ini that will be used in provisioning and running tests.
If you change want to changes your cluster definition, you must rerun this to regenerate the ansible host file.

```
python conf/ini_to_ansible_host.py --ini-file=conf/hosts.ini
```

Now update the generated temp_ansible_hosts file to have this format:

```
[couchbase_servers]
ec2-54-205-165-155.compute-1.amazonaws.com ansible_ssh_host=ec2-54-205-165-155.compute-1.amazonaws.com

[sync_gateways]
ec2-54-158-112-128.compute-1.amazonaws.com ansible_ssh_host=ec2-54-158-112-128.compute-1.amazonaws.com

[load_generators]
ec2-54-163-112-228.compute-1.amazonaws.com ansible_ssh_host=ec2-54-163-112-228.compute-1.amazonaws.com
```

(this is needed for the gateload config generation script to work)

**VMs only**

One time only. Ansible playbooks require ssh access to run on the target hosts.  This script will attempt to install a common public key to ~/.ssh/knownhosts on the machines in the cluster via ssh-copy-id. 

```
~/.ssh » ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/sethrosetter/.ssh/id_rsa):<test-key>
```

Generate the ansible hosts file and attempt to install the shared key on all machines in the cluster

```
python conf/ini_to_ansible_host.py --ini-file=conf/hosts.ini --install-key=<test-key>.pub --ssh-user=<user>
```

## Set your user

If your ssh user is different then root, you may need to edit provision/ansible/playbooks/ansible.cfg

## Provision Cluster 

Example building from source:

```
$ python provision/provision_cluster.py --server-version=3.1.1 --sync-gateway-branch=feature/distributed_index_bulk_set
```

Example from a pre-built version (dev build):

```
$ python provision/provision_cluster.py --server-version=3.1.1 --sync-gateway-dev-build-url=feature/distributed_index --sync-gateway-dev-build-number=345
```

Like all scripts, run `python provision/provision_cluster.py -h` to view help options.

If you experience ssh errors, you may need to verify that the key has been added to your ssh agent

```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/<test-key>
```

## Run Performance Tests

**Gateload**

Currently the load generation is specified in ansible/files/gateload_config.json.
(In progress) Allow this to be parameterized

```
python run_tests.py
    --use-gateload
    --gen-gateload-config
```

**Gatling**

```
python run_tests.py
    --number-pullers=0
    --number-pushers=7500
```

## Run Functional Tests



