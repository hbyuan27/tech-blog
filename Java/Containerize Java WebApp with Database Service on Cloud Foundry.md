Containerize Java WebApp with Database Service on Cloud Foundry
========================================================================

Tags： Spring-Boot PostgreSQL Cloud-Foundry Docker Micro-Service

---
If you are new to Java development or Ubuntu, here is the guide - [***How To Setup Java IDE on Ubuntu***][1]

It would help you to setup the development environment, with a simple Spring-Boot web app running on it.

In this tutorials, we will move a local Java WebApp and its database instance into separated containers. And later deploy the containers on Cloud Foundry bind to a PostgreSQL service instance.

# **Install Docker (CE)**
Refer to offical guide - [Get Docker for Ubuntu][2].

# **Install Cloud Foundry CLI** (Command Line Interface)
- Add the Cloud Foundry Foundation public key and package repository to your system:

  ``` bash
  wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
  echo "deb http://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
  ```
- Update your local package index:

  `sudo apt-get update`

- Install the cf CLI:

  `sudo apt-get install cf-cli`

# **Build PostgreSQL Container as Database Server**
Now, instead of using the installed database server, we're going to use a docker container to run as the database server for the WebApp - which will also be running on a container too.
The benefits are:
- You don't need to install and manage a database server yourself.
- Easy to extend the scale of database instance.
- Easy to share storage between docker containers.
- Easy to manage multiple database connections for one application.

**Steps:**
- **Pull the official PostgresSQL image**

  `sudo docker pull postgres`

  From its [Dockerfile][3] we know, the data volume is already configured as `VOLUME /var/lib/postgresql/data`. There is no need to configure it again using `docker run -v` command.

  And from its [docker-entrypoint.sh][4] we know, the default user name and database name is **postgres**.

- **Create your own user and database**

  You can configure them via environment variables in an env-file (e.g. ***env.db*** ):
  ``` properties
  POSTGRES_DB=<your_db>
  POSTGRES_USER=<your_user>
  POSTGRES_PASSWORD=<your_pwd>
  ```

- **Start a PostgreSQL container**:

  Go to the file directory of ***env.db*** , run:

  `sudo docker run --name pg-db --env-file=env.db -d postgres`

# **Build Demo App Container**
- **Update Demo Project**

  Use Maven Docker Plugin to simply the build steps. In ***pom.xml*** :
  ``` xml
  <plugin>
  	<groupId>com.spotify</groupId>
  	<artifactId>docker-maven-plugin</artifactId>
  	<version>0.4.11</version>
  	<configuration>
  		<imageName>hbyuan27/demo</imageName>
  		<dockerDirectory>src/main/resources</dockerDirectory>
  		<resources>
  			<resource>
  				<targetPath>/</targetPath>
  				<directory>${project.build.directory}</directory>
                  <include>${project.build.finalName}.jar</include>
  			</resource>
  		</resources>
  	</configuration>
  </plugin>
  ```

- **Prepare the Dockerfile**

  ``` bash
  FROM frolvlad/alpine-oraclejdk8:slim
  ADD demo.jar app.jar
  ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
  ```
  Put it to the right place according to the configuration field "**dockerDirectory**" of Maven Docker Plugin.

- **Update Database Connection**

  Modify the Spring-Boot ***application.properties***
  ``` properties
  spring.datasource.url= jdbc:postgresql://${POSTGRES_PORT_5432_TCP_ADDR}:${POSTGRES_PORT_5432_TCP_PORT}/<your_db>
  spring.datasource.username=<your_user>
  spring.datasource.password=<your_pwd>
  spring.datasource.driverClassName=org.postgresql.Driver
  spring.jpa.hibernate.ddl-auto=create-drop
  ```

- **Build Your Demo App Image**

  Run `sudo mvn clean package docker:build`, you will get a new image in your docker registry.

  Run `sudo docker images` and you can see a image named ***hbyuan27/demo***

- **Start Your Demo App Container and Bind to Database Container**

  `sudo docker run --name demo --link pg-db:postgres -p 8080:8080 -d hbyuan27/demo`


**Run Demo App in Containers**

Now, open 'http://localhost:8080/hello' and 'http://localhost:8080/history' on your host machine, now the application container can do CRUD with your database container.

**A Useful Link**

[Manage Data in Containers][5]

***Optional - Take a Quiz***

Restart you demo container, you will find all the history records are gone. Why? And how to fix it?

#### [**GitHub Code - Standalone Docker WebApp**][6]

----------

# **Move Demo App to Cloud Foundry**
First, Cloud Foundry login:

```
cf api <your_api_endpoint>
cf login
```

- **Create a Database Service Instance for Demo App on Cloud Platform**

  Find all available database service in marketplace and create an instance:
  ```
  cf marketplace
  cf create-service postgresql v9.4-container demo-pg-servic
  ```

