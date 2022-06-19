# DevOps Coding Challenge

This challenge is about creating a devops solution for a [Laravel](https://laravel.com/)  application using docker and seting up a full solution from continuous testing to deployment to monitoring, with this following features:

## **Features 🚀**

Tech stack for the application is LEMP.

- Need to build and setup a deployment environnement using docker.
- Need to have Supervisor for Laravel queue.
- Need to have a way to monitor application health and security alerts.
- Need to document the entire setup process ([Notion](https://notion.so/) Recommended).
- Option to rollback deployments.
- Database and application need to be separated.

and the following criteria:

## **Evaluation criteria 🚨**

- Scalability, support for spike request.
- Reliability.
- Zero-down time deployments.
- High Security.
- Monitoring

My solution is this repo 😎 and here is my setup :

![Untitled](DevOps%20Coding%20Challenge%20781a5cd1c3ec428c876e03dbe8f762cb/Untitled.png)

## **Project Setup**

1. Create repository on github
2. Initiat the repository locally and create laravel project with composer:

   ```php
   **composer create-project laravel/laravel .**
   ```

3. Setup php-fpm, nginx, supervisord configuration:

   ```
   worker_processes  auto;

   error_log  /var/log/nginx/error.log notice;
   pid        /tmp/nginx.pid;

   events {
       worker_connections  1024;
   }

   http {
       include       /etc/nginx/mime.types;
       default_type  application/octet-stream;

       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for"';

       access_log  /var/log/nginx/access.log  main;

       sendfile        on;
       #tcp_nopush     on;

       keepalive_timeout  65;

       gzip  on;

       include /etc/nginx/conf.d/*.conf;
   }
   ```

   ```
   server {
       listen 8080;

       root /var/www/public;

       add_header X-Frame-Options "SAMEORIGIN";
       add_header X-Content-Type-Options "nosniff";

       index index.php;

       charset utf-8;

       location / {
           try_files $uri $uri/ /index.php?$query_string;
       }

       location = /favicon.ico { access_log off; log_not_found off; }
       location = /robots.txt  { access_log off; log_not_found off; }

       error_page 404 /index.php;

       location ~ \.php$ {
           try_files $uri =404;
           fastcgi_pass localhost:9000;
           fastcgi_index index.php;
           include fastcgi_params;
           fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
           fastcgi_param PATH_INFO $fastcgi_path_info;
       }

       location ~ /\.(?!well-known).* {
           deny all;
       }
   }
   ```

   ```
   [supervisord]
   nodaemon=true
   logfile=/dev/stdout
   logfile_maxbytes=0
   pidfile=/tmp/supervisord.pid
   loglevel = INFO

   [program:php-fpm]
   command = /usr/local/sbin/php-fpm
   autostart=true
   autorestart=true
   stdout_logfile=/dev/stdout
   stdout_logfile_maxbytes=0
   stderr_logfile=/dev/stderr
   stderr_logfile_maxbytes=0

   [program:nginx]
   command=/usr/sbin/nginx -g "daemon off;"
   autostart=true
   autorestart=true
   stdout_events_enabled=true
   stderr_events_enabled=true
   stdout_logfile=/dev/stdout
   stdout_logfile_maxbytes=0
   stderr_logfile=/dev/stderr
   stderr_logfile_maxbytes=0
   ```

4. Create Dockerfile

   `Dockerfile` defines all the commands that will create the container image

   ```docker
   FROM composer:2 AS build

   # copying the source directory and install the dependencies with composer
   COPY . /app

   # run composer install to install the dependencies
   RUN composer install \
     --no-interaction \
     --no-progress

   FROM php:8-fpm AS prod

   # Set working directory
   WORKDIR /var/www

   # Install dependencies
   RUN apt-get update && apt-get install -y \
     build-essential \
     libpng-dev \
     libjpeg62-turbo-dev \
     libfreetype6-dev \
     locales \
     zip \
     jpegoptim optipng pngquant gifsicle \
     vim \
     unzip \
     zip \
     git \
     curl \
     supervisor \
     nginx

   # Clear cache
   RUN apt-get clean && rm -rf /var/lib/apt/lists/*

   # Install extensions
   RUN docker-php-ext-install pdo_mysql

   # Add user for laravel application
   RUN groupadd -g 1000 www
   RUN useradd -u 1000 -ms /bin/bash -g www www

   # Copy existing application directory contents
   COPY --chown=www:www --from=build /app /var/www

   # Allow access to logs for www
   RUN chmod o+rwx /var/log/nginx
   RUN chmod o+rw /var/log/nginx/*
   RUN chmod o+rwx /var/lib/nginx
   # Copy nginx config
   COPY ./docker/nginx.conf /etc/nginx/nginx.conf
   COPY ./docker/default.conf /etc/nginx/conf.d/default.conf

   # Copy supervisor config
   COPY ./docker/supervisord.conf /etc/supervisord.conf

   # Change current user to www
   USER www

   # Expose port 8080 and start supervisord
   EXPOSE 8080
   CMD supervisord -n -c /etc/supervisord.conf
   ```

   Beginning with the building where i setup the environment:

   ```docker
   FROM composer:2 AS build

   # copying the source directory and install the dependencies with composer
   COPY . /app

   # run composer install to install the dependencies
   RUN composer install \
     --no-interaction \
     --no-progress
   ```

   After that I’m using [php-fpm7](https://hub.docker.com/_/php/) docker image as my base image for prod stage:

   ```docker
   FROM php:7.0-fpm
   ```

   Installing all the dependencies needed (nginx,supervisor…..):

   ```docker
   # Install dependencies
   RUN apt-get update && apt-get install -y \
     build-essential \
     libpng-dev \
     libjpeg62-turbo-dev \
     libfreetype6-dev \
     locales \
     zip \
     jpegoptim optipng pngquant gifsicle \
     vim \
     unzip \
     zip \
     git \
     curl \
     supervisor \
     nginx
   ```

   Then we clean the cache and the informations for each package resource to minimize the image size:

   ```docker
   # Clear cache
   RUN apt-get clean && rm -rf /var/lib/apt/lists/*
   ```

   To connect laravel with mysql it must install `pdo_mysql` extension:

   ```docker
   # Install extensions
   RUN docker-php-ext-install pdo_mysql
   ```

   To secure the application i created a new user and i gave the ownership to this user, besides copying the app content to the /var/www and giving the access to the logs:

   ```docker
   # Add user for laravel application
   RUN groupadd -g 1000 www
   RUN useradd -u 1000 -ms /bin/bash -g www www

   # Copy existing application directory contents
   COPY --chown=www:www --from=build /app /var/www

   # Allow access to logs for www
   RUN chmod o+rwx /var/log/nginx
   RUN chmod o+rw /var/log/nginx/*
   RUN chmod o+rwx /var/lib/nginx

   ```

   Updating the configurations and changing the user to www to enforce security:

   ```docker
   # Copy nginx config
   COPY ./docker/nginx.conf /etc/nginx/nginx.conf
   COPY ./docker/default.conf /etc/nginx/conf.d/default.conf

   # Copy supervisor config
   COPY ./docker/supervisord.conf /etc/supervisord.conf

   # Change current user to www
   USER www
   ```

   When the container instance starts it executes the instruction “**`supervisord -n -c /etc/supervisord.conf`**”, the process of **supervisord** starts and forks two process children (**NGINX**,**PHP-FPM**) to supervise.

5. Create **Kubernetes** cluster on GCP(Google Cloud Provider) ([Link](https://zero-to-jupyterhub.readthedocs.io/en/latest/kubernetes/google/step-zero-gcp.html))
6. Setup gcloud,kubectl,gke-gcloud-auth plugin localy.
7. Deploying mysql in the cluster using helm:

   ```
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm install my-release bitnami/mysql
   ```

8. Create CI/CD.

```yaml
name: Build and deploy to GCP
on:
  push:
    branches:
      - main
env:
  PROJECT_ID: zeta-essence-353611
  GKE_CLUSTER: mycluster
  GKE_ZONE: us-central1

jobs:
  app:
    name: Deploy the app
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      # Setup gcloud CLI
      - uses: google-github-actions/setup-gcloud@94337306dda8180d967a56932ceb4ddcf01edae7
        with:
          service_account_key: ${{ secrets.GKE_SA_KEY }}
          project_id: ${{ env.PROJECT_ID }}

      # Configure Docker to use the gcloud command-line tool as a credential
      # helper for authentication
      - run: |-
          gcloud --quiet auth configure-docker
      # Get the GKE credentials
      - uses: google-github-actions/get-gke-credentials@fb08709ba27618c31c09e014e1d8364b02e5042e
        with:
          cluster_name: ${{ env.GKE_CLUSTER }}
          location: ${{ env.GKE_ZONE }}
          credentials: ${{ secrets.GKE_SA_KEY }}

      # Build the Docker image
      - name: Build
        run: |-
          docker build \
            --tag "gcr.io/$PROJECT_ID/simplelaravel:$GITHUB_SHA" -f ./docker/Dockerfile .

      # Push the Docker image to Google Container Registry
      - name: Publish
        run: |-
          docker push "gcr.io/$PROJECT_ID/simplelaravel:$GITHUB_SHA"

        # Deploy the Docker image to the GKE cluster
      - name: Deploy
        run: |-
          envsubst < deployment.yml | kubectl apply -f -
          kubectl rollout status deployment/simplelaravel
```

Using Github Actions, we created an action to handle the deployment of the application after every push on the main branch, the action executes the following steps:

1. First we checkout the branch using the plugin:`actions/checkout@v2`.
2. We setup the gcloud CLI to be able to interact with the GCP using `google-github-actions` plugin where we provide the secrte GKE_SA key for the service account key and the PROJECT_ID for the environment.
3. Configure **Docker** to use the gcloud command-line tool as a credential helper for authentication so we could push our images to **GCP REGISTRY.**
4. Configure **Kubernetes** to use the gcloud command-line tool as a credential helper for authentication so we could use **kubectl** with **Google Kubernetes Engine**(GKE)**.**
5. Build the image with the SHA of the commit.
6. Push the image to GCP REGISTRY.
7. Deploy the Docker image to the GKE cluster and here is the deployment file:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: simplelaravel
  name: simplelaravel
spec:
  type: LoadBalancer
  ports:
    - name: web
      port: 8080
  selector:
    app: simplelaravel

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simplelaravel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simplelaravel
  template:
    metadata:
      labels:
        app: simplelaravel
    spec:
      containers:
        - name: simplelaravel
          image: gcr.io/$PROJECT_ID/simplelaravel:$GITHUB_SHA
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          ports:
            - containerPort: 8080
          env:
            - name: DB_DATABASE
              value: laravel
            - name: DB_USERNAME
              value: root
            - name: DB_PASSWORD
              value: devpass
            - name: DB_HOST
              value: my-release-with-set-mysql
            - name: APP_KEY
              value: base64:7yBl6dF5XgmnYmBOx/GhJxmwhuP3y3GUpQjxXhb5vXg=
---
```

First thing to do is to have a **LoadBalancer** service in which we can expose the application to the internet, and the second thing is to specify the image of the app with the environment variable to get connected with the Mysql service.
