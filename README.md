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

Si el nodo se creó desde cero, entonces el paso _Creación de un nuevo nodo_ modificó diversos ficheros, a saber:

* Si el nodo era validador, se modifican los siguientes ficheros:
	* [`data/permissioned-nodes_general.json`](data/permissioned-nodes_general.json)
	* [`data/permissioned-nodes_validator.json`](data/permissioned-nodes_validator.json)
	* [`data/static-nodes.json`](data/static-nodes.json)
* Si el nodo era general, se modican en cambio estos ficheros:
	* [`data/constellation-nodes.json`](data/constellation-nodes.json)
	* [`data/permissioned-nodes_validator.json`](data/permissioned-nodes_validator.json)

Nótese que el nombre de los ficheros hace referencia a los nodos por los que son consumidos, no a los nodos que los han modificado durante su creación.

Además de estos cambios, que ocurren automáticamente al ejecutar el script [`init.sh`](scripts/init.sh), existen otros dos ficheros que deben modificarse manualmente, dependiendo del tipo de nodo creado, para indicar los datos de contacto de administración de los nodos: [DIRECTORY_VALIDATOR.md](DIRECTORY_VALIDATOR.md) o [DIRECTORY_REGULAR.md](DIRECTORY_REGULAR.md)

Una vez que disponemos de todos estos ficheros modificados solo es necesario arrancarlo usando el script [`start.sh`](scripts/start.sh)

En cambio, si el nodo es validador, el resto de nodos de la red, deben ejecutar el script `restart.sh` con la opción onlyUpdate:
```
$ ./restart.sh onlyUpdate
```

Entonces, el el fichero `~/alastria/logs/quorum-XXX.log` del nuevo nodo validador aparecerá el siguiente mensaje de error:
```
ERROR[12-19|12:25:05] Failed to decode message from payload    address=0x59d9F63451811C2c3C287BE40a2206d201DC3BfF err="unauthorized address"
```
Esto es debido a que el resto de validadores de la red no han aceptado todavía al nodo como validador. Para solicitar dicha aceptación debemos anotar la dirección (address) del nodo.

### Publicación del nodo ###

Con los ficheros modificados, tanto automáticamente como manualmente, se debe crear un pull request contra este repositorio. Si el nodo era validador y tiene un address que aún no está autorizado, entonces debe indicarse dicha dirección en el pull request.

Para la inclusión en la red de nuevos nodos validadores, los administradores del resto de miembros validadores deben usar el RPC API para añadir su dirección:


```
> istanbul.propose("0x59d9F63451811C2c3C287BE40a2206d201DC3BfF", true);
```

Así, el nuevo nodo estará levantado y sincronizado con la red.

> **NUNCA DEBE REALIZARSE EL PROPOSE SIN HABER ACTUALIZADO ANTES LOS FICHEROS DE PERMISIONADO (restart.sh onlyUpdate).**

> **NUNCA SE DEBE ELIMINAR UN NODO DE LA RED SIN REALIZAR LA SOLICITUD DE ELIMINACIÓN PRIMERO A TRAVÉS DE UN PULL REQUEST PARA QUE EL RESTO DE MIEMBROS VALIDADORES LOS ELIMINEN DE SUS LISTAS PRIMERO Y REALICEN UNA RONDA DE VOTACIÓN:**

```
> istanbul.propose("0x59d9F63451811C2c3C287BE40a2206d201DC3BfF", false);
```

### Reinicialización de un nodo existente ###

Si ya disponemos de un nodo Alastria instalado en la máquina, y deseamos realizar una inicialización limpia del nodo manteniendo nuestro **enode**, nuestras claves constellation y nuestras cuentas actuales, podemos ejecutar:

    ```
	$ ./init.sh backup <<NODE_TYPE>> <<NODE_NAME>>
	```

Este será el procedimiento a seguir por los nodos miembros ante actualizaciones de la infraestructura.

### Operación del nodo ###

 * Faced with errors in the node, we can choose to perform a clean restart of the node, for this we must execute the following commands:
```
$ systemctl restart constellation
$ systemctl restart geth
```

 * Todos los nodos incluyen un monitor que permiten al equipo técnico de Alastria realizar labores de mantenimiento y gestión. La ejecución
del monitor es opcional y puede ejecutarse lanzando el script de start con el flag `--monitor`:
```
$ ./start.sh --monitor
```
 * También, disponemos de un script de restart para actualizar el nodo sin parar ningún proceso (por ejemplo ante actualizaciones del permissioned-nodes*):
```
$ ./restart.sh onlyUpdate
```
O para reiniciar completamente
el nodo:
```
$ ./restart.sh auto || <<CURRENT_HOST_IP>>
```

 * El script `./scripts/backup.sh` permite realizar copias de seguridad del estado del nodo. Ejecutando `./scripts/backup.sh keys` se hace una copia de seguridad de las claves
y el enode de nuestro nodo y con `./scripts/backup.sh full` realizamos una copia de seguridad de todo el estado del nodo y de la blockchain. Todas las copias de seguridad se almacenan en el directorio home como `~/alastria-keysBackup-<date>/` y `~/alastria-backup-<date>/`, respectivamente.

 * Existe un script `./scripts/clean.sh` que limpia el nodo actual y exige una resincronización del mismo al iniciarlo de nuevo. Esto solventa posibles errores de sincronización. Su efecto es el mismo que el de ejecutar directamente `./scripts/start.sh clean`


**NOTE**
If we want to generate the node using an enode and the keys of an existing node we must make a backup of the keys
of the old node:
```
$ ansible-playbook -i inventory --private-key=~/.ssh/id_rsa -u vagrant site-everis-alastria-backup.yml
```
This will generate the folder ~/alastria-keysBackup-<date> whose contents should be moved to ~ / alastria-node / data / keys.
The keys of this directory (which has to keep the folder structure of the generated backup) will be the ones used
in the image of the node that we are going to generate.
of the old node:
