<img src="https://help.veracode.com/internal/api/webapp/header/logo" width="200" /><br>  
  
# Verademo Apps on Docker  
  
## What is this about  
This a documentation how to run the Verademo Web App (https://gitlab.com/verademo-app/verademo-web), the Verademo API (https://gitlab.com/verademo-app/verademo-api) and the corresponding database for both of them in a dockerized environment.  
  
## How to run  
This is based on the 3 reposirotries on this group https://gitlab.com/verademo-app, you should clone all of them into the same root folder to get this structure  
<img src="https://gitlab.com/verademo-app/verademo-docker/-/raw/main/pictures/file_structure.png" />  
This way the example should right away and you don't need to adjust the docker files and folders.  
  
It makes use of `docker-compose`that lets you run multiple docker containers at once, all on the same network able to interact which each other. The main docker-compose.yml file is found on the root directory of this repository and will start the 3 metioned docker containers.  
  
In easy words you only need to run  
`docker-compose -f docker-compose.yml up`  
to start all 3 containers and  
`docker-compose -f docker-compose.yml down -v`  
to stop all 3 containers.  
  
There is no need to downlaod the base images upfront, the `docker-compose` process will make sure they will be downloaded.  
  
If you want to fully wipe everything and restart from sratch, make sure to also delete the corresponding docker images with `docker image rm IMAGE-ID`.  
  
Content of the docker-compose.yml:  
```yaml
version: '3.3'
services:
  web-app:
    build:
      context: ../verademo-web/target/
      dockerfile: ../../verademo-docker/docker/Dockerfile-Tomcat
    ports: 
      - "8080:8080"
    environment:
      CLASSPATH: /usr/local/tomcat/webapps/ROOT/WEB-INF/lib
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: blab
      MYSQL_USER: blab
      MYSQL_PASSWORD: "z2^E6J4$$;u;d"
    depends_on:
      - db

  api:
    build:
      context: ../verademo-api/
      dockerfile: ../verademo-docker/docker/Dockerfile-NodeJS
    ports: 
      - "8000:8000"
    depends_on:
      - db

  db:
    platform: linux/x86_64
    image: mysql:5.7
    volumes:
      - ./mysql:/var/lib/mysql
      - ./mysql-dump:/docker-entrypoint-initdb.d
      - ./blab_schema.sql:/etc/mysql/blab_schema.sql
      - ./mysqld.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf
    environment:
      MYSQL_ROOT_PASSWORD: 'root'
      MYSQL_DATABASE: 'blab'
      MYSQL_USER: 'blab'
      MYSQL_PASSWORD: 'z2^E6J4$$;u;d'
    ports:
      - 3306:3306
```  
  
Let's look at all 3 sections individually.  
  
### The Wep App section  
```yml
  web-app:
    build:
      context: ../verademo-web/target/
      dockerfile: ../../verademo-docker/docker/Dockerfile-Tomcat
    ports: 
      - "8080:8080"
    environment:
      CLASSPATH: /usr/local/tomcat/webapps/ROOT/WEB-INF/lib
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: blab
      MYSQL_USER: blab
      MYSQL_PASSWORD: "z2^E6J4$$;u;d"
    depends_on:
      - db
```
This will use the docker file `/docker/Dockerfile-Tomcat` and start a brand new tomcat container in version 9.0.58. It copies over the `verademo.war` file from the veradmo web app (https://gitlab.com/verademo-app/verademo-web). The container will be started with an open port on 8080 and sets a few environment variables.  
Once started you should have a running web applciation at http://YOUR-LOCAL-IP:8080/verademo.  
  
### The API Section  
```yml
  api:
    build:
      context: ../verademo-api/
      dockerfile: ../verademo-docker/docker/Dockerfile-NodeJS
    ports: 
      - "8000:8000"
    depends_on:
      - db
```
This will use the docker file `/docker/Dockerfile-NodeJS`and start a brand new node.js container in version 16. It copies over the verademo nod.js API from the veradmo-api repository (https://gitlab.com/verademo-app/verademo-api). The container will be started with an open port on 8000.  
Once started you should have a running API applciation at http://YOUR-LOCAL-IP:8000/.  
Please refere to the full documentation of the API https://gitlab.com/verademo-app/verademo-api.  
  
### The Database Section  
```yml
  db:
    platform: linux/x86_64
    image: mysql:5.7
    volumes:
      - ./mysql:/var/lib/mysql
      - ./mysql-dump:/docker-entrypoint-initdb.d
      - ./blab_schema.sql:/etc/mysql/blab_schema.sql
      - ./mysqld.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf
    environment:
      MYSQL_ROOT_PASSWORD: 'root'
      MYSQL_DATABASE: 'blab'
      MYSQL_USER: 'blab'
      MYSQL_PASSWORD: 'z2^E6J4$$;u;d'
    ports:
      - 3306:3306
```
This will use the base image `mysql:5.7`and start a brand new mysql container. It copies over a few things, like the database schema for the database and a mysql.cnf file to run on localhost. The container will be started with an open port on 3306.  
Once started you should have a running mysql database at YOUR-LOCAL-IP:3306.  
The mysql root user has the password `root`, the database for the web app and API is using a user called `blab`with password `z2^E6J4$;u;d`, note the doubel `$` one is used to XXXX the second. The real password is only with one `$`.  
The mysql start will create 2 new folders on your harddisk to store the mysql data. They are `/mysql/` and `/mysql-dump/`.
  
## Manual changes to be done  
Only one manual changes is requires to run this correctly on your local machine.  
  
The IP the API is running on has to be changed at `/verademo-api/config/db.config.js`, it still refelcts the IP my local machine was running on. The API should not run on `localhost` as you won't be able to scan with dynamic API scanning (https://docs.veracode.com/r/Using_Veracode_Dynamic_Analysis) and the ISM (https://docs.veracode.com/r/c_using_ism) in that case.  
```  
const { createPool } = require("mysql");
const db = createPool({
  port: 3306,
  host: "192.168.178.80",
  user: "blab",
  password: "z2^E6J4$;u;d",
  database: "blab",
  connectionLimit: 10,
});
module.exports = db;
```  
  
## The first run  
On the first run you should make sure to rest the database. This will either rest what is already on the database or add all required data to the database. The rest function can be found on the web app behind the rest button.  
<img src="https://gitlab.com/verademo-app/verademo-docker/-/raw/main/pictures/db_reset.png" />   
This also prepares the databased to be used by the API.  

## License  
[![MIT license](https://img.shields.io/badge/License-MIT-blue.svg)](license)  