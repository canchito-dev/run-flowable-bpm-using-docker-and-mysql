# Run Flowable BPM using Docker and MySQL
Execute Flowable Apps in a dockerized environment and using MySQL

## What you’ll need
*   About 30 minutes
*   A favorite IDE or Spring Tool Suite™ already install. In this tutorial, we used Visual Studio Code. You can download it from [here](https://code.visualstudio.com/download).
*   Docker Desktop for you operating system already installed. For this tutorial, we used Docker Desktop for Windows. You can download it from [here](https://docs.docker.com/docker-for-windows/install/).

## Introduction
Flowable is a set of very compact and highly efficient open source business process engines. They provide a workflow and Business Process Management (BPM) platform for developers, system admins and business users.

This lightweight and fast engine is written in Java. It is coupled with native Case Management (CMMN) and DMN engines. They are Apache 2.0 licensed open source, with a committed community.

In this article, you will learn how to compile Flowable source code and generate valid Docker images. Once you have the images, we will use Docker Compose for defining and running a multi-container Docker applications.

Because of license constraints, there are no official Flowable docker images with the MySQL connectors included. So, we will do some modifications to the source code, so that the images that we generate, include MySQL connectors by default.

So let’s get started!!

## Modifying the source code
Start by downloading the latest release of Flowable. You can get it from their official [Github](https://github.com/flowable/flowable-engine/releases) site. Once you have it, import it into your favorite IDE. In this tutorial, we used Visual Studio Code.

Once you have imported it, you will need to modify some _pom.xml_ files found in different directories.

_Root > pom.xml_

By default, Flowable engine uses an in-memory database to store execution and historical data while running process instances. Since we will be using MySQL, we need to add the specific database driver dependency.

Go to `root folder`, and modify the _pom.xml_ file as shown below. Basically, we are modifying the H2 dependency’s scope from runtime to test, and the MySQL dependency’s scope from test to runtime.

```xml
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>1.4.197</version>
    <scope>test</scope>
  </dependency>

  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.27</version>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

Next, you will have to modify individual _pom.xml_ file for each Flowable’s apps. Look for the docker image build and push section. As you will notice, by default, it uses Postgre database and driver. Modify it to use MySQL’s, just as in the below snipped.

```xml
<!-- docker image build and push -->
<profile>
  <id>docker</id>
  <dependencies>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <scope>compile</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      …
      …
    <plugins>
  <build>
<profile>
```

You will have to do this for the _pom.xml_ found under:

```
Root
  |- modules
    |- flowable-app-rest
    |- flowable-ui-admin-app
    |- flowable-ui-idm-app
    |- flowable-ui-modeler-app
    |- flowable-app-rest
```

That’s it. Now you are ready to build Flowable’s images.

## Building Flowable’s images
Until now, we have only prepared Flowable to include MySQL’s dependency. It is time to build the source code, and create a Docker image from it.

To do so, the Flowable Team was kind enough to create some scripts that do this for us. Under the `root folder`, you will see a `docker directory`. Inside you will see a script called `build-all-images.sh`. I recommend you to edit it, and include the skip test. Otherwise it takes too long to build and create the images. In the end, the script should look something like this:

```
#!/bin/bash
echo "Building all Java artifacts"
cd ..
mvn -Pdistro clean install -DskipTests

echo "Building REST app image"
cd modules/flowable-app-rest
mvn -Pdocker,swagger clean package -DskipTests

echo "Building ADMIN app image"
cd ../flowable-ui-admin
mvn -Pdocker clean package -DskipTests

echo "Building IDM app image"
cd ../flowable-ui-idm
mvn -Pdocker clean package -DskipTests

echo "Building MODELER app image"
cd ../flowable-ui-modeler
mvn -Pdocker clean package -DskipTests

echo "Building TASK app image"
cd ../flowable-ui-task
mvn -Pdocker clean package -DskipTests

echo "Done..."
```

If you want, you could add the `-U`, to force all the dependencies to be updated.

## Creating the docker compose yml file

By now, you should be able to see all of Flowable’s application images in your repository. Let’s list them.

```
$ docker image ls
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
flowable/flowable-task      latest              08a369ab9e95        45 hours ago        179MB
flowable/flowable-modeler   latest              c87376d3b49f        45 hours ago        169MB
flowable/flowable-idm       latest              ba444b20d379        45 hours ago        162MB
flowable/flowable-admin     latest              eb1f12567a13        46 hours ago        170MB
flowable/flowable-rest      latest              9f81bec4177a        46 hours ago        170MB
```

As mentioned in the before, we will use Docker Compose for defining and running a multi-container Docker applications. Here is the `docker-compose.yml` file that we will use.

```
version: '3.7'
services:

    flowable-db:
        image: mysql:5.7.26
        container_name: flowable-mysql-5.7.26
        volumes:
            - db_data:/var/lib/mysql
        environment:
            MYSQL_ROOT_PASSWORD: flowable
            MYSQL_DATABASE: flowable
            MYSQL_USER: flowable
            MYSQL_PASSWORD: flowable
        ports:
            - 3306:3306

    adminer:
        image: adminer
        ports:
            - 18080:8080

    flowable-rest-app:
        image: flowable/flowable-rest
        depends_on:
            - flowable-db
        environment:
            - SERVER_PORT=9977
            - SPRING_DATASOURCE_DRIVER-CLASS-NAME=com.mysql.jdbc.Driver
            - SPRING_DATASOURCE_URL=jdbc:mysql://flowable-db:3306/flowable?autoReconnect=true
            - SPRING_DATASOURCE_USERNAME=flowable
            - SPRING_DATASOURCE_PASSWORD=flowable
            - FLOWABLE_REST_APP_ADMIN_USER-ID=rest-admin
            - FLOWABLE_REST_APP_ADMIN_PASSWORD=test
            - FLOWABLE_REST_APP_ADMIN_FIRST-NAME=Rest
            - FLOWABLE_REST_APP_ADMIN_LAST-NAME=Admin
        ports:
            - 9977:9977
        entrypoint: ["./wait-for-something.sh", "flowable-db", "3306", "MySQL", "java", "-jar", "app.war"]

    flowable-idm-app:
        image: flowable/flowable-idm
        container_name: flowable-idm
        depends_on:
            - flowable-db
        environment:
            - SERVER_PORT=8080
            - SPRING_DATASOURCE_DRIVER-CLASS-NAME=com.mysql.jdbc.Driver
            - SPRING_DATASOURCE_URL=jdbc:mysql://flowable-db:3306/flowable?autoReconnect=true
            - SPRING_DATASOURCE_USERNAME=flowable
            - SPRING_DATASOURCE_PASSWORD=flowable
            - FLOWABLE_REST_APP_ADMIN_USER-ID=rest-admin
            - FLOWABLE_REST_APP_ADMIN_PASSWORD=test
            - FLOWABLE_REST_APP_ADMIN_FIRST-NAME=Rest
            - FLOWABLE_REST_APP_ADMIN_LAST-NAME=Admin
        ports:
            - 8080:8080
        entrypoint: ["./wait-for-something.sh", "flowable-db", "3306", "MySQL", "java", "-jar", "app.war"]

    flowable-task-app:
        image: flowable/flowable-task
        container_name: flowable-task
        depends_on:
            - flowable-db
            - flowable-idm-app
        environment:
            - SERVER_PORT=9999
            - SPRING_DATASOURCE_DRIVER-CLASS-NAME=com.mysql.jdbc.Driver
            - SPRING_DATASOURCE_URL=jdbc:mysql://flowable-db:3306/flowable?autoReconnect=true
            - SPRING_DATASOURCE_USERNAME=flowable
            - SPRING_DATASOURCE_PASSWORD=flowable
            - FLOWABLE_COMMON_APP_IDM-URL=http://flowable-idm-app:8080/flowable-idm
            - FLOWABLE_COMMON_APP_IDM-REDIRECT-URL=http://localhost:8080/flowable-idm
            - FLOWABLE_COMMON_APP_IDM-ADMIN_USER=admin
            - FLOWABLE_COMMON_APP_IDM-ADMIN_PASSWORD=test
        ports:
            - 9999:9999
        entrypoint: ["./wait-for-something.sh", "flowable-db", "3306", "MySQL", "java", "-jar", "app.war"]

    flowable-modeler-app:
        image: flowable/flowable-modeler
        container_name: flowable-modeler
        depends_on:
            - flowable-db
            - flowable-idm-app
            - flowable-task-app
        environment:
            - SERVER_PORT=8888
            - SPRING_DATASOURCE_DRIVER-CLASS-NAME=com.mysql.jdbc.Driver
            - SPRING_DATASOURCE_URL=jdbc:mysql://flowable-db:3306/flowable?autoReconnect=true
            - SPRING_DATASOURCE_USERNAME=flowable
            - SPRING_DATASOURCE_PASSWORD=flowable
            - FLOWABLE_COMMON_APP_IDM-URL=http://flowable-idm-app:8080/flowable-idm
            - FLOWABLE_COMMON_APP_IDM-REDIRECT-URL=http://localhost:8080/flowable-idm
            - FLOWABLE_COMMON_APP_IDM-ADMIN_USER=admin
            - FLOWABLE_COMMON_APP_IDM-ADMIN_PASSWORD=test
            - FLOWABLE_MODELER_APP_DEPLOYMENT-API-URL=http://flowable-task-app:9999/flowable-task/app-api
        ports:
            - 8888:8888
        entrypoint: ["./wait-for-something.sh", "flowable-db", "3306", "MySQL", "java", "-jar", "app.war"]

    flowable-admin-app:
        image: flowable/flowable-admin
        container_name: flowable-admin
        depends_on:
            - flowable-db
            - flowable-idm-app
            - flowable-task-app
        environment:
            - SERVER_PORT=9988
            - SPRING_DATASOURCE_DRIVER-CLASS-NAME=com.mysql.jdbc.Driver
            - SPRING_DATASOURCE_URL=jdbc:mysql://flowable-db:3306/flowable?autoReconnect=true
            - SPRING_DATASOURCE_USERNAME=flowable
            - SPRING_DATASOURCE_PASSWORD=flowable
            - FLOWABLE_COMMON_APP_IDM-URL=http://flowable-idm-app:8080/flowable-idm
            - FLOWABLE_COMMON_APP_IDM-REDIRECT-URL=http://localhost:8080/flowable-idm
            - FLOWABLE_COMMON_APP_IDM-ADMIN_USER=admin
            - FLOWABLE_COMMON_APP_IDM-ADMIN_PASSWORD=test
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_PROCESS_SERVER-ADDRESS=http://flowable-task-app
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_PROCESS_PORT=9999
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_PROCESS_CONTEXT-ROOT=flowable-task
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_PROCESS_REST-ROOT=process-api
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CMMN_SERVER-ADDRESS=http://flowable-task-app
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CMMN_PORT=9999
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CMMN_CONTEXT-ROOT=flowable-task
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CMMN_REST-ROOT=cmmn-api
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_DMN_SERVER-ADDRESS=http://flowable-task-app
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_DMN_PORT=9999
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_DMN_CONTEXT-ROOT=flowable-task
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_DMN_REST-ROOT=dmn-api
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_FORM_SERVER-ADDRESS=http://flowable-task-app
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_FORM_PORT=9999
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_FORM_CONTEXT-ROOT=flowable-task
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_FORM_REST-ROOT=form-api
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CONTENT_SERVER-ADDRESS=http://flowable-task-app
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CONTENT_PORT=9999
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CONTENT_CONTEXT-ROOT=flowable-task
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_CONTENT_REST-ROOT=content-api
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_APP_SERVER-ADDRESS=http://flowable-task-app
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_APP_PORT=9999
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_APP_CONTEXT-ROOT=flowable-task
            - FLOWABLE_ADMIN_APP_SERVER-CONFIG_APP_REST-ROOT=app-api
        ports:
            - 9988:9988
        entrypoint: ["./wait-for-something.sh", "flowable-db", "3306", "MySQL", "java", "-jar", "app.war"]

volumes:
    db_data: {}
```
In the file, we specify container for MySQL, and the Flowables apps. Notice that we have included [adminer](https://www.adminer.org/en/) database manager, so that we can access the database from a web interface.

One very important thing to notice, is that each container has an entry point, which executes a script called `wait-for-something.sh` (thanks to Flowable’s team). The script simple waits until the database is up, before starting Flowable’s app. Here is a copy of the script:

```
#!/bin/ash
set -e

host="$1"
port="$2"
description="$3"
shift 3
cmd="$@"

until nc -z "$host" "$port"; do
    echo "$description is unavailable - sleeping"
    sleep 1
done

>&2 echo "$description is up - executing command"
exec $cmd
```
Finally, we are ready to start Flowable. Execute the following command in the same directory where you created the `docker-compose.yml` file and `wait-for-something.sh` script:

```
$ docker-compose up -d
Starting flowable_adminer_1           ... done
Starting flowable-mysql-5.7.26        ... done
Starting flowable-idm                 ... done
Starting flowable_flowable-rest-app_1 ... done
Starting flowable-task                ... done
Starting flowable-admin               ... done
Starting flowable-modeler             ... done
```

Congratulations, you have successfully built and deployed Flowable’s app with MySQL.

## Accessing Flowable’s apps
Here is a list of the URLs that you need to access Flowable’s apps and adminer:

*   flowable-rest: [http://localhost:9977/flowable-rest](http://localhost:9977/flowable-rest)
*   flowable-rest-swagger: [http://localhost:9977/flowable-rest/docs](http://localhost:9977/flowable-rest/docs)
*   flowable-idm: [http://localhost:8080/flowable-idm](http://localhost:8080/flowable-idm)
*   flowable-task: [http://localhost:9999/flowable-task](http://localhost:9999/flowable-task)
*   flowable-modeler: [http://localhost:8888/flowable-modeler](http://localhost:8888/flowable-modeler)
*   flowable-admin: [http://localhost:9988/flowable-admin](http://localhost:9988/flowable-admin)
*   adminer: [http://localhost:18080/](http://localhost:18080/)

## Summary
We hope that, even though this was a very basic introduction, you understood how to build and create a docker image of all of Flowable’s app, and how to deploy them in a docker compose environment.

Please feel free to contact us. We will gladly response to any doubt or question you might have.

The full implementation of this article can be found in the GitHub repository.
