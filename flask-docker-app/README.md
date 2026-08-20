# Flask Docker Practice

This project documents my Docker learning from containerising a simple Flask application through to connecting a Flask container to a separate MySQL container using a custom Docker network.

The learning progression is:

```text
Application
    ↓
Dockerfile
    ↓
docker build
    ↓
Docker Image
    ↓
docker run
    ↓
Docker Container
    ↓
Docker Network
    ↓
Multiple Containers Communicating
```

---

# Part 1 - Containerising the Flask Application

## 1. Flask Application

A basic Flask application was first tested locally before Docker was introduced.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from my practice container!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)
```

`0.0.0.0` allows Flask to listen on all available network interfaces, which is important when the application runs inside a container.

---

## 2. Testing Flask Locally

Create a Python virtual environment:

```bash
python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

Install Flask:

```bash
pip install flask
```

Run the application:

```bash
python app.py
```

At this stage Docker is not involved:

```text
Host Machine
    ↓
Python Virtual Environment
    ↓
Flask
    ↓
app.py
    ↓
localhost:5001
```

To leave the virtual environment:

```bash
deactivate
```

The `.venv` directory should not be committed to Git.

---

## 3. Dockerfile Fundamentals

The initial Dockerfile was:

```dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY . .
RUN pip install flask
EXPOSE 5001
CMD ["python", "app.py"]
```

### FROM

```dockerfile
FROM python:3.8-slim
```

Defines the base image. The container gets its own Python runtime rather than depending on the host machine's Python installation.

### WORKDIR

```dockerfile
WORKDIR /app
```

Sets `/app` as the working directory inside the image. Following Dockerfile instructions operate relative to this directory.

### COPY

```dockerfile
COPY . .
```

Copies the files from the current Docker build context into `/app` inside the image.

### RUN

```dockerfile
RUN pip install flask
```

Executes while the image is being built and installs Flask inside the image.

```text
RUN = executes while BUILDING the image
```

### EXPOSE

```dockerfile
EXPOSE 5001
```

Documents the port the application is intended to listen on inside the container. It does not publish the port to the host by itself.

### CMD

```dockerfile
CMD ["python", "app.py"]
```

Defines the default command that runs when a container starts.

```text
RUN → executes during image build
CMD → executes when the container starts
```

---

## 4. Building and Running the Image

Build the image:

```bash
docker build -t flask-docker-app .
```

View images:

```bash
docker images
```

Run the container:

```bash
docker run -d -p 5001:5001 --name flask-docker-app flask-docker-app
```

Port mapping syntax:

```text
-p HOST_PORT:CONTAINER_PORT
```

Therefore:

```text
localhost:5001
      ↓
Host port 5001
      ↓
Docker port mapping
      ↓
Container port 5001
      ↓
Flask
```

View running containers:

```bash
docker ps
```

The core Docker lifecycle is:

```text
Dockerfile
    ↓
docker build
    ↓
IMAGE
    ↓
docker run
    ↓
CONTAINER
```

---

# Part 2 - Docker Networking: Flask + MySQL

The next exercise expanded the application from one container to two containers that communicate with each other.

The goal was:

```text
Browser
   ↓
localhost:5002
   ↓
Flask Container (myapp)
   ↓
Custom Docker Network
   ↓
MySQL Container (mydb)
```

Instead of Flask simply returning a static message, Flask connects to MySQL, asks MySQL for its version and displays the result in the browser.

---

## 5. Docker Networking Basics

Docker networking allows containers to communicate with:

- Other containers
- The host machine
- External networks

Two useful network modes introduced were bridge and host.

### Bridge Network

Bridge networking keeps containers on a Docker-managed network. A user-defined bridge network is useful when multiple containers need to communicate.

```text
        Docker Bridge Network
┌─────────────────────────────┐
│                             │
│   Flask  ─────────→ MySQL   │
│                             │
└─────────────────────────────┘
```

Containers on the same user-defined network can communicate using container names rather than hard-coded IP addresses.

### Host Network

Host networking allows a container to use the host's network stack more directly. This provides less network isolation and can make port conflicts more likely.

For normal multi-container applications, user-defined bridge networks are particularly useful.

---

## 6. Creating a Custom Docker Network

Create the network:

```bash
docker network create my-custom-network
```

This creates a network that both application containers can join.

Conceptually:

```text
my-custom-network
│
├── myapp   (Flask)
│
└── mydb    (MySQL)
```

View Docker networks with:

```bash
docker network ls
```

