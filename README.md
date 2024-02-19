# Dockerizing Django App with Nginx and Celery

This documentation outlines the process of dockerizing a Django web application along with Nginx as a reverse proxy server and Celery for handling asynchronous tasks. 
## Components
- **Gauss**: The primary web application built using the Django framework.
- **Nginx**: A high-performance web server acting as a reverse proxy for the Django application.
- **Celery**: A distributed task queue managing background and asynchronous tasks.
- **Docker**: Containerization platform for packaging the application and its dependencies into isolated containers.
  
## Files needed

You will need these files in order to run Django with Docker:

- `.dockerignore`: Specifies files and directories to be ignored during Docker builds.
- `Docker/Gauss/Dockerfile`: Dockerfile for the Django application.
- `Docker/Nginx/Dockerfile`: Dockerfile for Nginx.
- `docker-compose.yml`: Configuration file for Docker Compose.
- `scripts/beat_start`: Script to start Celery beat.
- `scripts/django_start`: Script to start the Django development server.
- `scripts/nginx.conf`: Configuration file for Nginx.
- `scripts/worker_start`: Script to start Celery worker.

### `.dockerignore`

The `.dockerignore` file specifies which files and directories should be ignored by Docker during the image build process. This helps optimize the build by excluding unnecessary files and directories
- `**/__pycache__`, `**/.venv`: Ignores Python cache directories and virtual environment directories.
- `**/.git`, `**/.gitignore`: Ignores Git version control files and directories.
- `**/node_modules`, `**/npm-debug.log`: Ignores Node.js dependencies and debug logs.
- `**/docker-compose*`, `**/compose*`, `**/Dockerfile*`: Ignores Docker-related files and directories.
- `LICENSE`, `README.md`: Ignores licensing and documentation files.

By specifying these patterns, unnecessary files and directories are excluded from the Docker context, improving build performance and keeping the resulting image size smaller.
### `Docker/Gauss/Dockerfile`

The `Docker/Gauss/Dockerfile` defines the instructions for building a Docker image for your Django application. Below is the explanation of each section:

1. **Base Image**:
    ```Dockerfile
    FROM python:3.7.17-slim-bullseye
    ```
    - Specifies the base image as `python:3.7.17-slim-bullseye`, a slim version of Python 3.7 on Debian Bullseye.

2. **Creating Directory Structure**:
    ```Dockerfile
    RUN mkdir -p /var/www/python
    ```
    - Creates a directory `/var/www/python` to store the Django project.

3. **Updating and Installing Dependencies**:
    ```Dockerfile
    RUN apt update \
        && apt install -y git \
        && apt install gcc libpq-dev -y \
        && apt-get install -y locales locales-all
    ```
    - Updates the package index and installs `git`, `gcc`, `libpq-dev`, and locale packages.

4. **Copying Source Code**:
    ```Dockerfile
    COPY / /var/www/python/gauss3
    ```
    - Copies the local source code into the Docker image at `/var/www/python/gauss3`.

5. **Creating Log Directory and File**:
    ```Dockerfile
    RUN mkdir -p /var/www/python/gauss3/log \
        && touch /var/www/python/gauss3/log/logGaussDomotica.log
    ```
    - Creates a log directory and an empty log file.

6. **Setting Working Directory**:
    ```Dockerfile
    WORKDIR /var/www/python/gauss3
    ```
    - Sets the working directory to `/var/www/python/gauss3`.

7. **Exposing Ports**:
    ```Dockerfile
    EXPOSE 8000
    ```
    - Exposes port 8000 for the Django application.

8. **Environment Variables**:
    ```Dockerfile
    ENV PYTHONDONTWRITEBYTECODE=1 \
        PYTHONUNBUFFERED=1
    ```
    - Prevents Python from writing bytecode and sets unbuffered output for easier logging.

9. **Installing Python Dependencies**:
    ```Dockerfile
    COPY requirements.txt /var/www/python/gauss3/requirements.txt
    RUN python -m pip install -r requirements.txt
    ```
    - Copies `requirements.txt` and installs Python dependencies.

10. **Copying Configuration Files**:
    ```Dockerfile
    COPY gauss/settings.py /var/www/python/gauss3/gauss/settings.py
    ```
    - Copies `settings.py` to the Django project directory.

