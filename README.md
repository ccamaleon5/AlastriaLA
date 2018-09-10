# ALASTRIA LATIN AMERICA AND THE CARIBBEAN #

## References

* In the next installation we will use **Ubuntu 18.04** as operating system and all the commands related to this operating system. In addition, links of the prerequisites will be placed in case it is required to install in another operating system.

* An important consideration is that we will use Ansible, for which the installation is done from a local machine on a remote server. That means that the local machine and the remote server will communicate by ssh.

* The Github of Alastria can be followed as a reference [https://github.com/alastria/alastria-node](https://github.com/alastria/alastria-node)

## System Requirements

Features of the machine for nodes of the testnet:

* **CPU's**: 2 cores

* **RAM Memory**: 4 Gb

* **Hard Disk**: 30 Gb SSD

* **Operating System**: Ubuntu 16.04, Ubuntu 18.04, CentOS 7.4 or Red Hat Enterprise Linux 7.4, always 64 bits

It is necessary to enable the following network ports in the machine in which we are going to deploy the node:

* **8443**: TCP - Port to monitor

* **9000**: TCP - Port for communication of Constellation.

* **21000**: TCP/UDP - Port to establish communication between geth processes.

* **22000**: TCP - Port to establish the RPC communication. (this port is used for applications that communicate with Alastria, and may be leaked to the Internet)

## Prerequisites

### Install Ansible ###

For the installation we will use Ansible. For this reason it is necessary to install Ansible on the **local machine** that will perform the installation of the node on the remote machine.

Following the instructions to [install ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) in local machine.

```
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

### Clone Repository ####

To configure and install Quorum and Constellation, you must clone this git repository in **local machine**.

```
$ git clone https://github.com/alastria/alastria-node.git
$ cd alastria-node/
```

### Install Python ###

* In order for ansible to work, it is necessary to install Python on the **remote machine** where the node will be installed, for this reason it is necessary to install python2.7 and python-pip.

* If you need to install python-pip in Redhat use [https://access.redhat.com/solutions/1519803](https://access.redhat.com/solutions/1519803)

```
$ sudo apt-get update
$ sudo apt-get install python2.7
$ sudo apt-get install python-pip
```

## Quorum + Constellation Instalation ##

### Creation of a new Node ###

* There are two types (Validator / Regular) that can be created in the Quorum network.

* First change the IP located within the **inventory file** by the public IP of the remote server that will be the node
        ```
	$ vi inventory
        [test]
        192.168.10.72
	```

* To deploy a validator node execute the next sentence in local machine.

	```
	$ ansible-playbook -i inventory -e first_node=false --private-key=~/.ssh/id_rsa -u vagrant site-everis-alastria-validator.yml
	```

* To deploy a regular node execute the next sentence in local machine.

	```
	$ ansible-playbook -i inventory --private-key=~/.ssh/id_rsa -u vagrant site-everis-alastria-regular.yml
	```
* When starting the installation, it will request that some data be entered, such as the public IP, node type, account password and node name. The name of the node will be the one that will finally appear in the network monitoring tool.

* At the end of the installation, if everything was correct, a GETH service will be created for the case of the validator node managed by Systemctl and with stop status.

* In the case of a regular node if everything was correct, a CONSTELLATION service and a GETH service managed by Systemctl will be created and with stop status.

* Now, it is necessary to perform the configuration of files before the execution of the node, please, follow the next steps.

## Node Configuration
It is necessary to follow the next steps for the configuration of the nodes:

### Configuración del fichero de nodos Quorum ###

If the node was created from scratch, then step _Creating a new node_ modified several files, namely:

* If the node was a validator, the following files are modified:
	* [`data/permissioned-nodes_general.json`](data/permissioned-nodes_general.json)
	* [`data/permissioned-nodes_validator.json`](data/permissioned-nodes_validator.json)
	* [`data/static-nodes.json`](data/static-nodes.json)
* If the node was general, these files are modified instead:
	* [`data/constellation-nodes.json`](data/constellation-nodes.json)
	* [`data/permissioned-nodes_validator.json`](data/permissioned-nodes_validator.json)

Note that the name of the files refers to the nodes by which they are consumed, not the nodes that have modified them during their creation.

In addition to these changes, which occur automatically when executing the node creation, there are two other files that must be modified manually, depending on the type of node created, to indicate the contact data of Administration of the nodes: [DIRECTORY_VALIDATOR.md] (DIRECTORY_VALIDATOR.md) or [DIRECTORY_REGULAR.md] (DIRECTORY_REGULAR.md)

### Start Regular Node ###

Once we have all these modified files it is only necessary to start it using the command:

```
$ systemctl start constellation
$ systemctl start geth
```

### Start Validator Node ####

On the other hand, if the node is a validator, the rest of the nodes in the network must execute:

```
$ systemctl start geth
```

Then, the file `~/alastria/logs/quorum-XXX.log` of the new validator node the following error message will appear:
```
ERROR[12-19|12:25:05] Failed to decode message from payload    address=0x59d9F63451811C2c3C287BE40a2206d201DC3BfF err="unauthorized address"
```
This is because the rest of the validators in the network have not yet accepted the node as a validator. To request such acceptance we must note the address of the node. the following error message

### Publicación del nodo ###

* Con los ficheros modificados, tanto automáticamente como manualmente, se debe crear un **pull request** contra este repositorio. 

* Si el nodo era validador y tiene un address que aún **no está autorizado**, entonces debe indicarse dicha **dirección** en el pull request.

* For the inclusion in the network of new validator nodes, the administrators of the rest of the validating members must use the RPC API to add their address:


```
> istanbul.propose("0x59d9F63451811C2c3C287BE40a2206d201DC3BfF", true);
```

Thus, the new node will be raised and synchronized with the network.

> **NEVER MAKE THE PROPOSE WITHOUT UPDATING BEFORE THE FILES OF PERMISSIONED (Systemctl restart geth).**

> **A NETWORK NODE MUST NEVER BE ELIMINATED WITHOUT MAKING THE REMOVAL APPLICATION FIRST THROUGH A PULL REQUEST SO THAT THE REST OF VALIDATING MEMBERS WILL REMOVE THEM FROM THEIR LISTS FIRST AND MAKE A VOTING ROUND:**

```
> istanbul.propose("0x59d9F63451811C2c3C287BE40a2206d201DC3BfF", false);
```

### Reinitialization of an existing node ###

* If we already have an Alastria node installed on the machine, and we want to perform a clean initialization of the node keeping our **enode**, our constellation keys and our current accounts, we can execute:

    ```
    $ ./init.sh backup <<NODE_TYPE>> <<NODE_NAME>>
    ```

This will be the procedure to be followed by the member nodes before infrastructure updates.

### Node Operation ###

 * Faced with errors in the node, we can choose to perform a clean restart of the node, for this we must execute the following commands:
```
$ systemctl restart constellation
$ systemctl restart geth
```

 * Also, we have a restart script to update the node without stopping any process (for example before permissioned-nodes* updates):
```
$ systemctl restart geth
```

 * The next statement allows you to back up the node's state. Makes a backup copy of the keys and the enode of our node. All backup copies are stored in the home directory as `~/alastria-keysBackup`.

ansible-playbook -i inventory -e validator=true --private-key=~/.ssh/id_rsa -u vagrant site-everis-alastria-backup.yml 


 * Existe un script `./scripts/clean.sh` que limpia el nodo actual y exige una resincronización del mismo al iniciarlo de nuevo. Esto solventa posibles errores de sincronización. Su efecto es el mismo que el de ejecutar directamente `./scripts/start.sh clean`


**NOTE**
If we want to generate the node using an enode and the keys of an existing node we must make a backup of the keys
of the old node:

```
$ ansible-playbook -i inventory -e validator=true --private-key=~/.ssh/id_rsa -u vagrant site-everis-alastria-backup.yml 

```

This will generate the folder ~/alastria-keysBackup whose contents should be moved to ~/alastria/data/keys.
The keys of this directory (which has to keep the folder structure of the generated backup) will be the ones used
in the image of the node that we are going to generate.