---

## 7. Starting the MySQL Container

The course originally demonstrated MySQL 5.7, but on the Apple Silicon Mac that image did not provide a compatible `linux/arm64/v8` manifest, so MySQL 8 was used instead.

Run MySQL:

```bash
docker run -d \
  --name mydb \
  --network my-custom-network \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  mysql:8
```

### Command Breakdown

```text
docker run
→ Create and start a container

-d
→ Run in detached/background mode

--name mydb
→ Name the container "mydb"

--network my-custom-network
→ Attach MySQL to the custom Docker network

-e MYSQL_ROOT_PASSWORD=my-secret-pw
→ Set the MySQL root password as an environment variable

mysql:8
→ Use the MySQL 8 image
```

The MySQL container listens internally on port `3306`.

It does not need to be published to the Mac just for the Flask container to access it because both containers communicate through the Docker network.

---

## 8. Updating Flask to Connect to MySQL

The Flask application was changed to:

```python
from flask import Flask
import MySQLdb

app = Flask(__name__)

@app.route('/')
def hello_world():
    db = MySQLdb.connect(
        host="mydb",
        user="root",
        passwd="my-secret-pw",
        db="mysql"
    )

    cur = db.cursor()
    cur.execute("SELECT VERSION()")
    version = cur.fetchone()

    return f'Hello, World! MySQL version: {version[0]}'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5002)
```

The most important line for Docker networking is:

```python
host="mydb"
```

`mydb` is the name of the MySQL container.

Because both containers are connected to `my-custom-network`, Docker's internal DNS can resolve the container name.

Conceptually:

```text
Flask asks for "mydb"
        ↓
Docker DNS
        ↓
Find container named mydb
        ↓
MySQL container :3306
```

This means there is no need to hard-code a changing container IP address such as `172.x.x.x`.

---

## 9. Adding the MySQL Python Client

The Flask application now contains:

```python
import MySQLdb
```

The `MySQLdb` module is provided by the Python `mysqlclient` package.

The original Dockerfile only installed Flask:

```dockerfile
RUN pip install flask
```

It therefore needed to become:

```dockerfile
RUN pip install flask mysqlclient
```

However, the first build failed because `python:3.8-slim` is intentionally minimal and did not contain the Linux development tools/libraries required to build `mysqlclient`.

The error included:

```text
pkg-config: not found
Can not find valid pkg-config name
```

This demonstrated an important Docker concept:

> Python packages can depend on operating-system-level packages. Those dependencies must also exist inside the Docker image.

---

## 10. Fixing the mysqlclient Build

The Dockerfile was updated to install the required system dependencies first:

```dockerfile
FROM python:3.8-slim

WORKDIR /app

COPY . .

RUN apt-get update && apt-get install -y \
    pkg-config \
    default-libmysqlclient-dev \
    build-essential

RUN pip install flask mysqlclient

EXPOSE 5002

CMD ["python", "app.py"]
```

### What the new packages do

`pkg-config`

Helps build tools locate installed libraries and their compiler/linker settings.

`default-libmysqlclient-dev`

Provides development files needed to build software that communicates with MySQL-compatible databases.

`build-essential`

Provides common Linux compilation tools required when Python packages need native code compiled during installation.

The exact package names are less important than the lesson:

```text
Python dependency
      ↓
May require OS dependency
      ↓
Install OS dependency in Dockerfile
      ↓
pip install succeeds
```

---

## 11. Building the Flask + MySQL Image

The updated application image was built with:

```bash
docker build -t flask-docker-app-mysql .
```

The resulting image was:

```text
flask-docker-app-mysql:latest
```

Remember:

```text
Dockerfile
    ↓
docker build
    ↓
Image
```

The image must successfully exist before `docker run` can create a container from it.

---

## 12. Running the Flask Container on the Network

Run Flask with:

```bash
docker run -d \
  --name myapp \
  --network my-custom-network \
  -p 5002:5002 \
  flask-docker-app-mysql
```

### Command Breakdown

```text
--name myapp
→ Name the Flask container "myapp"

--network my-custom-network
→ Put Flask on the same network as MySQL

-p 5002:5002
→ Host port 5002 maps to Flask container port 5002

flask-docker-app-mysql
→ Image used to create the Flask container
```

The two containers are now connected through the same Docker network:

