# Flask Docker Practice — Part 2: Docker Networking

This part continues from the original Flask Docker exercise and focuses on linking a Flask container to a separate MySQL container through a custom Docker network.

## Goal

```text
Browser
   ↓
localhost:5002
   ↓
Flask Container (myapp)
   ↓
my-custom-network
   ↓
MySQL Container (mydb)
```

The Flask application connects to MySQL, runs `SELECT VERSION()` and displays the MySQL version in the browser.

---

# 1. Create a Custom Docker Network

```bash
docker network create my-custom-network
```

View networks:

```bash
docker network ls
```

The custom network acts like a private network for the containers:

```text
my-custom-network
├── myapp  → Flask
└── mydb   → MySQL
```

Containers on the same user-defined network can communicate using container names instead of hard-coded IP addresses.

---

# 2. Start the MySQL Container

On the Apple Silicon Mac, MySQL 5.7 did not provide a compatible ARM image, so MySQL 8 was used:

```bash
docker run -d \
  --name mydb \
  --network my-custom-network \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  mysql:8
```

### Breakdown

```text
docker run
→ create and start a container

-d
→ detached/background mode

--name mydb
→ call the container mydb

--network my-custom-network
→ connect it to our custom network

-e MYSQL_ROOT_PASSWORD=my-secret-pw
→ create the MySQL root password environment variable

mysql:8
→ use the MySQL 8 image
```

MySQL listens internally on port `3306`. We did not need to publish 3306 to the Mac because Flask communicates with MySQL through the Docker network.

---

# 3. Update Flask to Talk to MySQL

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

The key line is:

```python
host="mydb"
```

`mydb` is the MySQL container name. Docker DNS resolves that name because both containers are on `my-custom-network`.

```text
Flask asks for "mydb"
        ↓
Docker DNS
        ↓
mydb container
        ↓
MySQL :3306
```

---

# 4. Add the MySQL Python Dependency

The new Python code imports:

```python
import MySQLdb
```

This comes from the `mysqlclient` Python package, so the Docker image needs more than Flask.

Initially:

```dockerfile
RUN pip install flask mysqlclient
```

failed because `python:3.8-slim` is deliberately minimal. The build reported that `pkg-config` was missing.

This taught an important lesson:

```text
Python package
      ↓
may require OS-level dependencies
      ↓
those dependencies must also be installed in the image
```

---

# 5. Updated Dockerfile

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

The added Linux packages allow `mysqlclient` to build successfully.

- `pkg-config` helps locate installed libraries.
- `default-libmysqlclient-dev` provides MySQL client development files.
- `build-essential` provides compilation tools.

The important lesson is not memorising these package names; it is understanding that application dependencies can themselves have system dependencies.

---

# 6. Build the New Image

```bash
docker build -t flask-docker-app-mysql .
```

Check it exists:

```bash
docker images
```

Expected image:

```text
flask-docker-app-mysql:latest
```

Remember:

```text
Dockerfile
    ↓
docker build
    ↓
IMAGE
```

The build must succeed before `docker run` can create a container from that image.

---

# 7. Run Flask on the Same Network

```bash
docker run -d \
  --name myapp \
  --network my-custom-network \
  -p 5002:5002 \
  flask-docker-app-mysql
```

### Breakdown

```text
--name myapp
→ name the Flask container

--network my-custom-network
→ connect Flask to the same network as MySQL

-p 5002:5002
→ Mac/host port 5002 → Flask container port 5002

flask-docker-app-mysql
→ image used to create the container
```

Now the architecture is:

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

# 8. Verify Everything

```bash
docker ps
```

The important running containers are:

```text
myapp → flask-docker-app-mysql
mydb  → mysql:8
```

Open:

```text
http://localhost:5002
```

The page should return a message containing the MySQL version.

Full request flow:

```text
Browser
   ↓
localhost:5002
   ↓
myapp :5002
   ↓
Python executes SELECT VERSION()
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

# Errors and Troubleshooting

## Image Not Found

Trying to run `flask-docker-app-mysql` before it had successfully built caused Docker to report that it could not find the image locally and then try to pull it from a registry.

Lesson:

```text
docker build succeeds
        ↓
image exists locally
        ↓
docker run can use it
```

## Command Typo

This typo joined the network name and `-p` option:

```text
--network my-custom-networkv-p 5002:5002
```

Docker then interpreted `5002:5002` incorrectly as an image name.

Correct command:

```bash
docker run -d --name myapp --network my-custom-network -p 5002:5002 flask-docker-app-mysql
```

## MySQL 5.7 on Apple Silicon

The MySQL 5.7 attempt returned:

```text
no matching manifest for linux/arm64/v8
```

Using `mysql:8` solved this for the exercise.

---

# Useful Commands

```bash
# Create network
docker network create my-custom-network

# View networks
docker network ls

# Start MySQL
docker run -d --name mydb --network my-custom-network -e MYSQL_ROOT_PASSWORD=my-secret-pw mysql:8

# Build Flask + MySQL image
docker build -t flask-docker-app-mysql .

# View images
docker images

# Start Flask on the network
docker run -d --name myapp --network my-custom-network -p 5002:5002 flask-docker-app-mysql

# View running containers
docker ps

# View all containers
docker ps -a

# Inspect the custom network
docker network inspect my-custom-network

# Stop containers
docker stop myapp mydb

# Start them again
docker start mydb myapp

# Remove stopped containers
docker rm myapp mydb

# Remove the network when no longer needed
docker network rm my-custom-network
```

---

# Key Takeaways

The original exercise taught:

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

Part 2 adds networking:

```text
CONTAINER A
     │
     │
DOCKER NETWORK
     │
     │
CONTAINER B
```

Together:

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

The biggest lesson from this exercise:

> Containers are isolated, but Docker networks allow them to communicate and form a multi-container application. Containers on the same user-defined network can discover each other by name instead of relying on hard-coded IP addresses.