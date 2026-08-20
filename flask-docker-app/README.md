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

- `from flask import Flask` imports Flask.
- `app = Flask(__name__)` creates the Flask application.
- `@app.route('/')` defines what happens at the root URL.
- The return statement displays the message in the browser.
- `app.run(host='0.0.0.0', port=5000)` starts Flask and allows it to listen on all available network interfaces.

---

# 2. Running Flask Locally

Before containerising the application, I tested it directly on my machine.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install flask
python app.py
```

The application could then be accessed at `http://localhost:5000`.

At this stage Docker was not involved:

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

The `.venv` directory should not be committed to Git and can be excluded with:

```gitignore
.venv/
```

---

# 4. Creating the Dockerfile

```dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY . .
RUN pip install flask
EXPOSE 5000
CMD ["python", "app.py"]
```

A Dockerfile is a set of instructions used to build a Docker image.

## FROM

```dockerfile
FROM python:3.8-slim
```

Defines the base image. The container gets its own Python 3.8 runtime regardless of the Python version installed on the host.

## WORKDIR

```dockerfile
WORKDIR /app
```

Sets `/app` as the working directory inside the Docker image, conceptually similar to `cd /app`.

## COPY

```dockerfile
COPY . .
```

Copies files from the build context into `/app` inside the image.

```text
COPY source destination
COPY .      .
     ↑      ↑
   host    /app
```

## RUN

```dockerfile
RUN pip install flask
```

`RUN` executes while the Docker image is being built. It installs Flask inside the image so the container does not depend on Flask being installed on the host.

```text
RUN = executed while BUILDING the image
```

## EXPOSE

```dockerfile
EXPOSE 5000
```

Documents that the application inside the container is intended to listen on port 5000. `EXPOSE` does not publish the port to the host by itself.

## CMD

```dockerfile
CMD ["python", "app.py"]
```

Defines the default command that runs when a container starts.

```text
RUN → executes while building the IMAGE
CMD → executes when starting the CONTAINER
```

---

# 5. Building the Docker Image

```bash
docker build -t hello-flask-practice .
```

- `docker build` builds an image.
- `-t` assigns a tag/name.
- `hello-flask-practice` is the image name.
- `.` uses the current directory as the build context.

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

```bash
docker images
```

After building, the image appears as `hello-flask-practice:latest`.

```text
Docker Image ≠ Running Application
```

A container still needs to be created from the image.

---

# 7. Running the Docker Container

```bash
docker run -d -p 5000:5000 --name flask-practice hello-flask-practice
```

- `docker run` creates and starts a container from an image.
- `-d` runs it in detached/background mode.
- `-p 5000:5000` publishes/maps the host port to the container port.
- `--name flask-practice` gives the container a human-readable name.
- `hello-flask-practice` specifies the image.

Port syntax:

```text
-p HOST_PORT:CONTAINER_PORT
```

---

# 8. Docker Port Mapping

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

The host and container ports do not have to be the same. For example:

```bash
docker run -p 8080:5000 hello-flask-practice
```

would map host port 8080 to container port 5000.

---

# 9. Viewing Running Containers

```bash
docker ps
```

This shows the image, command, status, ports and container name.

---

# 10. Docker Image vs Docker Container

An image is a packaged, reusable template. A container is a running instance of an image.

```text
IMAGE
hello-flask-practice
        ↓
    docker run
        ↓
CONTAINER
flask-practice
```

One image can be used to create multiple containers.

---

# 11. Running Locally vs Running With Docker

Before Docker:

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

With Docker:

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

The application itself did not change. What changed was the environment running it.

---

# 12. Why Docker Is Useful

Without Docker, another engineer may need to install the correct Python version, create a virtual environment, install dependencies and reproduce the same configuration.

Docker packages the application and required environment together:

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

---

# Useful Commands

```bash
python3 -m venv .venv
source .venv/bin/activate
deactivate
pip install flask
python app.py

docker build -t hello-flask-practice .
docker images
docker run -d -p 5000:5000 --name flask-practice hello-flask-practice
docker ps
docker ps -a
docker stop flask-practice
docker start flask-practice
docker rm flask-practice
```

---

# Key Takeaways

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

The biggest lesson from this exercise:

> Docker does not necessarily change what the application does. It changes how the application and its dependencies are packaged, distributed and run.