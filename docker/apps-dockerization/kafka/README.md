## Kafka with Zookeeper Setup
### Step-1: Write compose file
```dockerfile
version: '3.1'

services:
  zookeeper:
    image: wurstmeister/zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"

  kafka:
    image: wurstmeister/kafka
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```
### Step-2:  Run docker compose
```shell
$ docker compose -f docker-compose.yaml up -d
```
Here is the breakdown of exactly what each part of `docker compose -f docker-compose.yaml up -d` does:

* **`docker compose`**: This invokes the Docker Compose tool, which manages multi-container Docker applications defined in configuration files.
* **`-f docker-compose.yaml`**: The `-f` (or `--file`) flag tells Docker to use a specific file for configuration. While `docker-compose.yaml` is the default name it looks for anyway, explicitly adding this flag is useful if you have custom file names (like `docker-compose.prod.yaml`).
* **`up`**: This is the core action command. It tells Docker to look inside the specified file, build/pull the required container images, create the networks and volumes, and start the containers.
* **`-d`**: Short for "detached" mode. This runs the containers in the background of your terminal. Without `-d`, your terminal would lock up showing live streaming logs, and closing the terminal window would stop your Kafka cluster.

#### What happens behind the scenes when you run it?
1. **Image check**: Docker reads your file and checks if the `apache/kafka` image is already downloaded on your machine. If not, it downloads it automatically.
2. **Network setup**: It configures the internal virtual network so the containers can communicate safely.
3. **Port mapping**: It routes port `9092` from inside the container to your host machine (`localhost`).
4. **Initialization**: It starts Kafka, assigns it a container name, and returns control of the command line back to you.

## Kafka UI Setup
 Run below command to pull and up Kafka-UI
```shell
# Download the Kafka UI image configured in the Compose files.
docker compose -f docker-compose.yaml -f docker-compose-kafka-ui.yaml pull kafka-ui

# Recreate Kafka UI and run it in the background.
docker compose -f docker-compose.yaml -f docker-compose-kafka-ui.yaml up -d --force-recreate kafka-ui

# Show Kafka UI's container status, image, and published ports.
docker compose -f docker-compose.yaml -f docker-compose-kafka-ui.yaml ps kafka-ui
```
### Kafka-UI URL
#### If you are using WSL
If Kafka is running inside WSL then it won't be accessible on localhost. So you need to run below command to get the private network address assigned to your WSL environment.
```shell
$ hostname -I
```
```shell
# Output as below
172.26.252.179 192.168.49.1 172.17.0.1 172.18.0.1
```
So the Kafka UI is accessible on 
```shell
http://172.26.252.179:8080/
```

#### Without WSL
If you are not using WSL so the Kafka UI can be accessed via below URL
```shell
http://localhost:8080/
```