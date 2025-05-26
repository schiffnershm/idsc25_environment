# IDSC 2025 Workshop Test Environment

## Getting Started

To follow along the examples of the workshop you need the test environment running.

### 1. Prerequisites

#### Install Docker (if not already installed)
- [Docker](https://www.docker.com/products/docker-desktop) 

#### Build Simulator Docker Image

A. Clone the Simulator project
- [Simulator](https://github.com/schiffnershm/idsc25_simulator)

B. Create a docker image of the Simulator.

Execute the following command in the root folder of the Simulator project, containing the __Dockerfile__. 
```shell
docker build -t idsc-iot-simulator .
```
---

### 2. Starting the Test Environment

Switch to the `docker` folder:

```shell
cd docker
```

Start all services (Flink, Redpanda, Console, etc.) with a custom project name:

```shell
docker compose -p idsc2025 up -d
```

Check that all containers are running:

```shell
docker compose -p idsc2025 ps
```

To stop and remove all containers, networks, and volumes:

```shell
docker compose -p idsc2025 down
```

---

### 3. Using the Flink SQL Client

To open an interactive Flink SQL Client shell:

```shell
docker compose -p idsc2025 run sql-client
```

You can now execute Flink SQL statements against your running environment.

---

### 4. Accessing the Redpanda Console

After starting the environment, open your browser and go to:

```
http://localhost:8082
```

This gives you a web UI to inspect topics, messages, and schemas in Redpanda.

---

### 5. Accessing the Flink Dashboard

After starting the environment, open your browser and go to:

```
http://localhost:8081
```

This gives you a web UI to inspect your Flink cluster and jobs.

---

## Additional Notes

- All SQL examples and workshop materials are in the `examples-sql` folder.
- For troubleshooting, check container logs with:

  ```shell
  docker compose -p idsc2025 logs <service-name>
  ```

- You can customize the project name by changing the `-p idsc2025` flag.

---

## Author

Stephan Schiffner  
IDSC 2025, Salzburg



