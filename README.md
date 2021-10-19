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

## Configure root user
Configure the root user for the Mongo server

```sh
docker container exec -it <container name> mongo
use admin
db.createUser({user: "<user name>", pwd: "pwd", roles: ["root"]})
```