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