```text
                    Host Machine
                         │
                  localhost:5002
                         │
                         ▼
                ┌────────────────┐
                │     myapp      │
                │ Flask + Python │
                │     :5002      │
                └───────┬────────┘
                        │
                        │ host="mydb"
                        │
                my-custom-network
                        │
                        ▼
                ┌────────────────┐
                │      mydb      │
                │    MySQL 8     │
                │     :3306      │
                └────────────────┘
```

---

## 13. Verifying the Containers

Check running containers:

```bash
docker ps
```

The important containers should be:

```text
myapp → flask-docker-app-mysql
mydb  → mysql:8
```

Opening:

```text
http://localhost:5002
```

should return a message containing the MySQL version.

The request flow is:

```text
Browser
   ↓
localhost:5002
   ↓
Host port 5002
   ↓
myapp :5002
   ↓
Python executes MySQL query
   ↓
Docker resolves "mydb"
   ↓
my-custom-network
   ↓
mydb :3306
   ↓
MySQL returns version
   ↓
Flask returns response
   ↓
Browser
```

---

# Troubleshooting Lessons

## Image Not Found

An attempted command used:

```bash
docker run ... flask-docker-app-mysql
```

before the image had successfully built.

Docker therefore reported that it could not find the image locally and attempted to pull it from a registry.

Lesson:

```text
docker build must succeed
        ↓
Image exists locally
        ↓
docker run can use it
```

Check available images with:

```bash
docker images
```

---

## Command Typo

A typo accidentally joined the network name and `-p` option:

```text
--network my-custom-networkv-p 5002:5002
```

Docker then interpreted the command incorrectly and treated `5002:5002` as an image name.

Correct structure:

```bash
docker run -d --name myapp --network my-custom-network -p 5002:5002 flask-docker-app-mysql
```

Lesson: Docker CLI arguments are positional enough that a missing space can change how later arguments are interpreted.

---

## MySQL 5.7 on Apple Silicon

The attempted MySQL 5.7 container failed with:

```text
no matching manifest for linux/arm64/v8
```

MySQL 8 was used for this exercise instead:

```bash
docker run -d --name mydb --network my-custom-network -e MYSQL_ROOT_PASSWORD=my-secret-pw mysql:8
```

---

# Useful Commands

```bash
# Build the basic Flask image
docker build -t flask-docker-app .

# View images
docker images

# View running containers
docker ps

# View all containers
docker ps -a

# Create a custom network
docker network create my-custom-network

# View Docker networks
docker network ls

# Start MySQL on the custom network
docker run -d --name mydb --network my-custom-network -e MYSQL_ROOT_PASSWORD=my-secret-pw mysql:8

# Build the Flask + MySQL image
docker build -t flask-docker-app-mysql .

# Start Flask on the same network
docker run -d --name myapp --network my-custom-network -p 5002:5002 flask-docker-app-mysql

# Stop the containers
docker stop myapp mydb

# Start them again
docker start mydb myapp

# Remove stopped containers
docker rm myapp mydb

# Inspect a network and its connected containers
docker network inspect my-custom-network

# Remove a network when it is no longer needed
docker network rm my-custom-network
```

---

# Key Takeaways

## Docker Lifecycle

```text
Dockerfile
    ↓
docker build
    ↓
IMAGE
    ↓
docker run
    ↓
CONTAINER
```

## Docker Networking

```text
CONTAINER A
     │
     │
DOCKER NETWORK
     │
     │
CONTAINER B
```

## Full Application

```text
SOURCE CODE
     ↓
DOCKERFILE
     ↓
   IMAGE
     ↓
FLASK CONTAINER
     │
     │ Docker Network
     ↓
MYSQL CONTAINER
```

## Container Names

Containers on the same user-defined Docker network can communicate using names:

```text
myapp → host="mydb" → Docker DNS → mydb
```

This avoids relying on changing container IP addresses.

## Port Publishing

```text
-p HOST_PORT:CONTAINER_PORT
```

The Flask container needs port `5002` published so the browser on the host can access it.

The MySQL port does not need to be published to the host for Flask to communicate with MySQL through the Docker network.

## Dependencies

```text
Application dependency
        ↓
Python package
        ↓
May require Linux/system package
        ↓
Dockerfile must provide both
```

## Biggest Lesson From the Networking Exercise

> Containers are isolated processes, but Docker networks allow them to form a multi-container application. Instead of hard-coding IP addresses, containers on a user-defined network can discover each other by name.

The progression so far is therefore:

```text
Single application
      ↓
Docker image
      ↓
Single container
      ↓
Port mapping
      ↓
Docker networking
      ↓
Multiple communicating containers
```
