# Running Java applications on Ampere A1 compute platform 

This example shows how to run a containerized Java web application on the Ampere A1 compute platform using Apache Tomcat and MySQL. The Ampere A1 compute platform based on Ampere Altra CPUs represent a generational shift for enterprises and application developers that are building workloads that can scale from edge devices to cloud data centers. The unique design of this  platform delivers consistent and predictable performance as there are no resource contention within a compute core and offers more isolation and security. This new class of compute shapes on Oracle Cloud Infrastructure  provide an unmatched platform that combines power of the Altra CPUs with the security, scalability and eco-system of services on OCI.


Estimated time: 20 minutes

### Objectives

- Build and deploy a Java EE application on an Ampere A1 instance 

### Prerequisites

- Your Oracle Cloud Trial Account
- You have already created an Ampere A1 instance.
- You have already setup security rules and firewall options to enable application connectivity
  
## Run Java EE applications on Ampere A1 Compute Platform 

To run this application, first prepare an Ampere A1 compute instance with a few required packages, such as container tools and `git`. Then, clone the repository and build the application by using the included Maven `pom.xml`. Lastly, start the MySQL and Tomcat docker containers by using the container tools.

### Install the Container Tools

Oracle Linux 8 uses Podman to run and manage containers. Podman is a daemonless container engine for developing, managing, and running Open Container Initiative containers and container images on your Linux system. Podman provides a Docker-compatible command line application that can be used as a replacement for docker. Installing the `podman-docker` package provides the `docker` command that transparently invokes `podman`.

1. Login to the instance using SSH. Use the key either generated by you or provided during the instance creation step. The default username for instances using the Oracle Linux operating system is `opc`.

1. Install the `container-tools` module that pulls in all the tools required to work with containers.
    ```
    <copy>sudo dnf module install container-tools:ol8</copy>
    ```

    ```
    <copy>sudo dnf install podman-docker git</copy>
    ```

<!-- 1. Set SELinux to be in permissive mode so that Podman can easily interact with the host.
    
    **Note**: This is not recommended for production use. However, setting up SELinux policies for containers are outside the scope of this tutorial. For details, see the Oracle Linux 8 documentation.

    ```
    sudo setenforce 0
    ``` -->

### Clone the Source Code

To get started, use SSH to log in to the compute instance and clone the repository.

```
$<copy>git clone https://github.com/oracle-quickstart/oci-arch-tomcat-mds.git</copy>
$<copy>cd oci-arch-tomcat-mds/java</copy>
```

## Build the Web Application

Java web applications are packaged as web application archives, or WAR files. WAR files are zip files with metadata that describes the application to a servlet container like Tomcat. This example uses Apache Maven to build the WAR file for the application. 
To build the application, run the following command. Be sure to run the command from the location where the source files were cloned to.

```
$<copy>podman run -it --rm --name todo-build \</copy>
<copy>    -v "$(pwd)":/usr/src:z \</copy>
<copy>    -w /usr/src \</copy>
<copy>    maven:3 mvn clean install</copy>
```
This command creates a `target` directory and the WAR file inside it. Note that we aren’t installing Maven, but instead running the build tooling inside the container.

## Run the Application on the Ampere A1 Compute Platform

The application uses the Tomcat servlet container and the MySQL database. Both Tomcat and the MySQL database support the ARM64v8 architecture that the Ampere A1 compute platform uses.

1. Create a pod using Podman.
    ```
    $<copy>podman pod create --name todo-app -p 8080:8080 --infra-image k8s.gcr.io/pause:3.1</copy>
    ```

2. Start the database container in the pod.

    ```
    $<copy>podman run --pod todo-app -d \</copy>
    <copy>-e MYSQL_ROOT_PASSWORD=pass \</copy>
    <copy>-e MYSQL_DATABASE=demo \</copy>
    <copy>-e MYSQL_USER=todo-user \</copy>
    <copy>-e MYSQL_PASSWORD=todo-pass \</copy>
    <copy>--name todo-mysql \</copy>
    <copy>-v "${PWD}"/src/main/sql:/docker-entrypoint-initdb.d:z \</copy>
    <copy>mysql/mysql-server:8.0</copy>
    ```

    For the MySQL database, the database initialization scripts are provided to the container, which creates the required database users and tables at startup. This is done by mounting the `/src/main/sql` directory from the host as `/docker-entrypoint-initdb.d` inside the container. The official MySQL image you are using here is configured to execute `.sql` files in this directory at startup.  For more options, including how to export and back up data, see the [documentation](https://hub.docker.com/_/mysql).


3. Deploy the application that you built as a WAR file with a Tomcat server.
   
   ```
    $<copy>podman run --pod todo-app -d \</copy>
    <copy>--name todo-tomcat \</copy>
    <copy>-v "${PWD}"/target/todo.war:/usr/local/tomcat/webapps/todo.war:z \</copy>
    <copy> tomcat:9</copy>
    $<copy>podman logs -f todo-tomcat</copy>
   ```

    The database connect information and the application are provided to the Apache Tomcat container though the `src/main/resources/todo.properties`. The JDBC URL uses `localhost` as the MySQL server host. This is because containers within the same pod can communicate with each other using `localhost`. The application WAR file is provided as a mount to the container.

    Tomcat deploys the application on startup, and the port mapping to the host makes the application available over the public IP address for the compute instance.


4. Enter the public IP address of the compute instance in a browser with port `8080`. You should be able to see the application. `http://<ip_address>:8080/todo/`

## Troubleshooting

Podman containers can be inspected just like Docker containers (you can even alias `podman` as `docker`). Here are some common commands for inspecting the containers:

- `podman ps -pa` - shows running and exited containers, and the pods they belong to. 
- `podman logs -f todo-mysql` - shows the output from the specified container (`todo-mysql` in this example). Press `Ctrl+c` to exit.

## Next Steps

This tutorial covered how you can get started with building and deploying  Java and Java EE applications on the Ampere A1 platform. The Ampere A1 platform with up to 160 cores offers opportunities to vertically scale your multi-threaded applications and micro services. You can also move your existing workloads to the platform using the same approaches described in the tutorial. Both Java SE as well as GraalVM Enterprise Edition are included in the OCI subscription. When deploying production workloads, it is recommended that you move out from the containerized deployment of MySQL to the much more robust managed service that OCI provides. It provides automated backups and you can seamlessly deploy your database in a highly available manner. 