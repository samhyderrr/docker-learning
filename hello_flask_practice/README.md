# Flask Docker Practice

This project was created as part of my Docker learning to understand the full process of taking a simple application and running it inside a Docker container.

The main goal was to understand the following workflow:

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
Application Running
```

---

## What I Built

I created a simple Python Flask web application that displays a message in the browser.

The project contains:

```text
hello_flask_practice/
├── app.py
├── Dockerfile
└── README.md
```

The application runs on port `5000`.

---

# 1. Creating the Flask Application

The Flask application is stored in `app.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from my practice container!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### What the code does

- `from flask import Flask`
  - Imports Flask into the Python application.

- `app = Flask(__name__)`
  - Creates the Flask application.

- `@app.route('/')`
  - Defines what happens when someone visits the root `/` URL.

- `return 'Hello from my practice container!'`
  - Returns the message displayed in the browser.

- `app.run(host='0.0.0.0', port=5000)`
  - Starts the Flask web server on port `5000`.
  - `0.0.0.0` allows the application to listen on all available network interfaces, which is important when running the application inside a container.

---

# 2. Running Flask Locally

Before containerising the application, I tested it directly on my machine.

I created a Python virtual environment:

```bash
python3 -m venv .venv
```

Activated it:

```bash
source .venv/bin/activate
```

Installed Flask:

```bash
pip install flask
```

Then ran the application:

```bash
python app.py
```

The application could then be accessed at:

```text
http://localhost:5000
```

At this stage, Docker was not involved.

The application was running using:

```text
Host Machine
    ↓
Python Virtual Environment
    ↓
Flask
    ↓
app.py
    ↓
localhost:5000
```

To leave the virtual environment:

```bash
deactivate
```

---

# 3. Why Use a Python Virtual Environment?

A virtual environment keeps project-specific Python packages separate from the system Python installation.

For example:

```text
System Python
│
├── Project A
│   └── .venv
│       └── Flask version A
│
└── Project B
    └── .venv
        └── Flask version B
```

This prevents different projects from interfering with each other's dependencies.

The `.venv` directory should not be committed to Git.

It can be excluded using:

```gitignore
.venv/
```

---

# 4. Creating the Dockerfile

I then created a `Dockerfile` to describe how Docker should package and run the application.

```dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY . .
RUN pip install flask
EXPOSE 5000
CMD ["python", "app.py"]
```

A Dockerfile is essentially a set of instructions used to build a Docker image.

---

## FROM

```dockerfile
FROM python:3.8-slim
```

Defines the base image.

In this project, the application uses a lightweight image containing Python 3.8.

This means the container can use Python 3.8 regardless of which Python version is installed on the host machine.

For example:

```text
Host Machine → Python 3.14

Container A → Python 3.8
Container B → Python 3.11
Container C → Python 3.12
```

This is one of the major benefits of containerisation: applications can have isolated environments and dependencies.

---

## WORKDIR

```dockerfile
WORKDIR /app
```

Sets `/app` as the working directory inside the Docker image.

This is conceptually similar to:

```bash
cd /app
```

Any following instructions operate relative to this directory.

The `/app` directory exists inside the Docker environment and is separate from the project directory on the host machine.

---

## COPY

```dockerfile
COPY . .
```

Copies files from the build context into the Docker image.

The syntax can be thought of as:

```text
COPY source destination
```

In this example:

```text
COPY . .
     ↑ ↑
     │ └── Destination: current Docker working directory (/app)
     │
     └── Source: current build context
```

This copies the application files into `/app` inside the image.

---

## RUN

```dockerfile
RUN pip install flask
```

`RUN` executes a command while the Docker image is being built.

In this project, it installs Flask inside the Docker image.

This means the container has its own Flask installation and does not depend on Flask being installed on the host machine.

### Important

```text
RUN = executed while BUILDING the image
```

It would therefore not make sense to use:

```dockerfile
RUN python app.py
```

to start the application because the application should start when the container runs, not while the image is being built.

---

## EXPOSE

```dockerfile
EXPOSE 5000
```

Documents that the application inside the container is intended to listen on port `5000`.

`EXPOSE` does not by itself make the application accessible through the host machine.

Port publishing is performed when the container is started using `-p`.

---

## CMD

```dockerfile
CMD ["python", "app.py"]
```

Defines the default command that runs when a container starts from the image.

In this project:

```text
Container Starts
      ↓
CMD
      ↓
python app.py
      ↓
Flask Starts
```

### RUN vs CMD

A key distinction is:

```text
RUN → executes while building the IMAGE

CMD → executes when starting the CONTAINER
```

---

# 5. Building the Docker Image

The image was built using:

```bash
docker build -t hello-flask-practice .
```

### Command Breakdown

`docker build`

Builds a Docker image.

`-t`

Allows the image to be given a tag/name.

`hello-flask-practice`

The name assigned to the image.

`.`

Uses the current directory as the Docker build context.

Docker then processes the Dockerfile:

