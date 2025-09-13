# Docker + Symfony Setup

## Requirements

### Docker (Required)
- **Docker Desktop**: [Download here](https://www.docker.com/products/docker-desktop/)
- **Docker Compose** (included with Docker Desktop)

**Installation Links:**
- **Windows**: https://docs.docker.com/desktop/install/windows-install/
- **macOS**: https://docs.docker.com/desktop/install/mac-install/
- **Linux**: https://docs.docker.com/desktop/install/linux-install/

### What's Included in Docker Containers
The following will be automatically installed inside the Docker containers:

- **PHP 8.2** with extensions (PDO, MySQL, Intl, Mbstring, Zip)
- **Composer** (PHP dependency manager)
- **Node.js 20** with npm/yarn
- **Nginx** web server
- **MySQL 8** database

### Optional (for local development without Docker)
If you prefer to run things locally instead of using Docker:

- **PHP 8.2+**: https://www.php.net/downloads.php
- **Composer**: https://getcomposer.org/download/
- **Node.js 18+**: https://nodejs.org/en/download/
- **MySQL 8**: https://dev.mysql.com/downloads/mysql/

## Setup Instructions

### Step 1: Create Dockerfile
```dockerfile
FROM php:8.2-fpm

RUN apt-get update && apt-get install -y \
    git unzip libicu-dev libonig-dev libzip-dev zlib1g-dev \
    && docker-php-ext-install pdo pdo_mysql intl mbstring zip \
    && rm -rf /var/lib/apt/lists/*

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

WORKDIR /var/www/html

# Set proper permissions for www-data
RUN chown -R www-data:www-data /var/www/html
```

### Step 2: Create docker-compose.yml
```yaml
services:
  php:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: symfony_php
    volumes:
      - ./:/var/www/html
      - ./init-symfony.sh:/usr/local/bin/init-symfony.sh:ro
    working_dir: /var/www/html
    command: ["sh", "-c", "/usr/local/bin/init-symfony.sh && php-fpm"]

  nginx:
    image: nginx:alpine
    container_name: symfony_nginx
    ports:
      - "8080:80"
    volumes:
      - ./:/var/www/html:delegated
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - php

  node:
    image: node:20
    container_name: symfony_node
    working_dir: /var/www/html
    volumes:
      - ./:/var/www/html
    command: ["tail", "-f", "/dev/null"]

  db:
    image: mysql:8
    container_name: symfony_db
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: symfony
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### Step 3: Create init-symfony.sh
```bash
#!/bin/bash

# Check if Symfony project already exists
if [ ! -f "composer.json" ]; then
    echo "Creating Symfony project..."

    # Create in temp and move to avoid directory not empty error
    composer create-project symfony/skeleton:"6.4.*" /tmp/symfony --no-interaction --prefer-dist
    cp -r /tmp/symfony/. .
    rm -rf /tmp/symfony

    # Install additional packages for web development
    composer require webapp

    # Fix permissions
    chown -R www-data:www-data /var/www/html
    chmod -R 755 /var/www/html

    echo "Symfony project created successfully!"
else
    echo "Symfony project already exists!"
fi
```

### Step 4: Create nginx config
```bash
mkdir -p docker/nginx
```

Create `docker/nginx/default.conf`:
```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/html/public;
    index index.php index.html;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

### Step 5: Run these commands
```bash
chmod +x init-symfony.sh
docker compose up -d --build
```
NOTE: You need to wait for the symfony file to be created to the folder
Access: http://localhost:8080

## Vue Integration

### Step 6: Install Webpack Encore
```bash
docker exec symfony_php composer require symfony/webpack-encore-bundle
```

### Step 7: Install Node dependencies
```bash
docker exec symfony_node yarn install
```

### Step 8: Install Vue
```bash
docker exec symfony_node yarn add vue @vitejs/plugin-vue
docker exec symfony_node yarn add --dev @vue/compiler-sfc
docker exec symfony_node yarn add vue-loader@^17.0.0 --dev
```

### Step 9: Configure Encore for Vue
Edit `webpack.config.js`:
```js
const Encore = require('@symfony/webpack-encore');

Encore
    .setOutputPath('public/build/')
    .setPublicPath('/build')
    .addEntry('app', './assets/app.js')
    .enableVueLoader()
    .splitEntryChunks()
    .enableSingleRuntimeChunk()
    .cleanupOutputBeforeBuild()
    .enableSourceMaps(!Encore.isProduction())
    .enableVersioning(Encore.isProduction())
;

module.exports = Encore.getWebpackConfig();
```

### Step 10: Update assets/app.js
```js
import { createApp } from 'vue';
import App from './App.vue';

createApp(App).mount('#app');
```

### Step 11: Create assets/App.vue
```vue
<template>
  <div>
    <h1>{{ message }}</h1>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const message = ref('Hello from Vue in Symfony!')
</script>
```

### Step 12: Build assets
```bash
docker exec symfony_node yarn build
```

### Step 13: Create a Controller
Create `src/Controller/HomeController.php`:
```php
<?php

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

class HomeController extends AbstractController
{
    #[Route('/', name: 'app_home')]
    public function index(): Response
    {
        return $this->render('home/index.html.twig');
    }
}
```

### Step 14: Create the Template
Create `templates/home/index.html.twig`:
```twig
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Symfony + Vue App</title>
    {{ encore_entry_link_tags('app') }}
</head>
<body>
    <div id="app"></div>
    {{ encore_entry_script_tags('app') }}
</body>
</html>
```

### Step 15: Test the Application
```bash
# Rebuild containers if needed
docker compose up -d

# Build assets in watch mode for development
docker exec symfony_node yarn watch
```

Access: http://localhost:8080 - You should see "Hello from Vue in Symfony!"