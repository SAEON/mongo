# Mongo
SAEON's MongoDB servers

## Server setup

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

Create the deploy script `/opt-deploy-docker-stack.sh` with the following content

```sh
#!/bin/sh
echo "Deploying stack: $2"
echo "Compose file: $1"

export $(cat docker-compose.env) > /dev/null 2>&1;
docker stack deploy -c $1 $2
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

## Deploy
Push to relevant branch to trigger server deployment, or to `main` branch to update the repo without triggering a deployment 

## Configure DB Server users
The root user for the database is configured on deployment. Add databases and users for each application manually

```sh
docker container exec -it 2212 mongosh -u root -p <root password> --authenticationDatabase admin
use admin # Users need to be created in the admin database
db.createUser({user: "<username>", pwd: "<password>", roles: [{role: "dbOwner", db: "<database>"}]})
```

## Database management
### Automate backup process
Add the following to the crontab for each database you want to have backed up

```
# Backup catalogue database daily (00:00) (<database>). NOTE - use the same mongo image as the container
0 0 * * * docker run --net=<Docker network> -v /opt/dbak:/dbak --rm mongo:5.0.2 sh -c "mongodump --uri=mongodb://mongo:27017 -u=root -p=<pswd> --authenticationDatabase=admin -d=<database> --archive --gzip > /dbak/<database>_bak_`date +\%Y-\%m-\%d_\%H-\%M-\%S.archive`" 2>&1

# Prune backups older than 90 days
0 0 * * 0 find /opt/dbak/ -mtime + 90 -type -f -delete
```

### Taking and restoring backups
#### Take a backup
```sh
# Navigate to home directory as a non-root uer
# (so that the dynamic volume mount works)
cd ~

docker run \
  --net=<mongo container network> \
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

#### Restore a backup
This command assumes a backup taken with the above command
```sh
# Navigate to home directory as a non-root uer
# (so that the dynamic volume mount works)
cd ~

docker run \
  -i \
  --net=<mongo container network>  \
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