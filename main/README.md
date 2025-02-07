# phpinfo
## Sample PHP application to help learning about Docker containers and Kubernetes orchestration

[![CI](https://github.com/sebastian-colomar/phpinfo/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/sebastian-colomar/phpinfo/actions/workflows/ci.yaml)
## The testing environment
We can use any available Linux shell to run the exercises.
In our case we are going to use Google Cloud Shell:
- https://shell.cloud.google.com

## The application
The application is very simple.
It is a PHP web server that will publish a file containing PHP code.
The PHP code is also very simple: `phpinfo()`.
```
echo '<?php phpinfo();?>' | tee index.php
```
```
php -f index.php -S 0.0.0.0:8080
```
```
curl -Is localhost:8080/index.php
```

## The container
In order to containerize the application we can simply run the following command:
```
docker run --cpus 0.01 --memory 10M --memory-reservation 10M --name phpinfo --publish 8080 --read-only --restart always --user nobody:nogroup --volume ${PWD}/index.php:/src/index.php:ro --workdir /src/ index.docker.io/library/php:alpine php -f index.php -S 0.0.0.0:8080
```
You can test the application from inside the container running the following command:
```
docker exec phpinfo curl -Is localhost:8080/index.php
```
Or you can test the application from outside the container with the following command:
```
curl -Is localhost:$( docker port phpinfo | cut -d: -f2 )/index.php
```
Or you can just check the container logs:
```
docker logs phpinfo
```
You can remove the container with the following command:
```
docker rm --force phpinfo
```

## The image
The above command will launch a container running an operating system downloaded from the public library hosted at index.docker.io, i.e. Docker Hub.
The exact image you are going to use is called php:alpine, which means that it is based on the basic Linux Alpine operating system with the PHP binaries included.

In case you need to customize this operating system image, you can do it with a Dockerfile. For example:
```
tee Dockerfile 0<<EOF

FROM index.docker.io/library/alpine:latest
RUN  apk add php

EOF
```
The above block will create a Dockerfile with instructions for downloading a basic Alpine Linux operating system with the latest version and adding the PHP binaries.
You can then build the Docker image with the following command:
```
docker build --tag localhost/my_library/my_php:alpine .
```
You can then run the container with a command similar to the one in the previous section:
```
docker run --cpus 0.01 --memory 10M --memory-reservation 10M --name phpinfo --publish 8080 --read-only --restart always --user nobody:nogroup --volume ${PWD}/index.php:/src/index.php:ro --workdir /src/ localhost/my_library/my_php:alpine php -f index.php -S 0.0.0.0:8080
```

## The compose file
We can use a compose file instead of manually creating individual containers with the Docker command line.
The advantages of using compose files are numerous (such as accountability, auditability, collaborative work, transparency, etc.).
Run the following command to create a Docker compose file:
```
tee docker-compose.yaml 0<<EOF

services:
  phpinfo:
    command:
      - php
      - -f
      - index.php
      - -S
      - 0.0.0.0:8080
    cpus: 0.01
    image: index.docker.io/library/php:alpine
    mem_limit: 10M
    mem_reservation: 10M
    ports:
      - 8080
    read_only: true
    restart: always
    scale: 1
    user: nobody:nogroup
    volumes:
      -
        read_only: true
        source: ./index.php
        target: /src/index.php
        type: bind
    working_dir: /src/
version: "2.4"

EOF
```
Once the file has been created, you can deploy the application with the following command:
```
docker-compose up
```
You can check the logs with the following command:
```
docker-compose logs
```
You can display the running processes with the following command:
```
docker-compose top
```
You can test the connection to the web server using the following command:
```
curl -Is localhost:$( docker-compose port phpinfo 8080 | cut -d : -f 2 )/index.php
```
You can remove the application with the following command:
```
docker-compose down
```
## The orchestrator
Although the above method is perfectly suitable for running our application, it will not provide high availability.
If the node is down, our application will be down too.
To avoid that situation, we need a multi-node cluster so that when one node is down, the other node takes the workload of the affected node.
We can easily create a cluster in this lab environment:
- https://labs.play-with-docker.com

You will first need to create a free Docker account in order to login to the Docker Playground:
- https://docker.com

Once you have logged in you will need to create at least two instances clicking in the "ADD NEW INSTANCE" to the left menu of the page.
Next step is to initialize the cluster running the following command on the first instance:
```
docker swarm init --advertise-addr $( hostname -i )
```
This last command will initialize a Docker Swarm cluster on the first instance which will act as a master node.
This will be the output of the previous command:
```
Swarm initialized: current node (xxx) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token xxx-xxx-xxx-xxx x.x.x.x:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
In order to add a worker node to this cluster we need to copy the `docker swarm join` full command and execute in the second instance.
This will be the output of the previous command on the second instance:
```
This node joined a swarm as a worker.
```
We repeat the same operation to add a second worker node to the cluster.

Now we can run the following command on the first master instance in order to check the cluster availability:
```
docker node ls
```
If you want to deploy the application in this highly available cluster you will need to first create a compose file:
```
tee docker-stack.yaml 0<<EOF

configs:
  my_config:
    file: ${PWD}/index.php
services:
  phpinfo:
    command:
      - php
      - -f
      - index.php
      - -S
      - 0.0.0.0:8080
    configs:
      - 
        mode: 0400
        source: my_config
        target: /src/index.php
        uid: '65534'
        gid: '65534'
    deploy:
      placement:
        constraints:
          -  "node.role==worker"
      replicas: 1
      resources:
        limits:
          cpus: '0.01'
          memory: 10M
        reservations:
          cpus: '0.01'
          memory: 10M
    image: index.docker.io/library/php:alpine
    ports:
      - 8080
    read_only: true
    user: nobody:nogroup
    working_dir: /src/
version: "3.8"

EOF
```
Before deploying your app, we need to re-create the PHP code in this new environment:
```
echo '<?php phpinfo();?>' | tee index.php
```
You will now deploy your application to high availability with the following command:
```
docker stack deploy --compose-file docker-stack.yaml phpinfo
```
You can see the result of the deployment with the following commands:
```
docker stack ls
docker stack ps phpinfo
docker stack services phpinfo
```
You can connect to the web server with the following command:
```
curl localhost:$( docker stack services phpinfo | awk /phpinfo_phpinfo/'{ print $6 }' | cut -d: -f2 | cut -d- -f1 )/index.php -Is
```
You can also check the service logs:
```
docker service logs phpinfo_phpinfo
```
You can remove the application with the following command:
```
docker stack rm phpinfo
```
## The Kubernetes orchestrator

We previously used Docker Swarm as the container orchestrator. In this section, we will use Kubernetes instead.

1. Login to the Kubernetes playground with your Docker account and start a new session:

   * https://labs.play-with-k8s.com/
1. Add a new instance and initialize the kubernetes cluster master node:

   ```
   kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16
   ```
1. Initialize the cluster networking:

   ```
   kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
   ```

Now we can deploy our application using this new platform with the following compose file:
```
tee kube-stack.yaml 0<<EOF

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: phpinfo-cm
data:
  index.php: |-
    <?php
      phpinfo();
    ?>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpinfo-deploy
spec:
  selector:
    matchLabels:
      app: phpinfo-deploy
  replicas: 1
  template:
    metadata:
      labels:
        app: phpinfo-deploy
    spec:
      containers:
        - name: phpinfo
          image: index.docker.io/library/php:alpine
          ports:
            - containerPort: 8080
              protocol: TCP
          resources:
            limits:
              cpu: 100m
              memory: 100M
            requests:
              cpu: 10m
              memory: 10M
          env:
            - name: AUTHOR
              value: Sebastian
          securityContext:
              readOnlyRootFilesystem: true
          volumeMounts:
            - mountPath: /src/index.php
              subPath: index.php
              name: phpinfo-volume
              readOnly: true
          workingDir: /src/
          args:
            - php
            - -f
            - index.php
            - -S
            - 0.0.0.0:8080
      restartPolicy: Always
      volumes:
        - name: phpinfo-volume
          configMap:
              defaultMode: 0400
              items:
                - key: index.php
                  path: index.php
                  mode: 0400
              name: phpinfo-cm
---
apiVersion: v1
kind: Service
metadata:
  name: phpinfo-svc
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: phpinfo-deploy
  type: NodePort
---

EOF
```


