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
### Taking and restoring backups
#### Take a backup
```sh
# Navigate to home directory as a non-root uer
# (so that the dynamic volume mount works)
cd ~

docker run \
  --net=catalogue_default \
  -v /home/$USER:/mongo-bak \
  --rm \
  mongo:5.0.3 \ # NOTE - use the same Image version as the container was built from
  sh -c \
  "mongodump \
    --uri=<mongo URI> \
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
  --net=catalogue_default \
  -v /home/$USER:/mongo-bak \
  --rm \
  mongo:5.0.3 \ # NOTE - use the same Image version as the container was built from
  sh -c \
  "mongorestore \
    --uri=<mongo URI> \
    -u=root \
    -p=<password> \
    --authenticationDatabase=admin \
    --gzip \
    --archive=/mongo-bak/mongo-backup.archive \
    --nsInclude=<database>.*"
```