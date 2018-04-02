# docker-oracle-setup

This document will keep an up to date version of my personal Oracle dockerized development environment. The main goal is to have one Oracle database with multiple versions of APEX installed. 

This is achieved using Oracle 12c containers (not to be confused with Docker containers). If you're not too familiar with Oracle 12c containers I highly recommend reading [this](http://www.oracle.com/technetwork/articles/database/multitenant-part1-pdbs-2193987.html) article which covers Container Databases (CDB) and Pluggable Databases (PDB).

<!-- TOC depthFrom:2 -->

- [Known Issues and Future Improvements](#known-issues-and-future-improvements)
- [References](#references)
- [Background](#background)
  - [Port Mapping](#port-mapping)
  - [Passwords](#passwords)
  - [Download Files](#download-files)
  - [Laptop Folder Structure](#laptop-folder-structure)
  - [Oracle Container Registry Setup](#oracle-container-registry-setup)
- [Setup](#setup)
  - [Directory structure](#directory-structure)
  - [Docker Network](#docker-network)
  - [Docker Images](#docker-images)
    - [Get Oracle Image](#get-oracle-image)
    - [ORDS](#ords)
    - [Docker Image Confirmation](#docker-image-confirmation)
- [Docker Containers](#docker-containers)
  - [Oracle Container](#oracle-container)
  - [Oracle CDB and PDB Setup](#oracle-cdb-and-pdb-setup)
    - [CDB Setup](#cdb-setup)
    - [PDB Setup](#pdb-setup)
      - [Create PDBs](#create-pdbs)
      - [APEX 5.0.4 Install](#apex-504-install)
      - [APEX 5.1.3 Install](#apex-513-install)
  - [ORDS Containers](#ords-containers)
    - [ORDS 5.0.4](#ords-504)
    - [ORDS 5.1.3](#ords-513)
    - [ORDS Container Wrappup](#ords-container-wrappup)
- [Useful Commands](#useful-commands)
  - [Quick Start](#quick-start)
  - [Docker](#docker)
  - [Oracle](#oracle)
    - [Connection Strings](#connection-strings)
    - [DBA](#dba)
    - [Sample Data](#sample-data)

<!-- /TOC -->



## Known Issues and Future Improvements

- Use Oracle's official ORDS image from the Container Registry

## References

 The following articles and all my scripts are a result of a combination of the code found in the links.
 
- [APEX and ORDS up and running in....2 steps!](http://joelkallman.blogspot.ca/2017/05/apex-and-ords-up-and-running-in2-steps.html) by [Joel Kallman](https://twitter.com/joelkallman)
- [Dockerize your APEX development environment](http://roelhartman.blogspot.ca/2017/10/dockerize-your-apex-development.html) by [Roel Hartman](https://twitter.com/RoelH)
- [Oracle Database 12c now available on Docker](https://sqlmaria.com/2017/04/27/oracle-database-12c-now-available-on-docker/) by [Maria Colgan](https://twitter.com/sqlmaria)

## Background

- All my scripts are Linux / MacOS focused. If you use a Windows machine you'll need to translate
- I specifically made reference to "your laptop" to emphasize what was run "on your machine" vs "in a docker container"
- I use [SQLcl](http://www.oracle.com/technetwork/developer-tools/sqlcl/overview/index.html) instead of SQLplus on my laptop. I've also renamed the default command `sql` to `sqlcl`. You can use `sqlplus` or follow my [sqlcl install instructions](http://www.talkapex.com/2015/04/installing-sqlcl/).

### Port Mapping

The following port mapping will be used. To help remember the DB version, the database port ends in its version number `122` for `12.2`. Likewise, the ORDS ports end in the APEX version they link to ex: `513` for `5.1.3`.

Container | Container Port | Laptop Port | Description
--- | --- | --- | ---
`oracle` | `1521` | `32122` | TNS listener for Oracle 12.2
`ords-504` | `8080` | `32504` | ORDS for APEX 5.0.4
`ords-513` | `8080` | `32513` | ORDS for APEX 5.1.3

### Passwords

Since this setup is for my own development environment (and to keep things simple) the following passwords will be used for the setup

Container | Username | Password | Description
--- | --- | --- | ---
`oracle`  | `sys` |  `Oradoc_db1` | All sys passwords for all PDBs will be the same
`oracle`  | `admin` |  `Oradoc_db1` | Workspace `Internal` for all APEX admin

### Download Files

Due to licensing restrictions I can't host/provide these files in Github or elsewhere. As such you'll need to download them manually. Download the following files and store them in your `~/Downloads` folder:

Application | Description
--- | ---
[APEX 5.1.3](http://www.oracle.com/technetwork/developer-tools/apex/downloads/index.html) | At the time of writing 5.1.3 was the most recent version.
[APEX 5.0.4](http://www.oracle.com/technetwork/developer-tools/apex/downloads/all-archives-099381.html) | This is the APEX archive page. Click on the [5.0 Archive](http://www.oracle.com/technetwork/developer-tools/apex/downloads/apex-5-archive-2606313.html) link.
[ORDS 17.4.1](http://www.oracle.com/technetwork/developer-tools/rest-data-services/downloads) | At the time of writing ORDS 17.4.1 was the most recent version.


### Laptop Folder Structure

We'll assume that the folder structure below is setup. A script is provided later on to create this.

Path | Description
--- | ---
`~/docker` | root
`~/docker/apex` | APEX installation files and images for each version. 2 versions of APEX are included in this example but more can easily be added
`~/docker/apex/5.1.3` | APEX 5.1.3 installation files  
`~/docker/apex/5.0.4` | APEX 5.1.3 installation files  
`~/docker/oracle`  |  Oracle 12.2 data files
`~/docker/ords`  |  ORDS Dockerfile (to build ORDS image)
`~/docker/tmp`  |  Temp folder

### Oracle Container Registry Setup

To work with Oracle's licensing for docker (and to avoid manually building images yourself) you'll need to login to the [container-registry.oracle.com](https://container-registry.oracle.com) and use your Oracle Technology Network (OTN) login and register.

One logged in go to `Database > Enterprise` and read the license agreement. If you agree to the Terms and Conditions click the Accept button.


## Setup

The following steps only ever need to be done once.

### Directory structure

The following script will create the directory structure as mentioned above

```bash
mkdir ~/docker
cd docker
mkdir apex
mkdir oracle
mkdir ords
```

Copy the files from the `~/Downloads` folder to the appropriate `~/docker` folder and unzip

```bash
# *** APEX ***
cd ~/docker/apex
cp ~/Downloads/apex*.zip .

# Unzip
unzip apex_5.0.4.zip
mv apex 5.0.4

unzip apex_5.1.3.zip
mv apex 5.1.3

# ORDS will be dealt in the ORDS image section
```

### Docker Network

In order for the containers to “talk” to each other we need to setup a Docker network and associate all the containers on this network. Containers can reference each other by their respective container names. When referencing another container on the same Docker network the port used is the container’s native port **not** the mapped port on your laptop.

```bash
docker network create oracle_network
# Other docker network commands (don't need to run them as part of install)
# Connect and existing container to a docker network
# docker network connect <network name> <container name>
# View a network and connected containers
# In this example "oracle_network" is the network we're interested in
docker network inspect oracle_network
```

### Docker Images

Two images are required to get the setup working. One for Oracle and one for ORDS.

#### Get Oracle Image

*If you haven't already done so already read above about the `container-registry.oracle.com`. You won't be able to pull the container until you agree to the Terms and Conditions.*

```bash
# Login to Oracle's container registry
docker login container-registry.oracle.com

# They're various docker 12c images. To help reduce the number (and size) of images on my laptop I only needed the 12.2 version
# This will take a while to run as the image size is around 3.5 GB
# Note currently the registry server is slow so you may have to wait a while for the image to download
docker pull container-registry.oracle.com/database/enterprise:12.2.0.1
```

#### ORDS

*Oracle has scripts available to build an ORDS image. At the time of writing (11-Nov-2017) there's an issue with the script ([Issue #646](https://github.com/oracle/docker-images/issues/646)) that prevents me from using it. I suspect that Oracle will include a pre-built image on their container registry in the future so I'll update the section with their image when/if it becomes available. In the mean time the image will be built using another script.*

**Important:** I've created a separate [ORDS Docker Image](https://github.com/martindsouza/docker-ords) repo. Since it's being actively updated and to avoid duplication of documentation please refer to the repo to build your ORDS Docker image. **Go through it now to create your ORDS Docker image**.

#### Docker Image Confirmation

At this point you should see the following (or similar) output when running `docker images`

```
docker images

REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
ords                                                17.4.1              a735271f7bf5        17 seconds ago      537MB
tomcat                                              8.0                 643bbbde6032        7 days ago          456MB
container-registry.oracle.com/database/enterprise   12.2.0.1            12a359cd0528        2 months ago        3.44GB
```


## Docker Containers

Now that the setup is complete we can create all the Docker containers

### Oracle Container

Adding the `-e TZ` option will set the appropriate timezine for the OS and the database. A full list of timezones can be found [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). If excluded it will default to UTC.

```bash
docker run -d -it \
--name oracle \
-p 32122:1521 \
-e TZ=America/Edmonton \
--network=oracle_network \
-v ~/docker/oracle:/ORCL \
-v ~/docker/apex:/tmp/apex \
container-registry.oracle.com/database/enterprise:12.2.0.1

# Running docker ps will result in:
docker ps

CONTAINER ID        IMAGE                                                        COMMAND                  CREATED             STATUS                            PORTS                               NAMES
6b4d96d63cf5        container-registry.oracle.com/database/enterprise:12.2.0.1   "/bin/sh -c '/bin/..."   4 seconds ago       Up 5 seconds (health: starting)   5500/tcp, 0.0.0.0:32112->1521/tcp   oracle

# More specifically if you need to create a script around this
# docker inspect --format="{{.State.Health.Status}}" oracle
# Will give you precise information about the status
```

You'll need to run `docker ps` several times until the status is `(healthy)`. Before you see `(healthy)`, the status will be `(unhealthy)`. This is normal. You will be able to monitor the `~/docker/oracle` directory increases in size.

You can follow the install by running `docker logs oracle`

### Oracle CDB and PDB Setup

#### CDB Setup

Starting in Oracle 12.2 the database should not come pre-installed with APEX. [Joel Kallman](https://twitter.com/joelkallman) wrote an [article](http://joelkallman.blogspot.ca/2016/03/an-important-change-coming-for-oracle.html) about why this. For some reason the container from Oracle comes with APEX installed in the CDB. It must first be removed.

```bash
# On your laptop:
docker exec -it oracle bash -c "source /home/oracle/.bashrc; bash"

# You should now be in the Oracle Docker container
cd /tmp/apex/5.1.3

sqlplus sys/Oradoc_db1@localhost/orclcdb.localdomain as sysdba

@apxremov.sql
-- Exit sqlplus
exit 

# Exit bash
exit
```

#### PDB Setup

The Oracle container database will come with a default PDB (`ORCLPDB1`). We'll leave it alone and create new PDBs for each version of APEX.

Note I've included the APEX installs in these sections since they're required for the ORDS containers.

##### Create PDBs

On your laptop connect to the database: `sqlcl sys/Oradoc_db1@localhost:32122:orclcdb as sysdba`

```sql
-- Create 5.1.3 PDB
create pluggable database orclpdb513 admin user pdb_adm identified by Oradoc_db1
  file_name_convert=('/u02/app/oracle/oradata/ORCL/pdbseed/','/u02/app/oracle/oradata/ORCL/ORCLPDB513/');

-- Create 5.0.4 PDB
create pluggable database orclpdb504 admin user pdb_adm identified by Oradoc_db1
  file_name_convert=('/u02/app/oracle/oradata/ORCL/pdbseed/','/u02/app/oracle/oradata/ORCL/ORCLPDB504/');

-- Running the following query will show the newly created PDBs but they are not open for Read Write:
select vp.name, vp.open_mode
from v$pdbs vp
;

NAME        OPEN_MODE
PDB$SEED    READ WRITE
ORCLPDB1    READ WRITE
ORCLPDB513  MOUNTED
ORCLPDB504  MOUNTED

-- Open the PDB
alter pluggable database orclpdb513 open read write; 
alter pluggable database orclpdb504 open read write; 

-- If nothing is changed the PDBs won't be loaded on boot.
-- They're a few ways to do this
-- See for reference https://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:9531671900346425939
-- alter pluggable database pdb_name save state;
-- alter pluggable database all save state;
-- alter pluggable database all except pdb$seed open read write
alter pluggable database all save state;

-- Exit sqlcl
exit
```


##### APEX 5.0.4 Install

```bash
# On your laptop
docker exec -it oracle bash -c "source /home/oracle/.bashrc; bash"

# You should now be in the Oracle Docker container
cd /tmp/apex/5.0.4
sqlplus sys/Oradoc_db1@localhost/orclpdb504.localdomain as sysdba
```

```sql
@apexins.sql SYSAUX SYSAUX TEMP /i/

-- APEX REST configuration
-- For APEX 5.0.4 I get an error when running the statement below
-- Just login and re-run
@apex_rest_config_core.sql oracle oracle

-- Required for ORDS install
alter user apex_public_user identified by oracle account unlock;

-- From Joels blog: http://joelkallman.blogspot.ca/2017/05/apex-and-ords-up-and-running-in2-steps.html
declare
    l_acl_path varchar2(4000);
    l_apex_schema varchar2(100);
begin
    for c1 in (select schema
                 from sys.dba_registry
                where comp_id = 'APEX') loop
        l_apex_schema := c1.schema;
    end loop;
    sys.dbms_network_acl_admin.append_host_ace(
        host => '*',
        ace => xs$ace_type(privilege_list => xs$name_list('connect'),
        principal_name => l_apex_schema,
        principal_type => xs_acl.ptype_db));
    commit;
end;
/

-- Setup APEX Admin password
begin
    apex_util.set_security_group_id( 10 );
    apex_util.create_user(
        p_user_name => 'ADMIN',
        p_email_address => 'martin@talkapex.com',
        p_web_password => 'Oradoc_db1',
        p_developer_privs => 'ADMIN',
        p_change_password_on_first_use => 'N');
    apex_util.set_security_group_id( null );
    commit;
end;
/
-- Exit SQL
exit
```

```bash
# You can now exit bash on the container
exit
```

##### APEX 5.1.3 Install


```bash
# On your laptop
docker exec -it oracle bash -c "source /home/oracle/.bashrc; bash"

# You should now be in the Oracle Docker container
cd /tmp/apex/5.1.3
sqlplus sys/Oradoc_db1@localhost/orclpdb513.localdomain as sysdba
```

```sql
@apexins.sql SYSAUX SYSAUX TEMP /i/

-- APEX REST configuration
@apex_rest_config_core.sql oracle oracle

-- Required for ORDS install
alter user apex_public_user identified by oracle account unlock;

-- From Joels blog: http://joelkallman.blogspot.ca/2017/05/apex-and-ords-up-and-running-in2-steps.html
declare
    l_acl_path varchar2(4000);
    l_apex_schema varchar2(100);
begin
    for c1 in (select schema
                 from sys.dba_registry
                where comp_id = 'APEX') loop
        l_apex_schema := c1.schema;
    end loop;
    sys.dbms_network_acl_admin.append_host_ace(
        host => '*',
        ace => xs$ace_type(privilege_list => xs$name_list('connect'),
        principal_name => l_apex_schema,
        principal_type => xs_acl.ptype_db));
    commit;
end;
/

-- Setup APEX Admin password
begin
    apex_util.set_security_group_id( 10 );
    apex_util.create_user(
        p_user_name => 'ADMIN',
        p_email_address => 'martin@talkapex.com',
        p_web_password => 'Oradoc_db1',
        p_developer_privs => 'ADMIN',
        p_change_password_on_first_use => 'N');
    apex_util.set_security_group_id( null );
    commit;
end;
/
-- Exit SQL
exit
```

```bash
# You can now exit bash on the container
exit
```

### ORDS Containers

Each APEX PDB should have an associated ORDS container. A few notes about each ORDS container:

- Instead of naming the ORDS containers with their version number, they'll be named to reference the corresponding APEX version. You may want to alter your naming scheme if you plan to test with multiple versions of ORDS.
- **Important:** Refer to [Docker ORDS](https://github.com/martindsouza/docker-ords) documentation on how to run each ORDS setup. Only modifications will be mentioned in each section below.

#### ORDS 5.0.4

```bash
docker run ...
  --name ords-504 \
  ... 
  --network=oracle_network \
  -e TZ=America/Edmonton \
  -e DB_HOSTNAME=oracle \
  -e DB_PORT=1521 \
  -e DB_SERVICENAME=orclpdb504.localdomain \
  -e APEX_PUBLIC_USER_PASS=oracle \
  -e APEX_LISTENER_PASS=oracle \
  -e APEX_REST_PASS=oracle \
  -e ORDS_PASS=oracle \
  -e SYS_PASS=Oradoc_db1 \
  ...
```

You should now be able to go to APEX via http://localhost:32504/ords


#### ORDS 5.1.3

```bash
docker run ...
  --name ords-513 \
  --network=oracle_network \
  -e TZ=America/Edmonton \
  -e DB_HOSTNAME=oracle \
  -e DB_PORT=1521 \
  -e DB_SERVICENAME=orclpdb513.localdomain \
  -e APEX_PUBLIC_USER_PASS=oracle \
  -e APEX_LISTENER_PASS=oracle \
  -e APEX_REST_PASS=oracle \
  -e ORDS_PASS=oracle \
  -e SYS_PASS=Oradoc_db1 \
  ...
```

You should now be able to go to APEX via http://localhost:32513/ords

#### ORDS Container Wrappup

When running ORDS containers for the first time they'll run in the foreground as the `-d` (`detached`) option was not provided (see [Docker documentation](https://docs.docker.com/engine/reference/run/#detached--d) for more info on this option). It's good to have it run in the foreground as it's easy to spot any issues with ORDS connecting to the database.

To stop stop the container hit `ctrl+c`. To restart the ORDS container run `docker start ords-513`.

## Useful Commands

This section covers some useful commands for Docker and managing Oracle CDB and PDBs.

### Quick Start

To quickly start and stop the docker containers:

```bash
# Start
docker start oracle
docker start ords-504
docker start ords-513

# Stop
docker stop -t 200 oracle
docker stop -t 200 ords-504
docker stop -t 200 ords-513

```

### Docker

```bash
# Docker stop conainer with timeout (for Oracle to shutdown properly)
docker stop -t 200 oracle

# Docker stop all containers 
docker stop -t 200 $(docker ps)

# Delete all containers
docker rm $(docker ps -aq)

# Delete all images
docker rmi $(docker images -a)

# View docker logs as run is happening where <name> is the name of the container that is running
docker logs <name>


# Docker Network
# List
docker network ls

# See containers on network
docker network inspect oracle_network
```




### Oracle

#### Connection Strings

```bash
sqlcl sys/Oradoc_db1@localhost:32122:orclcdb as sysdba
sqlcl sys/Oradoc_db1@localhost:32122/orclpdb1.localdomain as sysdba
sqlcl sys/Oradoc_db1@localhost:32122/orclpdb504.localdomain as sysdba
sqlcl sys/Oradoc_db1@localhost:32122/orclpdb513.localdomain as sysdba
```

#### DBA

```sql

# Drop PDB
drop pluggable database orclpdb513 including datafiles;
drop pluggable database orclpdb504 including datafiles;

# Create user (as sys)
-- Create account to develop with
define new_user = 'martin'
create user &new_user. identified by &new_user. container = current;
grant connect, resource, create any context to &new_user;
alter user &new_user quota unlimited on users;
```

#### Sample Data

Using SQLcl you can create the `dept` and `emp` table your schema using the following script:

```sql
@https://raw.githubusercontent.com/OraOpenSource/OXAR/master/oracle/emp_dept.sql
```
