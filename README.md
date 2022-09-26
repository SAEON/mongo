# Mongo
SAEON's MongoDB servers

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Local development](#local-development)
- [Server setup](#server-setup)
  - [Setup a limited permissions user called 'runner' and configure SSH login from the github-runner.saeon.int server](#setup-a-limited-permissions-user-called-runner-and-configure-ssh-login-from-the-github-runnersaeonint-server)
  - [Server administration](#server-administration)
- [Deploy](#deploy)
- [Database management](#database-management)
  - [Configure DB Server users](#configure-db-server-users)
  - [Configure backups](#configure-backups)
    - [Take a backup](#take-a-backup)
    - [Restore a backup](#restore-a-backup)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Local development
These easiest way to setup the MongoDB server on a local machine is via a Docker container

```sh
mkdir /home/$USER/mongo

docker network create --driver bridge saeon_local

docker run \
  --name mongo \
  --net=saeon_local \
  --restart always \
  --memory 1.5g \
  --cpus 2 \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  -v /home/$USER/mongo:/data/db \
  -d \
  mongo:5.0.12
```

# Server setup

- [Install Docker Engine](https://docs.docker.com/engine/install/centos/)
- [Init Docker Swarm mode](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/) (using a single node - Swarm mode allows setting service usage limits on CPU and memory)

## Setup a limited permissions user called 'runner' and configure SSH login from the github-runner.saeon.int server
```sh
sudo su
adduser runner
passwd runner # Enter a strong password, and add this password as a repository secret
```

The `runner` user needs to be able to run `docker`, but should not be in the `docker` group

```sh
visudo

# Add this line to the bottom of the visudo file
runner ALL=NOPASSWD: /opt/deploy-docker-stack.sh
```

Create the deploy script `/opt/deploy-docker-stack.sh` with the following content

```sh
#!/bin/sh

echo "Compose file: $1"
echo "Compose env file $2"
echo "Deploying stack: $3"

export $(cat $2) > /dev/null 2>&1;
docker stack deploy -c $1 $3
```

Make sure the script has the correct permissions

```sh
chown root /opt/deploy-docker-stack.sh 
chmod 755 /opt/deploy-docker-stack.sh
```

## Server administration
Add the following to the root crontab
```
0 0 * * 0 docker system prune -f > /opt/docker-system-clean.log 2>&1
```

# Deploy
## Deploy
Push to relevant branch to trigger server deployment, or to `main` branch to update the repo without triggering a deployment 

# Database management
## Configure DB Server users
The root user for the database is configured on deployment. Add databases and users for each application manually

```sh
docker container exec -it 2212 mongosh -u root -p <root password> --authenticationDatabase admin
use admin # Users need to be created in the admin database
db.createUser({user: "<username>", pwd: "<password>", roles: [{role: "dbOwner", db: "<database>"}]})
```

## Server/Database management
### Automate backup process
Add the following to the crontab for each database you want to have backed up

```
# Backup catalogue database daily (00:00) (<database>). NOTE - use the same mongo image as the container
0 0 * * * docker run --net=saeon_local -v /opt/dbak:/dbak --rm mongo:5.0.3 sh -c "mongodump --uri=mongodb://mongo:27017 -u=root -p=<pswd> --authenticationDatabase=admin -d=<database> --archive --gzip > /dbak/<database>_bak_`date +\%Y-\%m-\%d_\%H-\%M-\%S.archive`" 2>&1

# Prune backups older than 90 days
0 0 * * 0 find /opt/dbak/* -mtime +90 -exec rm {} \;

# Prune docker system
0 0 * * 0 docker system prune -f > /opt/docker-system-clean.log 2>&1
```

### Take a backup
```sh
# Navigate to home directory as a non-root uer
# (so that the dynamic volume mount works)
cd ~

docker run \
  --net=saeon_local \
  -v /home/$USER:/mongo-bak \
  --rm \
  <mongo image name>
  sh -c \
  "mongodump \
    --uri=mongodb://<container id>:27017 \
    -u=root \
    -p=<password> \
    --authenticationDatabase=admin \
    -d=<database> \
    --archive \
    --gzip \
    > \
    /mongo-bak/mongo-backup.archive"
```

### Restore a backup
This command assumes a backup taken with the above command
```sh
# Navigate to home directory as a non-root uer
# (so that the dynamic volume mount works)
cd ~

docker run \
  -i \
  --net=saeon_local  \
  -v /home/$USER:/mongo-bak \
  --rm \
  <mongo image name> \
  sh -c \
  "mongorestore \
    --uri=mongodb://<container id>:27017 \
    -u=root \
    -p=<password> \
    --authenticationDatabase=admin \
    --gzip \
    --archive=/mongo-bak/mongo-backup.archive \
    --nsFrom=<from db in bak>.* \
    --nsInclude=<from db in bak>.* \
    --nsTo=<target db>.*" 
```