- **Build Database Connection Between DemoApp & Database Service**

  Now the database service is up and you need to find a way to build the connection between containers.
  On a cloud platform on top of Cloud Foundry, you may not able to specify the database name or other variables by yourself.
  What you can do is to fetch the variables and use them to build the connection.

  There are two ways to implement it:
 - **Using "VCAP_SERVICES"**
  When using the **cf create-service** command, the system will provide you a set of environment variables via a JSON object called "**VCAP_SERVICES**".
  You can find parameters from it and build the database connection like:

    ``` java
    Connection conn = null;
    JSONObject obj = new JSONObject(System.getenv("VCAP_SERVICES"));
    JSONArray arr = obj.getJSONArray("postgresql");
    String dbname = arr.getJSONObject(0).getJSONObject("credentials").getString("dbname");
    String hostname = arr.getJSONObject(0).getJSONObject("credentials").getString("hostname");
    String port = arr.getJSONObject(0).getJSONObject("credentials").getString("port");
    String username = arr.getJSONObject(0).getJSONObject("credentials").getString("username");
    String password = arr.getJSONObject(0).getJSONObject("credentials").getString("password");
    String connection_url = "jdbc:postgresql://" + hostname + ":" + port + "/" + dbname;

    DataSource dataSource = new org.apache.tomcat.jdbc.pool.DataSource();
    dataSource.setUrl(connection_url);
    dataSource.setUsername(username);
    dataSource.setPassword(password);
    conn = dataSource.getConnection();
    ```
  - **Using "Spring Cloud Connectors"**

    The connectors help you to parse the JSON object and automatically integrated with Spring Framework. What you need to do is to add below dependencies in you **pom.xml**.
    ``` xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-spring-service-connector</artifactId>
            <version>1.2.4.RELEASE</version>
        </dependency>
        <!-- If you intend to deploy the app on Cloud Foundry, add the following -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-cloudfoundry-connector</artifactId>
            <version>1.2.4.RELEASE</version>
        </dependency>
    </dependencies>
    ```
- **Rebuild Demo App Image**

  In this tutorials we choose to use "Spring Cloud Connectors" since it is easier and you don't need to re-write you code a lot.
  Create a Spring configuration bean to manage the database:
  ``` java
  package cloud.app.demo;

  import javax.sql.DataSource;

  import org.springframework.cloud.config.java.AbstractCloudConfig;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;

  // TODO @Profile("cloud")
  @Configuration
  public class CloudConfig extends AbstractCloudConfig {
  	@Bean
  	public DataSource dataSource() {
  		return connectionFactory().dataSource();
  	}
  }
  ```
  And update the App.java file to enable it:
  ``` java
  @SpringBootApplication
  public class App extends SpringBootServletInitializer {

  	public static void main(String[] args) {
  		SpringApplication.run(App.class, args);
  	}

  	@Override
  	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
  		return builder.sources(App.class).sources(CloudConfig.class);
  	}
  }
  ```

  Now comment out the old database connection setting in **application.properties**
  ``` properties
  #spring.datasource.url= jdbc:postgresql://${POSTGRES_PORT_5432_TCP_ADDR}:${POSTGRES_PORT_5432_TCP_PORT}/<your_db>
  #spring.datasource.username=<your_user>
  #spring.datasource.password=<your_pwd>
  #spring.datasource.driverClassName=org.postgresql.Driver
  spring.jpa.hibernate.ddl-auto=create-drop
  ```
  Don't forget to add the "Spring Cloud Connectors" dependencies into your pom.xml

  Now, remove the old docker image to save some disk space:
  
  `sudo docker rmi hbyuan27/demo` (stop and remove containers first)

  Rebuild a new demo app image:
  
  `sudo mvn clean package docker:build`

- **Push Demo App Image to Docker Hub**

  Login with your DockerHub account and push the demo app image to your registry:

  ```
  sudo docker login
  sudo docker images
  sudo docker push <hbyuan27/demo>
  ```

- **Deploy Demo App as Docker Container onto Cloud Foundry**
  - Create manifest file for deployment - ***manifest.yml***

    ``` yaml
    applications:
     - name: <your_app_name>
        path: .
        memory: 1024M
        instances: 1
        routes:
         - route: <your_route.xxx.com>
        services:
         - demo-pg-service
    ```

  - Goto the directory of the manifest file, run command:
  
    `sudo cf push -o <your_docker_hub_image>`

  - Deploy multiple apps with the same app route, which can be considered as a simple implementation of load balance on Cloud Foundry.

 - **Run Demo App in Containers on Cloud Foundry**

  Now open access the app through the route: `http://<your_route>/hello`, refresh the page, you will find different jvm names which means the load balance is working now.

#### [**GitHub Code - Docker Based Demo App on Cloud Foundry**][7]


----------


#### **TODO - Use @Profile to Make the Code Available for Any Env**
#### **TODO - Support Multiple Database Type**
#### **TODO - Share Stored Data Between Containers on Cloud Foundry**


  [1]: Setup%20Ubuntu%20Development%20Environment%20for%20Java%20WebApp.md
  [2]: https://docs.docker.com/engine/installation/linux/ubuntu/
  [3]: https://github.com/docker-library/postgres/blob/master/9.4/Dockerfile
  [4]: https://github.com/docker-library/postgres/blob/master/9.4/docker-entrypoint.sh
  [5]: https://docs.docker.com/engine/tutorials/dockervolumes/
  [6]: https://github.com/hbyuan27/spring-boot-webapp-demo/tree/standalone-docker
  [7]: https://github.com/hbyuan27/spring-boot-webapp-demo/tree/cf-docker
