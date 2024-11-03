In this documentation , I am going to show how to build a Wordpress stack with Traefik, Nginx, PHP-fpm, and MariaDB in Docker, using Docker-compose , also this project has another stack , which is the Docusaurus static web page and it using Nginx to serve it's files . in case of the URI request ( /doc ) , the Traefik routes traffic to this container. 

The project uses Traefik as a reverse proxy, routes traffic based on URL paths, and sets up different containers for each service.

<br>

### Project Overview

### This project uses Docker Compose to orchestrate a complex web service. It involves several services running in separate Docker containers, including:

1. WordPress for content management.
2. Nginx for serving WordPress and reverse proxying to other services.
3. phpMyAdmin and MariaDB for database management.
4. Docusaurus for serving documentation under `/doc`.
5. Traefik as a reverse proxy to route traffic across these services.


## **The directory structure of this project**

```jsx
/opt/project-F/
├── traefik/
│   └── docker-compose.yml
│   ├── traefik.yml
│
│
├── wordpress/
│   ├── nginx/
│   └── docker-compose.yml
│
├── database/
│   ├── docker-compose.yml
│   ├──database-data/
│
│
├── doc/
│   ├── Dockerfile
│   ├── docusaurus.config.js
│   └── docker-compose.yml
```

also i edit the hosts file on my local machine first with the ip address of remote machine to see the changes on my browser :

```jsx
172.18.179.246 mhd.com
```

# 1. Traefik - Reverse Proxy

### Traefik is a dynamic reverse proxy that routes requests to different containers based on the path or host headers. It handles all incoming traffic and determines which service should process the request.

## Traefik Duties

