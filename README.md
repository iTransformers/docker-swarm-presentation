# Docker Swarm Presentation

## Prerequisites 
Install virtualbox on your host

### Get docker machine 
#### Windows 
```
if [[ ! -d "$HOME/bin" ]]; then mkdir -p "$HOME/bin"; fi && \
curl -L https://github.com/docker/machine/releases/download/v0.12.2/docker-machine-Windows-x86_64.exe > "$HOME/bin/docker-machine.exe" && \
chmod +x "$HOME/bin/docker-machine.exe"
```
#### 
```
curl -L https://github.com/docker/machine/releases/download/v0.12.2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
chmod +x /tmp/docker-machine &&
sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```
## Building a docker swarm cluster 
```
for i in 1 2 3; do
    docker-machine create -d virtualbox swarm-$i
done
```

## Verify that your machines are up and running 
```
docker-machine ls
```

The output should be something similar to
```
$ ./docker-machine.exe ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
swarm-1   -        virtualbox   Running   tcp://192.168.99.102:2376           v17.06.2-ce
swarm-2   -        virtualbox   Running   tcp://192.168.99.103:2376           v17.06.2-ce
swarm-3   -        virtualbox   Running   tcp://192.168.99.104:2376           v17.06.2-ce
```

Creating the cluster
```
eval "$(docker-machine env swarm-1)"

docker swarm init --advertise-addr $(docker-machine ip swarm-1)
```

##Connect to your docker machines through ssh 
```
docker-machine ssh swarm-1
```

## Adding the visualizer service
```
eval "$(docker-machine env swarm-1)"
docker service create \
  --name=visualizer \
  --publish=8000:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer
```
Wait a bit and check the status of the visualizer service. Pulling dockersamples/visualizer:latest will speed things up!

```
docker service ls 
```
Once up open in your browser

``` 
echo http://$(docker-machine ip swarm-1):8000
```

## Adding workers to the cluster
```
eval "$(docker-machine env swarm-1)"
JOIN_TOKEN=$(docker swarm join-token -q worker)

for i in 2 3; do
    eval "$(docker-machine env swarm-$i)"

    docker swarm join --token $JOIN_TOKEN \
        --advertise-addr $(docker-machine ip swarm-$i) \
        $(docker-machine ip swarm-1):2377
done
```

```
eval "$(docker-machine env swarm-1)"
docker node ls
```

## Creating network
```
eval "$(docker-machine env swarm-1)"
docker network create -d overlay routing-mesh
```

## Deploy a new service
```
eval "$(docker-machine env swarm-1)"
docker service create \
  --name=docker-routing-mesh \
  --publish=8080:8080/tcp \
  --network routing-mesh \
  --reserve-memory 20m \
  albertogviana/docker-routing-mesh:1.0.0
```

## Testing the service
```
curl http://$(docker-machine ip swarm-1):8080
curl http://$(docker-machine ip swarm-2):8080
curl http://$(docker-machine ip swarm-3):8080
```

## Scaling a service
```
docker service scale docker-routing-mesh=3
```

## Calling a service
```
while true; do curl http://$(docker-machine ip swarm-1):8080; sleep 1; echo "\n";  done
```

## Rolling updates
```
eval "$(docker-machine env swarm-1)"
docker service update \
  --update-failure-action pause \
  --update-parallelism 1 \
  --image albertogviana/docker-routing-mesh:2.0.0 \
  docker-routing-mesh
```

## Calling a service
```
while true; do curl http://$(docker-machine ip swarm-1):8080/health; sleep 1; echo "\n";  done
```

## Docker secret create
```
echo test | docker secret create my_secret -
```

## Docker secret ls
```
docker secret ls
```

## Deploy my secret
```
eval "$(docker-machine env swarm-1)"
docker service update \
  --update-failure-action pause \
  --update-parallelism 1 \
  --secret-add my_secret \
  --image albertogviana/docker-routing-mesh:2.0.0 \
  docker-routing-mesh
```

## Drain a node
```
docker node update --availability=drain swarm-3
```

## Listing nodes
```
docker node ls
```

## Bring the node back
```
docker node update --availability=active swarm-3
```

## Docker system info
```
docker system info
```

## Docker and disk?
```
docker system df
```

## Docker clean up
```
docker system prune
```