11. **Copying Scripts and Setting Permissions**:
    ```Dockerfile
    COPY ./scripts/worker_start /start-celeryworker
    COPY ./scripts/beat_start /start-celerybeat
    COPY ./scripts/django_start /start-django
    RUN sed -i 's/\r$//g' /start-celeryworker \
        && chmod +x /start-celeryworker \
        && sed -i 's/\r$//g' /start-celerybeat \
        && chmod +x /start-celerybeat \
        && sed -i 's/\r$//g' /start-django \
        && chmod +x /start-django
    ```
    - Copies startup scripts and adjusts permissions for execution.

12. **Static Files Collection**:
    ```Dockerfile
    RUN mkdir /var/www/python/gauss3/sitestatic \
        && python /var/www/python/gauss3/manage.py collectstatic --noinput
    ```
    - Creates a directory for static files and collects static files into it.

13. **Creating Non-root User**:
    ```Dockerfile
    RUN adduser -u 5678 --disabled-password --gecos "" appuser \
        && chown -R appuser /var/www/python/gauss3/
    USER appuser
    ```
    - Creates a non-root user `appuser` with UID 5678 and sets ownership of the project directory to `appuser`.
      
  ### `Docker/Nginx/Dockerfile`

The `Docker/Nginx/Dockerfile` defines the instructions for building a Docker image for Nginx, a web server, and reverse proxy. Below is the explanation of each section:

1. **Base Image**:
    ```Dockerfile
    FROM nginx:1.25
    ```
    - Specifies the base image as `nginx:1.25`, which is the official Nginx image.

2. **Removing Default Configuration**:
    ```Dockerfile
    RUN rm /etc/nginx/conf.d/default.conf
    ```
    - Removes the default Nginx configuration file `default.conf` from the container.

3. **Copying Custom Configuration**:
    ```Dockerfile
    COPY scripts/nginx.conf /etc/nginx/conf.d
    ```
    - Copies a custom Nginx configuration file `nginx.conf` from the host to the container's `/etc/nginx/conf.d` directory.
### `docker-compose.yml`

The `docker-compose.yml` file defines the services and their configurations for running your Dockerized Django application along with PostgreSQL, RabbitMQ, Nginx, and Celery workers. Here's the breakdown:

1. **Version**:
    ```yaml
    version: '3.4'
    ```
    - Specifies the version of Docker Compose file format being used.

2. **Services**:
    - **postgres-gauss**:
        - Runs PostgreSQL for the Django application.
        - Binds the container's `/var/lib/postgresql/data` to a volume named `postgres-gauss`.
        - Exposes port 5432 for PostgreSQL connections.
        - Loads environment variables from `.env`.
    - **rabbitmq**:
        - Runs RabbitMQ with management plugin for message queuing.
        - Exposes ports 5672 and 15672 for AMQP and management interface.
        - Binds the container's RabbitMQ data directory to a volume.
        - Loads environment variables from `.env`.
    - **gauss3**:
        - Runs the Django application using the custom Dockerfile in `Docker/Gauss`.
        - Depends on `postgres-gauss` and `rabbitmq`.
        - Maps port 8000 for Django web server.
        - Mounts volumes for static files and media files.
    - **celery_worker**:
        - Runs Celery worker for asynchronous tasks using the same Dockerfile as `gauss3`.
        - Depends on `postgres-gauss` and `rabbitmq`.
    - **celery_beat**:
        - Runs Celery beat for periodic tasks using the same Dockerfile as `gauss3`.
        - Depends on `postgres-gauss` and `rabbitmq`.
    - **nginx**:
        - Runs Nginx as a reverse proxy for the Django application.
        - Builds the Nginx image using the custom Dockerfile in `Docker/Nginx`.
        - Maps port 1337 to port 80.
        - Depends on `gauss3`.
        - Mounts volumes for static files and media files.

3. **Volumes**:
    - Defines volumes for PostgreSQL, RabbitMQ data, static files, and media files.

4. **Networks**:
    - Defines a custom network named `gauss_network` for inter-service communication.

This `docker-compose.yml` orchestrates the setup and linkage between various services required for your Django application to run in Docker containers.

### `scripts/beat_start`

The `scripts/beat_start` file is a Bash script used to start Celery beat, which is responsible for scheduling periodic tasks in your Django application. Below is the explanation of its content:

```bash
#!/bin/bash
```
- Specifies that this file should be interpreted by the Bash shell.

```bash
set -o errexit
set -o pipefail
set -o nounset
```
- Sets shell options:
  - `errexit`: Exits immediately if any command exits with a non-zero status.
  - `pipefail`: Causes a pipeline to return a non-zero status if any command in the pipeline fails.
  - `nounset`: Treats unset variables as an error when expanding them.

