# DOCUMENTATION OF PROJECT 20
---

## MIGRATION TO THE СLOUD WITH CONTAINERIZATION. PART 1 – DOCKER & DOCKER COMPOSE

**DEFINTION OF TERMS**
- WHAT IS DOCKER?

Docker is a software platform that allows you to build, test, and deploy applications quickly. Docker packages software into standardized units called containers that have everything the software needs to run including libraries, system tools, code, and runtime.

WHAT IS DOCKER USED FOR?

Docker is an open platform for developing, shipping, and running applications. Docker enables you to separate your applications from your infrastructure so you can deliver software quickly. With Docker, you can manage your infrastructure in the same ways you manage your applications.

WHAT IS COMPOSE?

Docker Compose is a tool that was developed to help define and share multi-container applications. With Compose, we can create a YAML file to define the services and with a single command, can spin everything up or tear it all down.

**MIGRATION TO THE СLOUD WITH CONTAINERIZATION. PART 1 – DOCKER & DOCKER COMPOSE**
-

Until now, we have been using VMs (AWS EC2) in Amazon Virtual Private Cloud (AWS VPC) to deploy your web solutions, and it works well in many cases. You have learned how easy to spin up and configure a new EC2 manually or with such tools as Terraform and Ansible to automate provisioning and configuration. We have also deployed two different websites on the same VM; this approach is scalable, but to some extent; imagine what  we need to deploy many small applications (it can be web front-end, web-backend, processing jobs, monitoring, logging solutions, etc.) and some of the applications will require various OS and runtimes of different versions and conflicting dependencies – in such case we would need to spin up serves for each group of applications with the exact OS/runtime/dependencies requirements. When it scales out to tens/hundreds and even thousands of applications (e.g., when we talk of microservice architecture), this approach becomes very tedious and challenging to maintain.

In this project, we will learn how to solve this problem and practice the technology that revolutionized application distribution and deployment back in 2013! We are talking of Containers and imply Docker. Even though there are other application containerization technologies, Docker is the standard and the default choice for shipping your app in a container!

-
Install Docker and prepare for migration to the Cloud
First, we need to install Docker Engine, which is a client-server application that contains:
Check the output below:

![Update](./images/sudo-apt%20update.PNG)

![Docker](./images/installing%20docker%20containerd.PNG)

![Dependencies](./images/installing%20relevant%20dependencies.PNG)

A server with a long-running daemon process dockerd.
APIs that specify interfaces that programs can use to talk to and instruct the Docker daemon.
A command-line interface (CLI) client docker.
In this prpoject, i spinned up anew instance and assembled the application from the Database layer – I used a pre-built MySQL database container, configured it, and made sure it is ready to receive requests from our PHP application.

Step 1: Pull MySQL Docker Image from Docker Hub Registry
Start by pulling the appropriate Docker image for MySQL. You can download a specific version or opt for the latest release, as seen in the following command: 

```
docker pull mysql/mysql-server:latest
```

![pulled images](./images/pulled%20images%20from%20mysql.PNG)


![docker](./images/test%20running%20docker.PNG)

List the images to check that you have downloaded them successfully: with `docker image ls`

Step 2: Deploy the MySQL Container to your Docker Engine
Once you have the image, move on to deploying a new MySQL container with:

```
docker run --name <container_name> -e MYSQL_ROOT_PASSWORD=<my-secret-pw> -d mysql/mysql-server:latest
```
 See the output below:

- Replace <container_name> with the name of your choice. If you do not provide a name, Docker will generate a random one
The -d option instructs Docker to run the container as a service in the background
Replace <my-secret-pw> with your chosen password
In the command above, we used the latest version tag. This tag may differ according to the image you downloaded.

Then, check to see if the MySQL container is running: Assuming the container name specified is mysql-server with `docker ps -a`

**CONNECTING TO THE MYSQL DOCKER CONTAINER**
-
Step 3: Connecting to the MySQL Docker Container
We can either connect directly to the container running the MySQL server or use a second container as a MySQL client. Let us see what the first option looks like. Run `docker exec -it mysql mysql -uroot -p`. Provide the root password when prompted. With that, you’ve connected the MySQL client to the server. See the output below:

![connection to database](./images/successful%20connection%20to%20the%20database.PNG)

Finally, change the server root password to protect your database. Exit the the shell with exit command.

The second approach
-
Ensure the above container is properly removed/deleted.

First, create a network with:
```
docker network create --subnet=172.18.0.0/24 tooling_app_network
```

![Second approach](./images/second%20approach.PNG)
Creating a custom network is not necessary because even if we do not create a network, Docker will use the default network for all the containers you run. By default, the network we created above is of DRIVER Bridge. So, also, it is the default network. You can verify this by running the docker network ls command.

But there are use cases where this is necessary. For example, if there is a requirement to control the cidr range of the containers running the entire application stack. This will be an ideal situation to create a network and specify the --subnet