- Routing `/` requests to WordPress: Traefik listens for requests to the root ( mhd.com ), which should be routed to the WordPress/Nginx service.
- Routing `/doc` requests to Docusaurus: Traefik also intercepts requests to ( mhd`.com/doc)` and directs them to the Docusaurus container, serving static documentation.
- Load balancing and health checks: It can distribute traffic and check service health, though load balancing is optional in this project.

before we jump into our files , i want to mention that in each directory i created a " .env " file , which i created variables there and assign them a value . it make it easier to naming stuff
```jsx
CONTAINER_NAME= myapp

DATABASE_NAME= mhd_db

DATABASE_USERNAME= mhd_user

DATABASE_PASSWORD= mhdlnx

DATABASE_ROOT_PASSWORD= root_mhdlnx
```
## Traefik Configuration Files

so first i download the image of Traefik from docker hub - i use the version 3.2 .

`docker pull traefik:3.2

---

to manage this service , in the Traefik directory , we create a "dockercompose.yml" to manage this service in this multi container project and also create "traefik.yml" file for it's configuration and changing how it behaves

dockercompose.yml
```jsx


services:

traefik:

container_name: ${CONTAINER_NAME}-traefik

env_file: .env

restart: always

image: traefik:v3.2

ports:

- "80:80"

- "443:443"

- "8080:8080"

volumes:

- /var/run/docker.sock:/var/run/docker.sock

- ./traefik.yml:/etc/traefik/traefik.yml

- ./acme.json:/acme.json

networks:

- traefik-network

command: --configFile=/etc/traefik/traefik.yml

networks:

traefik-network:

external: true
```

traefik.yml
```jsx


api:

dashboard: true

insecure: true

entryPoints:

web:

address: ":80"

websecure:

address: ":443"

providers:

docker:

endpoint: "unix:///var/run/docker.sock"

exposedByDefault: false

network: "internal"
```

so let's explain what is happening inside these files . first i want to explain  " traefik.yml " file then we come back to it's dockercompose file :

### traefik.yml

we specify 3 sections :

- the api section , which inside it we can enable Traefik dashboard and also set it to access in insecure mode . ( because this is a test project , i gave it "true" value , but remember in the real project , you shouldn't , in that case anyone can access your dashboard easily ) .
- in the entrypoint section , we tell traefik which ports should listen to . which is 80 and 443
- in the provider section , we consider docker as a provider service which enables Traefik to connect to Docker API .

### dockercompose.yml

- restart flag : the " always " value always restarts the container if it stops , if we manually stop it , it's restarted only when docker daemon restarts .
- then we after implementing the image , we've got port section , so here we mapped the container's internal ports to the host machine's ports ( the 8080 port is for Traefik dashboard , which is useful for monitoring all services that Traefik handles )
- the volume part of this file :
    - in the first line , we mapped the docker socket location from local to container , which is used to connect Traefik to Docker API .
    - we also mapped the Traefik configuration file ( traefik.yml ) into the container .
- after volume part , we have to set the network that Traefik should use to communicate with other containers . in our project Traefik container only need to route traffic to " Nginx " if the URI requests is root or route traffic to " Docusaurus " container if the URI requests is "/doc" , so we should first create a external network to connect these containers ( Traefik ==> Nginx , Docusaurus )
    - we name our network " traefik-network" and also mention it as external in our dockercompose file and give it to Traefik service
        
        `docker network create traefik-network`
        
        ---
        
- after that , we just check if the traefik service works or not by using " docker compose up -d "
    
    `docker compose up -d`
    
    ---


# 2. WordPress & Nginx - Content Management

## The WordPress site is served via Nginx, which handles requests to( mhd.com ) . The WordPress container runs the backend (PHP-FPM).

about Nginx container , in this project we just use it as a proxy to serve the files from wordpress container , which itself get the data from mariadb container .

first we pull our images , we make sure to get the fpm version of wordpress image , which doesn't have webserver inside it . so we can use Nginx image separately .

`docker pull nginx:latest`

---

`docker pull wordpress:fpm`

---

so let's dive into our wordpress directory which include our dockercompose file for nginx and wordpress and also an nginx directory for nginx configuration , which we're going to map that directory later .

**dockercompose.yml**

```jsx
services:

nginx:

container_name: ${CONTAINER_NAME}-nginx

image: nginx:latest

restart: always

env_file: .env

labels:

- "traefik.enable=true"

- "traefik.http.routers.nginx-http.rule=Host(`mhd.com`)"

- "traefik.http.routers.nginx-http.entrypoints=web"

- "traefik.http.services.nginx-http-service.loadbalancer.server.port=80"

- "traefik.http.routers.nginx-https.rule=Host(`mhd.com`)"

- "traefik.http.routers.nginx-https.entrypoints=websecure"

- "traefik.http.routers.nginx-https.tls=true"

volumes:

- ./nginx/default.conf:/etc/nginx/conf.d/default.conf

- ./nginx/access.log:/var/log/nginx/access.log

- ./nginx/error.log:/var/log/nginx/error.log

- ./wordpress-data:/var/www/html

networks:

- nginx-wp-network

- traefik-network

wordpress:

container_name: ${CONTAINER_NAME}-wordpress

image: wordpress:fpm

restart: always

env_file: .env

environment:

WORDPRESS_DB_HOST: database:3306

WORDPRESS_DB_NAME: ${DATABASE_NAME}

WORDPRESS_DB_USER: ${DATABASE_USERNAME}

WORDPRESS_DB_PASSWORD: ${DATABASE_PASSWORD}

volumes:

- ./wordpress-data:/var/www/html

`networks:

- nginx-wp-network

- wp-db-network

networks:

nginx-wp-network:

wp-db-network:

external: true

traefik-network:

external: true
```

in this docker compose file , we've got 2 servcies : Nginx and Wordpress

NGINX SECTION :

- so i use environment variable to dynamically name the container . and also the container will automatically restart if stops which is useful for maintaining uptime .
- labels section :
    - these labels are used by Traefik to handle Nginx service . we've got two lines that defines the rule for routing HTTP and HTTPS requests . it will route traffic for mhd.com to this container . also we have two lines that specify HTTP and HTTPS traffic will enter through Traefik "web" entrypoint ( port 80 ) and "websecure" entrypoint ( port 443 ) .
    - the lines that includes "loadbalancer" tells Traefik that Nginx container listens on port 80 for HTTP traffic .
- nginx volumes section :
    - i mapped the nginx configuration , access log , error log so with these , i can access container's nginx configuration and also see the logs
    - in the last line , This volume is also defined in the `wordpress` service and acts as a shared directory between the Nginx and WordPress containers. By mounting this directory on both containers, Nginx can serve files that WordPress generates without needing to copy or transfer files between containers.
- networks section :
    - nginx container connects to two networks , an internal network that connects Nginx to the Wordpress container . and also an External network that allows Nginx container to communicate with Traefik for routing

WORDPRESS SECTION :

- so similar to nginx container , the container will use of environment variable for name and we also use official wordpress:fpm image , we should use the fpm version , because we don't want to use the version that have webserver inside it .
- environment section :
    - in this section we provide specific environment variable for wordpress . so we points wordpress to a database service running on database at port 3306 . and also use another variables that we've got in .env file inside our directory .
- volumes section :
    - This mounts " `./wordpress-data"`  on the host to "`/var/www/html"` in the WordPress container, which is the root directory where WordPress files are stored. By keeping this directory shared between both Nginx and WordPress, Nginx can access the files directly, allowing it to serve content generated or uploaded in WordPress.
- networks :
    - wordpress has access to two networks :
        - nginx-wp-network : for communication between nginx and wordpress internally .
        - wp-db-network : this is external network that connects wordpress to mariadb database .

at the end of the docker compose file , we defined our network :

`docker network create wp-db-network`

---

note that we don't need to create the internal networks with is not externals . the docker create them automatically .

next take a look at "default.conf" file for our nginx directory :

note that we don't need to create the internal networks with is not externals . the docker create them automatically .

next take a look at "default.conf" file for our nginx directory :

‍‍‍‍

**default.conf**

```jsx
server {

listen 80;

server_name mhd.com www.mhd.com;

access_log /var/log/nginx/access.log;

error_log /var/log/nginx/error.log;

root /var/www/html;

index index.php index.html;

location / {

`try_files $uri $uri/ /index.php$args;

}

