---
title: Laravel Development Environment
layout: post
category: tutorial-tips
tags: [skill, develop]
feature: /assets/img/laravel-environment.png
---

## Mục đích của tài liệu
- Sử dụng Docker containers để thiết lập môi trường lâp trình cho ứng dụng Laravel 1 cách nhanh chóng.

<!--more-->
## Chuẩn bị kiến thức
> Để tập trung vào mục đích chính của tài liệu này nên mình sẽ không đi sâu vào các khái niệm hay kiến thức liên quan, nếu các bạn thiếu kiến thức nào hãy chủ động tìm hiểu bằng các nguồn có sẵn trên mạng. Ở đây mình chỉ liệt kê ra các kiến thức cần phải có để có thể bắt đầu công việc 1 cách trôi chảy nhất.
- Biết [docker](https://www.docker.com/) và [docker-compose](https://docs.docker.com/compose/) là gì
- Các khái niệm chính của Docker: Image, Container
- Cài `docker` và `docker-compose` trong máy
## Tiến hành cài đặt
### Sử dụng image có sẵn
> Đây là cách nhanh nhất và cũng đơn giản nhất để setup được các môi trường cơ bản cho 1 ứng dụng web.
Sử dụng các image có sẵn do FramgiaDockerTeam đã build. File `docker-compose.yml` sẽ như sau:
(Change the `project` with your own project name)

```

version: '3'

services:
    application:
        container_name: project_application
        image: debian
        volumes:
            - ./:/var/www/laravel
    workspace:
        container_name: project_workspace
        restart: always
        image: framgia/laravel-workspace
        links:
            - application
        tty: true
    php-fpm:
        container_name: project_php-fpm
        restart: always
        image: framgia/laravel-php-fpm
        links:
            - application
        expose:
            - "9000"
        links:
            - workspace
    nginx:
        container_name: project_nginx
        restart: always
        image: framgia/laravel-nginx
        links:
            - data
            - logs
            - application
        ports:
            - "8000:80"
        links:
            - php-fpm
    data:
        container_name: project_data
        image: debian
        volumes:
            - .docker/mysql:/var/lib/mysql
            - .docker/data:/data
    data_test:
        container_name: project_data_test
        image: debian
        volumes:
            - .docker/mysql_test:/var/lib/mysql
            - .docker/data_test:/data
    logs:
        container_name: project_logs
        image: debian
        volumes:
            - .docker/logs/nginx:/var/log/nginx
            - .docker/logs/mongodb:/var/log/mongodb
    mysql:
        container_name: project_mysql
        restart: always
        image: mysql
        links:
            - data
            - logs
        expose:
            - "3306"
        environment:
            MYSQL_DATABASE: homestead
            MYSQL_USER: homestead
            MYSQL_PASSWORD: secret
            MYSQL_ROOT_PASSWORD: root
    mysql_test:
        container_name: project_mysql_test
        restart: always
        image: mysql:5.7
        links:
            - data_test
        expose:
            - "3306"
        environment:
            MYSQL_DATABASE: homestead_test
            MYSQL_USER: homestead_test
            MYSQL_PASSWORD: secret
            MYSQL_ROOT_PASSWORD: root
    mongo:
        container_name: project_mongo
        restart: always
        image: mongo
        expose:
            - "27017"
        links:
            - data
            - logs
    redis:
        container_name: project_redis
        restart: always
        image: redis
        expose:
            - "6379"
        links:
            - data

```

# Các container sau sẽ được chạy mặc định sau khi chạy lệnh `docker-compose up`:
> Các bạn cũng có thể tùy biến (thêm hoặc xóa) các `services` cho phù hợp với yêu cầu môi trường phát triển của từng dự án.
- nginx
- application (for storing project source code)
- php-fpm
- workspace (for working around with the all project)
- mysql
- mysql_test (for running integration test)
- mongodb
- redis
- data (for storing mysql, mongo, redis data)
- data_test (for storing mysql, mongo, redis data while running test)
- logs (for storing some system logs)
#### Cách sử dụng
- Trong thư mục `root` của dự án, tạo file `docker-compose.yml`, sau đó copy nội dung file ở bên trên phệt vào.
- Nhớ đổi tên `project` keyword cho đúng với tên dự án thực tế nhé.
- Các dữ liệu (mysql, redis, mongo data,...) sẽ được lưu trong thư mục `.docker`. Vì thế chúng ta cần tạo thư mục `.docker` bên trong thư mục `root` của dự án và nhớ `chmod 777` và thêm vào file `.gitignore` cho thư mục này nhé.
- Chạy lênh `docker-compose up -d` và tận hưởng thành quả
- Chạy lệnh `docker exec -it project_workspace bash` để `exec` vào workspace container. Trong này bạn có thể chạy toàn bộ các lệnh `php artisan` hoặc tương tự
#### Ưu điểm
- Nhanh chóng, tiện dụng, đơn giản để triển khai
#### Nhược điểm
- Chỉ sử dụng được cho các dự án Laravel.
- Khó khăn hơn trong việc customize hoặc optimize
### Sử dụng Dockerfile
> Đối với Solution này chúng ta có thể khắc phục nhược điểm của việc sử dụng `hàng sẵn có của FramgiaDockerTeam` đồng thời có thể optimize/customize cho tất cả các môi trường mà project yêu cầu.
#### Preparing

#### Making docker-compose.yml file
Để có được môi trường cơ bản cho việc khởi chạy ứng dụng PHP, chúng ta sẽ tách thành 3 services chính như sau:
- `app` container dùng cho việc chạy và thực thi mã code PHP
- `web` container chạy NGINX,
- `database` container này sẽ chạy môi trường database (MySQL, MongoDB,...).
1. App Service
```
app:
    build:
      context: ./
      dockerfile: ./docker/app.Dockerfile
    volumes:
      - ./:/var/www/html
```
Notes:
- **dockerfile**: Chúng ta không dùng image có sẵn trên DockerHub mà sử dụng file `app.Dockerfile` để build `app` service
- *./:/var/www/html*: Mount thư mục hiện tại tới thư mục `/var/www/html` trong container
Bây giờ chúng ta tạo file app.Dockerfile bên trong thư mục `docker` với nội dung như sau:
```
FROM php:7.2-fpm-alpine

RUN apk --update add wget \
  curl \
  git \
  grep \
  build-base \
  libmemcached-dev \
  libmcrypt-dev \
  libxml2-dev \
  imagemagick-dev \
  pcre-dev \
  libtool \
  make \
  autoconf \
  g++ \
  cyrus-sasl-dev \
  libgsasl-dev

RUN docker-php-ext-install mysqli mbstring pdo pdo_mysql tokenizer xml
RUN pecl channel-update pecl.php.net \
    && pecl install memcached \
    && pecl install imagick \
    && pecl install mcrypt-1.0.1 \
    && docker-php-ext-enable memcached \
    && docker-php-ext-enable imagick \
    && docker-php-ext-enable mcrypt

RUN rm /var/cache/apk/* && \
    mkdir -p /var/www

WORKDIR /var/www/html
```
2. Web Service
Ta sẽ cần 1 webservice để xử lý các request đến và đưa đến cho laravel xử lý. Ngoài nginx ta có thể chọn apache.
```
web:
    build:
      context: ./
      dockerfile: ./docker/web.Dockerfile
    links:
      - app
    ports:
      - 8000:80
```
Notes:
- **dockerfile**: Tương tự như `app` service, nhưng khác ở chỗ chúng ta sử dụng file `web.Dockerfile` thay vì `app.Dockerfile`
- `links`: tới service `app` để liên kết 2 `container` lại với nhau
- `ports`: Ánh xạ cổng 8080 trên máy host vào cổng 80 trên container.
Bây giờ chúng ta tạo file web.Dockerfile bên trong thư mục `docker` với nội dung như sau:
```
FROM nginx:alpine

RUN addgroup -g 1000 -S www-data \
    && adduser -u 1000 -D -S -G www-data www-data

COPY ./docker/nginx.conf /etc/nginx/nginx.conf

WORKDIR /var/www/html
```

Notes:
- `nginx:alpine` là base image được sử dụng
- Thêm user và group mới vì `nginx: alpine` không có default user.
- Thêm `nginx.conf` để cấu hình server
> File cấu hình `nginx.conf`:
```
events {
  worker_connections  2048;
}

http {
    server {
        listen 80;
        index index.php index.html;
        root /var/www/html/public;

        location / {
            try_files $uri /index.php?$args;
        }

        location ~ \.css {
            add_header Content-Type text/css;
        }

        location ~ \.js {
            add_header Content-Type application/x-javascript;
        }

        location ~ ^/(assets/|css/|js/|index.html) {
            root /var/www/html/public;
            index index.html;
            access_log off;
        }

        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass app:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
        }
    }
}
```
Notes:
- Ta đặt `root` đúng với đường dẫn tới folder `public` trong `container` vì chúng ta đang ví dụ với ứng dụng Laravel.
3. Database Service
```
mysql:
    image: mysql:5.7
    expose:
      - 3306
    environment:
      - "MYSQL_DATABASE=homestead"
      - "MYSQL_USER=homestead"
      - "MYSQL_PASSWORD=secret"
      - "MYSQL_ROOT_PASSWORD=secret"
    volumes:
      - ./docker/data/mysql:/var/lib/mysql
```
Notes:
- `image: mysql:5.7` Service `mysql` này chỉ cần pull `image mysql` về là đủ. Các đặc tả cần thiết khác đều có sẵn trong image này nên ta sẽ không cần viết riêng.
- `enviroment`: Các biến môi trường. Với định nghĩa như ở trên, mysql sẽ tạo cho chúng ta 1 database và 1 user như vậy. (Nếu không đặc tả thì sẽ chỉ có 1 user root và database mặc định của mysql)

> Cuối cùng ta có được file docker-compose.yml như sau:

```

version: '3'

services:
  app:
    build:
      context: ./
      dockerfile: ./docker/app.Dockerfile
    volumes:
      - ./:/var/www/html
  web:
    build:
      context: ./
      dockerfile: ./docker/web.Dockerfile
    links:
      - app
    ports:
      - 8000:80
  mysql:
    image: mysql:5.6
    expose:
      - 3306
    environment:
      - "MYSQL_DATABASE=homestead"
      - "MYSQL_USER=homestead"
      - "MYSQL_PASSWORD=secret"
      - "MYSQL_ROOT_PASSWORD=secret"
    volumes:
      - ./docker/data/mysql:/var/lib/mysql
```
#### Usage
- Ta sẽ chạy các services trên bằng command sau: `docker-compose up -d`
> Lần đầu chạy, command sẽ cần khoảng vài phút để pull về các image cần thiết. Nhưng lần thứ 2 trở đi nếu các chỉ định images không thay đổi thì thời gian chạy sẽ nhanh hơn nhiều do không phải tải lại các image nữa.
- Để chạy các câu lệnh bên trong container `docker-compose exec <service-name> <câu lệnh>`
> Ví dụ: `docker-compose exec app php artisan migrate --seed`