For clarity’s sake, we will create a network with a subnet dedicated for our project and use it for both MySQL and the application so that they can connect.

Run the MySQL Server container using the created network.
-
First, let us create an environment variable to store the root password: `export MYSQL_PW=` and confirm it is created. Then, pull the image and run the container and ensure the container is running, all in one command like below:
```
 docker run --network tooling_app_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW  -d mysql/mysql-server:latest
 ```

 To know the status of the container, run `docker ps -a`

 As you already know, it is best practice not to connect to the MySQL server remotely using the root user. Therefore, we will create an SQL script that will create a user we can use to connect remotely.
Create a file and name it create_user.sql and add the below code in the file:
```
  CREATE USER ''@'%' IDENTIFIED BY ''; GRANT ALL PRIVILEGES ON * . * TO ''@'%'; 
```
Run the script:
Ensure you are in the directory create_user.sql file is located or declare a path
```
  docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < create_user.sql 
```

Connecting to the MySQL server from a second container running the MySQL client utility
-
To run the MYSQL Client container, run `docker run --network tooling_app_network --name mysql-client -it --rm mysql mysql -h mysqlserverhost -u  -p `
 See the output below

 ![database accessed](./images/mysql%20database%20accssed.PNG)

 Prepare database schema
 -

Now you need to prepare a database schema so that the Tooling application can connect to it.
Clone the Tooling-app repository from here
 [tooling](https://github.com/darey-devops/tooling.git) 

On your terminal, export the location of the SQL file
  `export tooling_db_schema=/tooling_db_schema.sql`. You can find the `tooling_db_schema.sql` in the `tooling/html/tooling_db_schema.sql` folder of cloned repo and verify the path is properly exported. Verify with: ` echo $tooling_db_schema`.

Use the SQL script to create the database and prepare the schema. With the docker exec command, you can execute a command in a running container.
 $ docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema 

Update the .env file with connection details to the database
The .env file is located in the html tooling/html/.env folder but not visible in terminal. you can use vi or nano.

```
MYSQL_IP=mysqlserverhost
MYSQL_USER=username
MYSQL_PASS=client-secrete-password
MYSQL_DBNAME=toolingdb
```

**Run the Tooling App**
-
Containerization of an application starts with creation of a file with a special name - 'Dockerfile' (without any extensions). This can be considered as a 'recipe' or 'instruction' that tells Docker how to pack your application into a container. In this project, you will build your container from a pre-created Dockerfile, but as a DevOps, you must also be able to write Dockerfiles. You can watch the video [here](https://www.youtube.com/watch?v=hnxI-K10auY) to get an idea how to create your Dockerfile and build a container from it.

So, let us containerize our Tooling application; here is the plan:
Make sure you have checked out your Tooling repo to your machine with Docker engine
First, we need to build the Docker image the tooling app will use. The Tooling repo you cloned above has a Dockerfile for this purpose. Explore it and make sure you understand the code inside it.
- Run docker build command
- Launch the container with docker run
- Try to access your application via port exposed from a container

Ensure you are inside the directory "tooling" that has the file Dockerfile and build your container :
Run `docker build -t tooling:0.0.1 . ` See the output below:

![running tooling](./images/running%20the%20todo%20app.PNG)

Then, run the container with:
```
docker run --network tooling_app_network -p 8085:80 -it tooling:0.0.1 
```
See the output below:

![running container](./images/running%20the%20container.PNG)

If everything works, you can open the browser and type http://localhost:8085 or your instance IP address on port 8085

You will see the login page like the one below:

![Tooling website](./images/loogin%20page.PNG)

The default email is test@gmail.com, the password is 12345 or you can check users' credentials stored in the toolingdb.user table.

PRACTICE TASK
Practice Task №1 – Implement a POC to migrate the PHP-Todo app into a containerized application.
Download php-todo repository from here

The project below will challenge you a little bit, but the experience there is very valuable for future projects.

Part 1
Write a Dockerfile for the TODO app
Run both database and app on your laptop Docker Engine
Access the application from the browser
Part 2
Create an account in Docker Hub
Create a new Docker Hub repository
Push the docker images from your PC to the repository
Part 3
Write a Jenkinsfile that will simulate a Docker Build and a Docker Push to the registry
Connect your repo to Jenkins
Create a multi-branch pipeline
Simulate a CI pipeline from a feature and master branch using previously created Jenkinsfile
Ensure that the tagged images from your Jenkinsfile have a prefix that suggests which branch the image was pushed from. For example, feature-0.0.1.
Verify that the images pushed from the CI can be found at the registry.

Check below to see the results of the practice tasks:

![todo app](./images/Todo%20App.PNG)

![connecting to hub](./images/connecting%20to%20hub.PNG)

![docker hub](./images/docker%20hub.PNG)

This brings us to the end of project 20 implementation.