location ~ \.php$ {

try_files $uri $uri/ =404;

fastcgi_split_path_info ^(.+\.php)(/.+)$;

fastcgi_pass wordpress:9000;

fastcgi_index index.php;

include fastcgi_params;

fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

fastcgi_param PATH_INFO $fastcgi_path_info;

}

}
```

# 3. Docusaurus - Documentation Service

## Docusaurus is a static site generator that hosts documentation. In this project, it handles requests to ( mhd.com/doc ) .

first we want to create the image for our static website  . we using a multi-stage build approach. It utilizes two stages: one for building the app using Node.js, and another for serving the built files with Nginx.

before we dive into it's docker file and configuration , we should have had two docker image to start :

`docker pull node:18-alpine`

---

---

first we have to install the docusaurus on our directory 

Before we start the installation, make sure you have the following installed:

1. **Node.js**: Docusaurus requires Node.js 14 or newer.
2. **Yarn**: While not strictly necessary since you can use npm, Yarn is recommended by the Docusaurus team.

we can install Node.js and Yarn with the following commands:

```bash
# Install Node.js
curl -sL <https://deb.nodesource.com/setup_14.x> | sudo -E bash -
sudo apt-get install -y nodejs

# Install Yarn
curl -sS <https://dl.yarnpkg.com/debian/pubkey.gpg> | sudo apt-key add -
echo "deb <https://dl.yarnpkg.com/debian/> stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update && sudo apt-get install yarn

```

### Installing Docusaurus

Once we have the prerequisites installed, we can create a new Docusaurus project:

1. Initialize a New Project:
Use the Docusaurus initialization command to create a new site. we can specify the template (classic, Facebook, or Bootstrap) according to your needs. The most commonly used template is "classic".
    
    ```bash
    npx create-docusaurus@latest my-website classic
    
    ```
    
    Replace `my-website` with your desired directory name.
    
2. Navigate to the Project Directory:
Change to the project directory that was created:
    
    ```bash
    cd my-website
    
    ```
    
3. Start the Development Server:
we can start the local development server to see your site in action:
    
    ```bash
    yarn start
    
    ```
    
    This command starts a local development server and opens up a browser window. Most changes are reflected live without having to restart the server.
    

### Next Steps

- **Build the Project:**
When we're ready to build the static files for our website, we can run:
    
    ```bash
    yarn build
    
    ```
    
    This command generates static content into the `build` directory and can be served using any static contents hosting service
    
    after these steps , we create Dockerfile and docker compose file :

### Dockerfille :

dockerfile

```jsx
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json /app

RUN HTTP_PROXY="[http://192.168.10.65:8080](http://192.168.10.65:8080/)"` `HTTPS_PROXY="[http://192.168.10.65:8080](http://192.168.10.65:8080/)"` `npm install --force

COPY . /app

#CMD ["npm","start"]

RUN npm run build

FROM nginx AS serve

COPY --from=builder /app/build /app

COPY nginx.conf /etc/nginx/conf.d/default.conf

CMD ["nginx", "-g", "daemon off;"]
```
---

builder stage:

- we use alpine version of node.js 18 as a base image .
- we set the working directory to /app inside the container .
- then we copy " package.json " from our host machine to the /app directory inside the contaier .
- we install the node.js dependencies with using a proxy server .
- then we copy all the other files to the /app directory in the container .
- at last , we run the build script that defined in the "package.json"file , which compiles the source code into a production version .

serve stage :

- in this stage we uses the official nginx image as base image . and we name this stage " serve " .
- so we copy the compiled files from /app/build directory in the "builder" stage into the "/app" directory of the nginx container .
- we had created an nginx file configuration into our root directory . so we mapped it to the configuration directory into container .
- since Nginx runs as a daemon in the background , which would cause the container to exit immediately after it starts , since Docker expects the main process in the container to run in the foreground .

### dockercompose.yml :

```jsx
services:

