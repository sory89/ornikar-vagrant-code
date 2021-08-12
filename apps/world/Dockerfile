FROM php:7.2-cli
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git
WORKDIR /app
RUN curl -sS https://getcomposer.org/installer | \
    php -- --install-dir=/usr/bin/ --filename=composer
COPY composer.json /app
RUN composer install --no-scripts --no-autoloader
COPY . /app
CMD php public/index.php
EXPOSE 3000