```text
FROM
  ↓
WORKDIR
  ↓
COPY
  ↓
RUN
  ↓
EXPOSE
  ↓
CMD
  ↓
Docker Image
```

---

# 6. Viewing Docker Images

To view locally available images:

```bash
docker images
```

After building, the image appeared as:

```text
hello-flask-practice:latest
```

At this stage, the application was packaged into an image but was not yet running.

```text
Docker Image ≠ Running Application
```

A container still needed to be created from the image.

---

# 7. Running the Docker Container

The container was started using:

```bash
docker run -d -p 5000:5000 --name flask-practice hello-flask-practice
```

### Command Breakdown

`docker run`

Creates and starts a container from an image.

`-d`

Runs the container in detached mode so it runs in the background.

`-p 5000:5000`

Publishes/maps the host port to the container port.

The syntax is:

```text
-p HOST_PORT:CONTAINER_PORT
```

Therefore:

```text
-p 5000:5000

Host :5000
    ↓
Container :5000
```

`--name flask-practice`

Assigns the container a human-readable name.

`hello-flask-practice`

Specifies the Docker image used to create the container.

---

# 8. Docker Port Mapping

The application can be accessed through:

```text
http://localhost:5000
```

The request flows through:

```text
Browser
   ↓
localhost:5000
   ↓
Host Port 5000
   ↓
Docker Port Mapping
   ↓
Container Port 5000
   ↓
Flask Application
```

The host and container ports do not have to be the same.

For example:

```bash
docker run -p 8080:5000 hello-flask-practice
```

would create:

```text
Browser
   ↓
localhost:8080
   ↓
Host :8080
   ↓
Container :5000
   ↓
Flask
```

The important syntax to remember is:

```text
-p HOST_PORT:CONTAINER_PORT
```

---

# 9. Viewing Running Containers

Running containers can be viewed using:

```bash
docker ps
```

For this project, the output showed information such as:

```text
IMAGE        → hello-flask-practice
COMMAND      → python app.py
STATUS       → Up
PORTS        → 5000->5000
NAME         → flask-practice
```

This confirmed that the application was running inside the Docker container.

---

# 10. Docker Image vs Docker Container

## Docker Image

An image is a packaged, reusable template.

The `hello-flask-practice` image contains roughly:

```text
Docker Image
│
├── Python 3.8
├── Flask
└── /app
    └── app.py
```

## Docker Container

A container is a running instance of an image.

For this project:

```text
IMAGE
hello-flask-practice
        ↓
    docker run
        ↓
CONTAINER
flask-practice
```

One Docker image can be used to create multiple containers.

---

# 11. Running Locally vs Running With Docker

One of the most important lessons from this project was understanding why the webpage looked identical before and after using Docker.

The application itself did not change.

What changed was the environment running it.

## Before Docker

```text
Host Machine
    ↓
Python
    ↓
.venv
    ↓
Flask
    ↓
app.py
    ↓
localhost:5000
```

The host machine was directly responsible for providing Python and Flask.

## With Docker

```text
Host Machine
    ↓
Docker
    ↓
Container
│
├── Python 3.8
├── Flask
└── app.py
     ↓
Flask :5000
     ↓
Docker Port Mapping
     ↓
localhost:5000
```

The container now provides the application's runtime and dependencies.

The local Python virtual environment is no longer required to run the containerised application.

---

# 12. Why Docker Is Useful

Without Docker, another engineer may need instructions such as:

```text
Install Python
Install the correct Python version
Create a virtual environment
Install Flask
Install other dependencies
Run the application
```

Different machines may have different versions or configurations, which can lead to the classic:

> "It works on my machine."

Docker allows the application and its required environment to be packaged together.

Conceptually:

```text
Application
+
Runtime
+
Dependencies
+
Configuration
+
Startup Command
=
Docker Image
```

Someone with Docker can then run the image without manually reproducing the same application environment.

---

# Useful Commands

```bash
# Create a Python virtual environment
python3 -m venv .venv

# Activate the virtual environment
source .venv/bin/activate

# Exit the virtual environment
deactivate

# Install Flask locally
pip install flask

# Run Flask locally
python app.py

# Build the Docker image
docker build -t hello-flask-practice .

# View Docker images
docker images

# Create and start the container
docker run -d -p 5000:5000 --name flask-practice hello-flask-practice

# View running containers
docker ps

# View all containers, including stopped containers
docker ps -a

# Stop the container
docker stop flask-practice

# Start the container again
docker start flask-practice

# Remove a stopped container
docker rm flask-practice
```

---

# Key Takeaways

The main Docker workflow is:

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
    ↓
Application Running
```

Important distinctions:

```text
Dockerfile = instructions for building an image

Image = packaged reusable template

Container = running instance of an image
```

```text
RUN = executes during image build

CMD = executes when the container starts
```

```text
EXPOSE = documents the intended container port

-p = actually publishes/maps host and container ports
```

```text
-p HOST_PORT:CONTAINER_PORT
```

The biggest lesson from this exercise:

> Docker does not necessarily change what the application does. It changes how the application and its dependencies are packaged, distributed and run.