docusaurus:

build:

context: .

container_name: ${CONTAINER_NAME}-docusaurus

labels:

- "traefik.enable=true"

- "traefik.http.routers.docusaurus.rule=Host(`mhd.com`) && PathPrefix(`/doc`)"

- "traefik.http.services.docusaurus.loadbalancer.server.port=80"

networks:

- traefik-network

networks:

traefik-network:

external: true
```

---

in this docker compose file for our docusaurus project :

- 
    - we tell Docker to build teh Docusaurus container using the Docker file that is located in the current directory ( . ) . so docker will look for the Dockerfile in this directory to build the container .
- 
    - we also set the container name using environment variable .
    - we've got label section . so since Traefik acts as a reverse proxy , so it handles routing requests for this service .
        - we enable Traefik for this container ==> traefik.enable=true
        - the second line in labels indicates a routing rule for Traefik to forward any requests that match the host " mhd.com" and also have URI request ( /doc )
        - with "loadbalancer" line , we tell Traefik that should forward requests to port 80 of the docusaurus container .
- 
    - we also use external Traefik-network , so this container can communicate with Traefik reverse proxy .


# 4. phpMyAdmin & MariaDB - Database Management

### phpMyAdmin is used to manage the MariaDB database. The database stores all WordPress content, and phpMyAdmin provides a web interface for database management.

first we get our images :

`docker pull mariadb`

---

`docker pull phpmyadmin`

---

dockercompose.yml

```jsx
services:

database:

container_name: ${CONTAINER_NAME}-mariadb

image: mariadb

env_file: .env

restart: always

environment:

MYSQL_DATABASE: ${DATABASE_NAME}

MYSQL_USER: ${DATABASE_USERNAME}

MYSQL_PASSWORD: ${DATABASE_PASSWORD}

MYSQL_ROOT_PASSWORD: ${DATABASE_ROOT_PASSWORD}

networks:

- wp-db-network

volumes:

- ./database-data:/var/lib/mysql

phpmyadmin:

container_name: ${CONTAINER_NAME}-phpmyadmin

image: phpmyadmin

env_file: .env

environment:

PMA_HOST: database

PMA_PORT: 3306

MYSQL_ROOT_PASSWORD: ${DATABASE_ROOT_PASSWORD}

networks:

- wp-db-network

networks:

wp-db-network:

external: true
```

---

### Docker Compose for phpMyAdmin & MariaDB:

Sets up both the database and the database management interface, with separate containers. WordPress connects to the MariaDB service for data storage, and phpMyAdmin connects to MariaDB for administrative purposes

mariadb section :

- we set dynamic name for our container that fetched from ' .env ' file . and also we use the mariadb docker image for mariadb and phpmyadmin image for another container .
- environment section :
    - we set name , username , password and root password for mariadb database
- volume section :
    - in this section we Mapped a local folder (`./database-data`) to MariaDB's data directory inside the container (`/var/lib/mysql`)

phpmyadmin section :

- environment section :
    - `PMA_HOST`: Tells phpMyAdmin which host (container) it should connect to, which is the `database` service (MariaDB container).
    - `PMA_PORT`: The port number MariaDB is running on (default is `3306`).
    - `MYSQL_ROOT_PASSWORD`: The root password for MariaDB, allowing phpMyAdmin to authenticate as the root user.
- network :
    - phpMyAdmin is also attached to the `wp-db-network`, enabling it to communicate with the MariaDB container

---


# 5. How It All Comes Together

- Traefik: All traffic enters through Traefik, which listens on ports 80 (HTTP) and 443 (HTTPS). It routes requests based on the path and host.
    - Requests to m`hd.com` are routed to the WordPress/Nginx service.
    - Requests to mhd[`.com/doc`](http://example.com/doc) are routed to the Docusaurus container.
- Nginx (WordPress): For mhd[`.com`](http://example.com/) requests, Nginx serves the WordPress front-end. If WordPress needs to interact with the database, it connects to the MariaDB container.
- Docusaurus: For `/doc` requests, Traefik directs the traffic to the Docusaurus container, where the static site is served by Nginx.

- 
