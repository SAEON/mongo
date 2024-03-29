# Mongo
SAEON's MongoDB instances

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Local development](#local-development)
- [Server setup](#server-setup)
  - [Deploy](#deploy)
- [Database management](#database-management)
  - [Add users](#add-users)
  - [Take a backup](#take-a-backup)
    - [Automate backup process](#automate-backup-process)
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
  --restart unless-stopped \
  --memory 1.5g \
  --cpus 2 \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  -v /home/$USER/mongo:/data/db \
  -d \
  mongo:latest
```

# Server setup
Refer to [server-deployment documentation](https://github.com/SAEON/deployment-platform#mongosaeonint)

## Deploy
Deploy via GitHub Actions workflow_dispatch triggers

# Database management
## Add users
The root user for the database is configured on deployment. Add databases and users for each application manually

```sh
# Log into the containerized database
docker \
  container \
  exec \
  -it \
  <container id> \
    mongosh \
      -u root \
      -p <password> \
      --authenticationDatabase admin

# Switch to the admin context (users must be created in this DB)
use admin

# Run the create user command
db.createUser({user: "<username>", pwd: "<password>", roles: [{role: "dbOwner", db: "<database>"}]})
```

## Take a backup
```sh
# Navigate to home directory as a non-root uer
# (so that the dynamic volume mount works)
cd ~
IMAGE=mongo:latest
docker run \
  --net=saeon_local \
  -v /home/$USER:/mongo-bak \
  --rm \
  $IMAGE \
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

### Automate backup process
Add the following to the crontab for each database you want to have backed up

```
# Backup catalogue database daily (00:00) (<database>). NOTE - use the same mongo image as the container
0 0 * * * docker run --net=saeon_local -v /opt/dbak:/dbak --rm mongo:latest sh -c "mongodump --uri=mongodb://mongo:27017 -u=root -p=<pswd> --authenticationDatabase=admin -d=<database> --archive --gzip > /dbak/<database>_bak_`date +\%Y-\%m-\%d_\%H-\%M-\%S.archive`" 2>&1

# Prune backups older than 90 days
0 0 * * 0 find /opt/dbak/* -mtime +90 -exec rm {} \;
```

## Restore a backup
This command assumes a backup taken with the backup command above
```sh
# Navigate to home directory as a non-root uer
# (so that the dynamic volume mount works)
cd ~
IMAGE=mongo:latest
docker run \
  -i \
  --net=saeon_local  \
  -v /home/$USER:/mongo-bak \
  --rm \
  $IMAGE \
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