```bash
exec celery -A gauss.celery_app beat -l INFO
```
- Executes the Celery beat command:
  - `-A gauss.celery_app`: Specifies the Celery app to use, where `gauss` is the name of your Django project and `celery_app` is the Celery app defined within it.
  - `beat`: Indicates that Celery beat should be started.
  - `-l INFO`: Sets the log level to INFO, which determines the verbosity of log messages.

This script ensures that Celery beat is started with appropriate error handling and logging configurations.

### `scripts/django_start`

The `scripts/django_start` file is a Bash script used to start the Django application server. Below is the explanation of its content:

```bash
#!/bin/bash
```
- Specifies that this file should be interpreted by the Bash shell.

```bash
set -o errexit
set -o pipefail
set -o nounset
```
- Sets shell options:
  - `errexit`: Exits immediately if any command exits with a non-zero status.
  - `pipefail`: Causes a pipeline to return a non-zero status if any command in the pipeline fails.
  - `nounset`: Treats unset variables as an error when expanding them.

```bash
exec /usr/local/bin/gunicorn gauss.wsgi -w 3 --threads 3 --bind 0.0.0.0:8000 --chdir=/var/www/python/gauss3
```
- Executes the Gunicorn command to run the Django application:
  - `/usr/local/bin/gunicorn`: Specifies the path to the Gunicorn executable.
  - `gauss.wsgi`: Indicates the WSGI application entry point for Gunicorn.
  - `-w 3`: Sets the number of worker processes to 3.
  - `--threads 3`: Sets the number of worker threads per process to 3.
  - `--bind 0.0.0.0:8000`: Binds the server to listen on all available network interfaces (`0.0.0.0`) on port `8000`.
  - `--chdir=/var/www/python/gauss3`: Changes the working directory to `/var/www/python/gauss3` before loading the application.

This script ensures that the Django application is started using Gunicorn with appropriate configurations.
### `scripts/nginx.conf`

The `scripts/nginx.conf` file contains the Nginx configuration for serving your Django application. Below is the explanation of its content:

```nginx
upstream django {
    server gauss3:8000 max_fails=0; 
}
```
- Defines an upstream server block named `django`, specifying the hostname `gauss3` and port `8000` where the Django application (served by Gunicorn) is running.

```nginx
server {
    listen 80;
    charset utf-8;
    client_max_body_size 75M;
    server_name localhost;
```
- Defines an Nginx server block:
  - `listen 80`: Specifies that Nginx should listen on port 80.
  - `charset utf-8`: Sets the character encoding to UTF-8.
  - `client_max_body_size 75M`: Sets the maximum allowed size of the client request body to 75 megabytes.
  - `server_name localhost`: Sets the server name to `localhost`.

```nginx
    location /static/ {
        alias /var/www/python/gauss3/sitestatic/;
    }
```
- Defines a location block for serving static files:
  - Maps requests to the `/static/` URL path to the directory `/var/www/python/gauss3/sitestatic/` where static files are collected.

```nginx
    location /media/ {
        alias /var/www/python/gauss3/media/;
    }
```
- Defines a location block for serving media files:
  - Maps requests to the `/media/` URL path to the directory `/var/www/python/gauss3/media/` where media files are stored.

```nginx
    location / {
        proxy_pass http://django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        autoindex on;
    }
```
- Defines a location block for handling all other requests:
  - Forwards requests to the upstream server defined in the `django` upstream block (`http://django`).
  - Sets HTTP headers `X-Forwarded-For` and `Host`.
  - Enables directory listing if indexes are not found.

This configuration ensures that Nginx serves static and media files directly and forwards other requests to the Django application server.
### `scripts/worker_start`

The `scripts/worker_start` file is a Bash script used to start a Celery worker process. Below is the explanation of its content:

```bash
#!/bin/bash
```
- Specifies that this file should be interpreted by the Bash shell.

```bash
set -o errexit
set -o pipefail
set -o nounset
```
- Sets shell options:
  - `errexit`: Exits immediately if any command exits with a non-zero status.
  - `pipefail`: Causes a pipeline to return a non-zero status if any command in the pipeline fails.
  - `nounset`: Treats unset variables as an error when expanding them.

```bash
exec celery -A gauss.celery worker -l INFO
```
- Executes the Celery worker command:
  - `-A gauss.celery`: Specifies the Celery app to use, where `gauss` is the name of your Django project and `celery` is the Celery app defined within it.
  - `worker`: Indicates that Celery should start a worker process.
  - `-l INFO`: Sets the log level to INFO, which determines the verbosity of log messages.

This script ensures that a Celery worker is started with appropriate error handling and logging configurations.
