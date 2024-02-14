# Oracle APEX 23c Container

See for more information:

* [Oracle Database XE Release 21c (21.3.0.0)](https://container-registry.oracle.com/ords/ocr/ba/database/express)
* [Oracle Database Free Release 23c (23.3.0.0)](https://container-registry.oracle.com/ords/ocr/ba/database/free)
* [Oracle Autonomous Database Free Repository](https://container-registry.oracle.com/ords/ocr/ba/database/adb-free)
* [Oracle REST Data Services (ORDS) with Application Express](https://container-registry.oracle.com/ords/ocr/ba/database/ords)
* [Seed Up Colima on a M1/M2 MacBook](https://www.tyler-wright.com/using-colima-on-an-m1-m2-mac/)

## MacBook M1/M2 Chipset

Setup a Dockers container without Docker Desktop using Colima project instead and use Podman Desktop as UI Dashboard.

Install required software:

``` bash
# Podman
brew install podman
brew install podman-desktop

# Docker and dependencies
brew install docker
brew install docker-buildx
brew install docker-completion
brew install docker-compose
brew install docker-credential-helper

# Emulate x86_64 software with Colima and QEMU.
brew install colima
brew reinstall qemu
```

### Colima x86_64 Virtual Machine

* Quick notes:
  * On initial startup, Colima initiates with a user specified runtime that defaults to Docker.
  * The default VM created by Colima has 2 CPUs, 2GiB memory and 60GiB storage.
  * The VM can be customized either by passing additional flags to colima start. e.g. `--cpu`, `--memory`, `--disk`, `--runtime`. Or by editing the config file with `colima start --edit`.
&nbsp;
* Create a x86 arch VM:

  ``` bash
  colima start --profile x86_64 --cpu 8 --memory 12 --arch x86_64 --vm-type=vz --network-address
  ```

* Use Rosetta virtualization framework:

  ``` bash
  # To install rosetta framework
  softwareupdate --install-rosetta

  # (Optional) You must delete a colima vm if already exists
  colima stop
  colima delete

  # Start a colima vm emulating x86_64 using Apple's new virtualization framework
  colima start --cpu 8 --memory 12 --arch x86_64 --vm-type vz --vz-rosetta
  ```

  *The command below should not be used, Oracle Database is not compatible with aarch64 cpus. It is only listed for documentation purposes.*

  ``` bash
  colima start --cpu 4 --memory 8 --arch aarch64 --vm-type=vz --vz-rosetta --network-address
  ```
  
&nbsp;
> Note: Running x86_64 arch containers can have issues translating instructions for ARM. Give it higher memory to the VM to avoid such issues.

&nbsp;

When starting a Colima virtual machine, it creates a sock file under `~/.colima` folder. The exact location depends if you add a profile while creating.

* Default: This is the default profile if you don't include the `--profile` flag on the start command.

  ``` bash
  ~/.colima/docker.sock
  ```

* If include the `--profile` flag, it will create a folder with the same name of the profile. 
  Example: `x86_64`

  ``` bash
  ~/.colima/<PROFILE_NAME>/docker.sock
  # ==== Example: Profile "x86_64"
  ~/.colima/x86_64/docker.sock
  ```

### Use Podman Desktop with Dockers and Colima

* Replacing Docker Desktop with Podman Desktop

  1. Install Podman CLI and Podman Desktop through the Podman official website or by running:

      ``` bash
      brew install podman
      brew install podman-desktop
      ```

  2. Use Colima Docker sock file for Podman.
      * By default Podman Desktop will look at `/var/run/docker.sock` for Dockers Sock file.  
      * Create a symlink so Podman Desktop can communicate with Colima VM:

        ``` bash
        # Default location
        sudo ln -s ~/.colima/docker.sock /var/run/docker.sock

        # Profile defined location
        sudo ln -s ~/.colima/<PROFILE_NAME>/docker.sock /var/run/docker.sock
        # ==== Example: Profile "x86_64"
        sudo ln -s ~/.colima/x86_64/docker.sock /var/run/docker.sock
        ```

      * Docker engine should be "Running" on Podman Desktop.

#### Troubleshooting some Colima issues

* Disabling ipv6 in colima VM. (MacOS only)

  ``` bash
  colima ssh
  sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
  ### Start Colima with the previous command ###
  ```

  > See "https://github.com/abiosoft/colima/issues/801"

* [Containers in emulated x86_64 VM become unreachable after a while (connection refused)](https://github.com/abiosoft/colima/issues/962)

---

If you are running Dockers on Windows or Linux x86, you can use Podman instead of Dockers installation

## Authenticate with Oracle Container Registry

Login to the Oracle Container Registry by running the command below:

``` bash
docker login container-registry.oracle.com
```

## Setup the network interface

Both containers, oracle database and ords server have to be on the same network so they can see each other during APEX installation and ORDS serve.

``` bash
docker network create apex-network
```

## Setup the volume device

You must create and use a Docker/Podman volume to make data persist.

``` bash
docker volume create apex-database-volume
```

## Configure Oracle Database and Oracle REST Data Services

### Using Oracle Database XE (21c)

* Pull the Oracle Database XE image:

  ``` bash
  docker pull container-registry.oracle.com/database/express:latest
  ```

* Tag the image and assign it a friendly name:

  ``` bash
  docker image tag container-registry.oracle.com/database/express:latest oracle-database-latest
  ```

* Delete the non-tagged image:

  ``` bash
  docker rmi container-registry.oracle.com/database/express:latest
  ```

### Using Oracle Database Free (23c)

* Pull the Oracle Database Free image:

  ``` bash
  docker pull container-registry.oracle.com/database/free:latest
  ```

* Tag the image and assign it a friendly name:

  ``` bash
  docker image tag container-registry.oracle.com/database/free:latest oracle-database-latest
  ```

* Delete the non-tagged image:

  ``` bash
  docker rmi container-registry.oracle.com/database/free:latest
  ```

* List the images:

  ``` bash
  docker images
  ```

* Run the database container:
Notice the `--hostname` flag is declaring a firendly hostname for the database container. This will be use later during the APEX installation.
You must change the `<Oracle Password>` for your own password.

  * without persistent data.

    ``` bash
    docker run -d --name apex-database \
               -p 1521:1521 \
               -p 5500:5500 \
               -e ORACLE_PWD=<Oracle Password> \
               -e ORACLE_CHARACTERSET=AL32UTF8 \
               --network=apex-network \
               --hostname apexdatabase \
               oracle-database-latest
    ```

* B) with persistent data.

  ``` bash
  docker run -d --name apex-database \
             -p 1521:1521 \
             -p 5500:5500 \
             -e ORACLE_PWD=<Oracle Password> \
             -e ORACLE_CHARACTERSET=AL32UTF8 \
             --network=apex-network \
             --hostname apexdatabase \
             -v apex-database-volume:/opt/oracle/oradata \
             oracle-database-latest
  ```

* You can monitor the startup process with:

  ``` bash
  docker logs -f apex-database
  ```

After the Oracle Database server indicates that the container has started, and the `STATUS` field shows `(healthy)`, client applications can connect to the database.

* **Oracle Enterprise Manager Express (Express Database only)**

  ``` log
  https://localhost:5500/em/
  ```

* **Connecting from Within the Container**

  ``` bash
  docker exec -it <oracle-db-container> sqlplus / as sysdba
  docker exec -it <oracle-db-container> sqlplus sys/<Oracle Password>@<XE/FREE> as sysdba
  docker exec -it <oracle-db-container> sqlplus system/<Oracle Password>@<XE/FREE>
  docker exec -it <oracle-db-container> sqlplus pdbadmin/<Oracle Password>@<XE/FREE>PDB1
  ```

* **Connecting from Outside the Container**

  By default, Oracle Database exposes port 1521 for Oracle client connections, using Oracle's SQL*Net protocol. SQL*Plus or any Oracle Java Database Connectivity (JDBC) client can be used to connect to the database server from outside of the container.
  To connect from outside of the container, start the container with the -p option, as described in the detailed Docker run command in the "Custom Configurations" section.
  &nbsp;
  Discover the mapped port by running the following command:
  
  ``` bash
  docker port apex-database #<container name>
  ```

  To connect from outside of the container using SQL*Plus, run the following commands:

  ``` bash
  # To connect to the database at the CDB$ROOT level as sysdba:
  sqlplus sys/<Oracle Password>@//localhost:1521/<XE/FREE> as sysdba

  # To connect as non sysdba at the CDB$ROOT level:
  sqlplus system/<Oracle Password>@//localhost:1521/<XE/FREE>

  # To connect to the default Pluggable Database (PDB) within the XE Database:
  sqlplus pdbadmin/<Oracle Password>@//localhost:1521/<XE/FREE>PDB1
  ```

### Using Oracle REST Data Services

* Pull the ORD image from registry:

  ``` bash
  docker pull container-registry.oracle.com/database/ords:latest
  ```

* Tag the image and assign it a friendly name:

  ``` bash
  docker image tag container-registry.oracle.com/database/ords:latest oracle-ords-latest
  ```

* Delete the non-tagged image:

  ``` bash
  docker rmi container-registry.oracle.com/database/ords:latest
  ```

* List the images:

  ``` bash
  docker images
  ```

#### Install APEX on Database Container

* On first time, create a directory for the `conn_string.txt` file which will hold the connection string that the container will use. This will be mapped as a specific volume when starting the container.

  ``` bash
  mkdir ords_secrets ords_config
  chmod 777 ords_config
  echo 'CONN_STRING=sys/<Oracle Password>@apexdatabase:1521/<XE/FREE>PDB1' > ords_secrets/conn_string.txt
  ```

* Run ORDS container without configuration volume:

  ``` bash
  docker run --rm --name apex-ords \
            -p 8181:8181 \
            -v <Your Project Path>/ords_secrets/:/opt/oracle/variables \
            --network=apex-network \
            oracle-ords-latest
  ```

* Login APEX

  ``` bash
  - URL:       http://localhost:8181/ords
  - Workspace: internal
  - User:      ADMIN
  - Password:  Welcome_1
  ```

  > Note: Both containers are attached to the same "Network Device" previously created, so both are running in the same network and can "see" each other.

**Using an ORDS configuration volume**
You can set a configuration mount with your ORDS configurations pointing to /etc/ords/config and the container will try to start the service with configurations on the mount point.

* Run the container with ords configurations mount .

  ``` bash
  docker run --rm --name apex-ords \
            -p 8181:8181 \
            -v <Your Project Path>/ords_secrets/:/opt/oracle/variables \
            -v <Your Project Path>/ords_config/:/etc/ords/config/ \
            --network=apex-network \
            oracle-ords-latest
  ```

**Configure SSL with Self-Signed Certificate**

* Set an SSL configuration.
To start a secure service on the ORDS container, create a folder called "ssl", put your Certificate and Key files in this folder, the files needs to be named "cert.crt" and "key.key".

  ``` bash
  mkdir -p ords_config/ssl
  cp cert_file.crt ords_config/ssl/cert.crt
  cp key_file.key  ords_config/ssl/key.key

  chmod -R 777 ords_config/
  ```

  > You must have the certificate installed on your host machine.


* Running with Config Volume so `ssl` folder can be detected by the initialization script on the ORDS Docker container:

  ``` bash
  docker run --rm --name apex-ords \
            -p 8181:8181 \
            -v <Your Project Path>/ords_secrets/:/opt/oracle/variables \
            -v <Your Project Path>/ords_config/:/etc/ords/config/ \
            --network=apex-network \
            oracle-ords-latest
  ```

* Login to APEX using SSL:

  ``` bash
  - URL:       https://localhost:8181/ords
  - Workspace: internal
  - User:      ADMIN
  - Password:  Welcome_1
  ```

_For more information, see [Simplified SSL Configuration for Oracle REST Data Services (ORDS) : A How-To Guide](https://blogs.oracle.com/fusionmiddlewaresupport/post/simplified-ssl-configuration-for-ords-a-howto-guide)_

----
----

## Run APEX with Autonomous Database

* Pulling the Oracle ADB image from registry.

  ``` bash
  docker pull container-registry.oracle.com/database/adb-free:latest
  ```

* Tag the image and assign it a friendly name:

  ``` bash
  docker image tag container-registry.oracle.com/database/adb-free:latest oracle-adb-latest
  ```

* Delete the non-tagged image:

  ``` bash
  docker rmi container-registry.oracle.com/database/adb-free:latest
  ```

* List the images:

  ``` bash
  docker images
  ```

### Configure the Autonmous Database

``` bash
docker run -d --name oracle-apex-atp\
           -p 1521:1522 \
           -p 1522:1522 \
           -p 8443:8443 \
           -p 27017:27017 \
           -e WORKLOAD_TYPE='ATP' \
           -e WALLET_PASSWORD=<Oracle Password> \
           -e ADMIN_PASSWORD=<Oracle Password> \
           -e no_proxy=localhost,127.0.0.1 \
           -e NO_PROXY=localhost,127.0.0.1 \
           --network=apex-network \
           --hostname apex-atp-server
           --cap-add SYS_ADMIN \
           --device /dev/fuse \
    oracle-adb-latest
```

### Connecting to Oracle Autonomous Database Free container

Container hostname is used to generate self-signed SSL certs to serve HTTPS traffic on port 8443. APEX and Database Actions can be accessed using the container host (or simply localhost)

#### Wallet Setup

In the container, TLS wallet is generated at location /u01/app/oracle/wallets/tls_wallet

``` bash
# Copy wallet to your host.
docker cp oracle-adb-latest:/u01/app/oracle/wallets/tls_wallet/adb_container.cert ./adb_container.cert

# Linux hosts only: Install the Certificate
sudo cp adb_container.cert /etc/pki/ca-trust/source/anchors
sudo update-ca-trust
```

For MacOS, please refer the [support guide](https://support.apple.com/guide/keychain-access/add-certificates-to-a-keychain-kyca2431/mac) to add certificate to keychain

For JDK truststore update, you can use `keytool`:

* Linux:

  ``` bash
  sudo keytool -import -alias adb_container_certificate -keystore $JAVA_HOME/lib/security/cacerts -file ./adb_container.cert 
  ```

* MacOS:

  ``` bash
  sudo keytool -import -alias adb_container_certificate -file ./adb_container.cert -keystore /Library/Java/JavaVirtualMachines/<Your JDK Version>/Contents/Home/lib/security/cacerts
  ```

  > Only if you install Java using a DMG image.

#### Available TNS aliases

| mTLS | TLS |
|---|---|
|my_<atp/adw>_medium   | my_<atp/adw>_medium_tls   |
|my_<atp/adw>_high     | my_<atp/adw>_high_tls     |
|my_<atp/adw>_low      | my_<atp/adw>_low_tls      |
|my_<atp/adw>_tp       | my_<atp/adw>_tp_tls       |
|my_<atp/adw>_tpurgent | my_<atp/adw>_tpurgent_tls |

### adb-cli

`adb-cli` can be used to perform database operations after container is up and running.
To use adb-cli, you can define the following alias for convenience

``` bash
alias adb-cli="podman exec <container_name> adb-cli"
```

Example:

``` bash
alias adb-cli="docker exec oracle-apex-atp adb-cli"
```

#### Available commands

``` log
>> adb-cli --help

Usage: adb-cli [OPTIONS] COMMAND [ARGS]...

  ADB-S Command Line Interface (CLI) to perform container-runtime database
  operations

Options:
  -v, --version  Show the version and exit.
  --help         Show this message and exit.

Commands:
  add-database
  change-password
```

#### Add Database

You can add a database using the `add-database` command.

``` bash
adb-cli add-database --workload-type "ADW" --admin-password "Welcome_1234"
```

#### Change Password

To change password for Admin user, use the following command.

``` bash
adb-cli change-password --database-name "MY_ADW" --old-password "Welcome_1234" --new-password "Welcome_12345"
```

### Migrating Data Across Containers

#### Docker Volume

To create a volume run the following command:

``` bash
docker volume create "apex-atp-volume"
```

Confirm existing volumes.

``` bash
docker volume ls
```

#### Mount Volume

To persist data across container restarts and removals, you should mount a volume at `/u01/data` and follow the steps mentioned in the [documentation to migrate PDB data across containers](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/autonomous-docker-container.html#GUID-03B5601E-E15B-4ECC-9929-D06ACF576857).

``` bash
docker run -d \
           -p 1521:1522 \
           -p 1522:1522 \
           -p 8443:8443 \
           -p 27017:27017 \
           -e WORKLOAD_TYPE='ATP' \
           -e WALLET_PASSWORD=<Oracle Password> \
           -e ADMIN_PASSWORD=<Oracle Password> \
           -e no_proxy=localhost,127.0.0.1 \
           -e NO_PROXY=localhost,127.0.0.1 \
           --network=bridge \
           --cap-add SYS_ADMIN \
           --device /dev/fuse \
           --name adb-free \
 \ ################################
           --volume apex-atp-volume:/scratch/apex-data \
 \ ################################
    oracle-adb-latest
```
