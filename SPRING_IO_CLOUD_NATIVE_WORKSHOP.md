# Cloud Native Java Workshop

The accompanying code for this workshop is [on Github](http://github.com/joshlong/cloud-native-workshop)

## Setup  

> microservices, for better or for worse, involve a lot of moving parts. Let's make sure we can run all those things in this lab.

- You will need JDK 8, Maven, an IDE and Docker in order to follow along. Specify important environment variables before opening any IDEs: `JAVA_HOME`.
- Go to the [Spring Initializr](http://start.spring.io) and specify the latest milestone of Spring Boot 1.3 and then choose EVERY checkbox except those related to AWS or Consul, then click generate. In the shell, run `mvn -DskipTests=true clean install` to force the resolution of all those dependencies so you're not stalled later. Then, run `mvn clean install` to force the resolution of the test scoped dependencies. You may discard this project after you've `install`ed everything.

(_Some versions of this workshop will not use Docker_)

## 1. "Bootcamp"

> In this lab we'll take a look at building a basic Spring Boot application that uses JPA and Spring Data REST. We'll look at how to start a new project, how Spring Boot exposes functionality, and how testing works.

- Go to the [Spring Initializr](http://start.spring.io) and select H2, REST Repositories, JPA, Vaadin, Web. Select the latest Spring Boot 1.3.x version. give it an `artifactId` of `reservation-service`.
- Run `mvn clean install` and import it into your favorite IDE using Maven import.
- Add a simple entity (`Reservation`) and a repository (`ReservationRepository`)
- Map the repository to the web by adding  `org.springframework.boot`:`spring-boot-starter-data-rest` and then annotating the repository with `@RepositoryRestResource`
- Add custom Hypermedia links
- Write a simple unit test

### Questions:
- What is Spring? Spring, fundamentally, is a dependency injection container. This detail is unimportant. What is important is that once Spring is aware of all the objects - _beans_ - in an application, it can provide services to them to support different use cases like persistence, web services, web applications, messaging and integration, etc.
- Why `.jar`s and not `.war`s? We've found that many organizations deploy only one, not many, application to one Tomcat/Jetty/whatever. They need to configure things like SSL, or GZIP compression, so they end up doing that in the container itself and - because they don't want the versioned configuration for the server to drift out of sync with the code, they end up version controlling the application server artifacts as well as the application itself! This implies a needless barrier between dev and ops which we struggle in every other place to remove.   
- How do I access the `by-name` search endpoint? Follow the links! visit `http://localhost:8080/reservations` and scroll down and you'll see _links_ that connect you to related resources. You'll see one for `search`. Follow it, find the relevant finder method, and then follow its link.


##  3. The Config Server

> The [12 Factor](http://12factor.net/config) manifesto speaks about externalizing that which changes from one environment to another - hosts,  locators, passwords, etc. - from the application itself. Spring Boot readily supports this pattern, but it's not enough. In this lab, we'll loko at how to centralize, externalize, and dynamically update application configuration with the Spring Cloud Config Server.

- Go to the Spring Initializr, choose the latest milestone of Spring Boot 1.3.x, specify an `artifactId` of `config-service` and add `Config Server` from the list of dependencies.
- You should `git clone` the [Git repository for this workshop - https://github.com/joshlong/bootiful-microservices-config](`https://github.com/joshlong/bootiful-microservices-config.git`)
- In the Config Server's `application.properties`, specify that it should run on port 8888 (`server.port=8888`) and that it should manage the Git repository of configuration that lives in the root directory of the `git clone`'d  configuration. (`spring.cloud.config.server.git.uri=...`).
- Add `@EnableConfigServer` to the `config-service` `DemoApplication`
- Add `server.port=8888` to the `application.properties` to ensure that the Config Server is running on the right port for service to find it.
- Add the Spring Cloud BOM (you can copy it from the Config Server) to the `reservation-service`.

We will need to modify the `reservation-service`'s `pom.xml` in order to make it a config client. To do this, add the following to your `pom.xml` of the `reservation-service` from step #1.

Example:

    <dependencyManagement>
        <dependencies>
          ...
          <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Brixton.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
          </dependency>
        </dependencies>
    </dependencyManagement>

**IMPORTANT**: Make sure that you make this modification in the `</dependencyManagement>` block. There are two places where Maven dependencies are added in a `pom.xml` and some people tend to get confused at this step. If you need help, please raise your hand and flag down an instructor.

Next, add `org.springframework.cloud`:`spring-cloud-starter-config` to the `reservation-service`. We'll add this dependency declaration to the general dependencies tag, unlike the previous modification to `</dependencyManagement>`.

Example:

    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
      </dependency>
    ..
    </dependencies>

Next:

Create a `boostrap.properties` that lives in the same place as `application.properties` and discard the `application.properties` file. Now we need only to tell the Spring application where to find the Config Server, with the property `spring.cloud.config.uri=${config.server:http://localhost:8888}`, and how to identify itself to the Config Server and other services, later, with `spring.application.name`.

Now, run the Config Server:

> We'll copy and paste  `bootstrap.properties` for each subsequent module, changing only the `spring.application.name` as appropriate.

In the `reservation-service`, create a `MessageRestController` and annotate it with `@RefreshScope`. Inject the `${message}` key and expose it as a REST endpoint, `/message`.

Trigger a refresh of the message using the `/refresh` endpoint.

**EXTRA CREDIT**: Install RabbitMQ server and connect the microservice to the event bus by adding the `org.springframework.cloud`:`spring-cloud-starter-bus-amqp` then triggering the refresh using the `/bus/refresh`.

## 4. Service Registration and Discovery

> In the cloud, applications live and die as capacity dictates, they're ephemeral. Applications should not be coupled to the physical location of other services as this state is fleeting. Indeed, even if it were fixed, services may quickly become overwhelmed, so it's very handy to be able to specify how to load balance among the available instances or indeed ask the system to verify that there are instances at all. In this lab, we'll look at the low-level `DiscoveryClient` abstraction at the heart of Spring Cloud's service registration and discovery support.

- Go to the Spring Initializr, select the `Eureka Server` (this brings in `org.springframework.cloud`:`spring-cloud-starter-eureka-server`) checkbox, name it `eureka-service` and then add `@EnableEurekaServer` to the `DemoApplication` class.
- Make sure this module _also_ talks to the Config Server as described in the last lab by adding the `org.springframework.cloud`:`spring-cloud-starter-config`.
- add `org.springframework.cloud`:`spring-cloud-starter-eureka` to the `reservation-service`
- Add `@EnableDiscoveryClient` to the `reservation-service`'s `DemoApplication` and restart the process, and then confirm its appearance in the Eureka Server at `http://localhost:8761`
- Demonstrate using the `DiscoveryClient` API
- Use the Spring Initializr, setup a new module, `reservation-client`, that uses the Config Server (`org.springframework.cloud`:`spring-cloud-starter-config`), Eureka Discovery (`org.springframework.cloud`:`spring-cloud-starter-eureka`), and Web (`org.springframework.boot`:`spring-boot-starter-web`).
- Create a `bootstrap.properties`, just as with the other modules, but name this one `reservation-client`.
- Create a `CommandLineRunner` that uses the `DiscoveryClient` to look up other services programmatically


**EXTRA CREDIT**: Install [Consul](http://Consul.io) and replace Eureka with Consul. You could use `./bin/consul.sh`, but prepare yourself for some confusion around host resolution if you're running Docker inside a Vagrant VM.

## 5. Edge Services: API gateways (circuit breakers, client-side load balancing)
> Edge services sit as intermediaries between the clients (smart phones, HTML5 applications, etc) and the service. An edge service is a logical place to insert any client-specific requirements (security, API translation, protocol translation) and keep the mid-tier services free of this burdensome logic (as well as free from associated redeploys!)

> Proxy requests from an edge-service to mid-tier services with a _microproxy_. For some classes of clients, a microproxy and security (HTTPS, authentication) might be enough.

- Add `org.springframework.cloud`:`spring-cloud-starter-zuul` and `@EnableZuulProxy` to the `reservation-client`, then run it.
- Launch a browser and visit the `reservation-client` at `http://localhost:9999/reservation-service/reservations`. This is proxying your request to `http://localhost:8000/reservations`.

> API gateways are used whenever a client - like a mobile phone or HTML5 client - requires API translation. Perhaps the client requires coarser grained payloads, or transformed views on the data

- In the `reservation-service`, create a client side DTO - named `Reservation`, perhaps? - to hold the `Reservation` data from the service. Do this to avoid being coupled between client and service
- Add `org.springframework.boot`:`spring-boot-starter-hateoas`
- Add a REST service called `ReservationApiGatewayRestController` that uses the `@Autowired @LoadBalanced RestTemplate rt` to make a load-balanced call to a service in the registry using Ribbon.
- Map the controller itself to `/reservations` and then create a new controller handler method, `getReservationNames`, that's mapped to `/names`.
- In the `getReservationNames` handler, make a call to `http://reservation-service/reservations` using the `RestTemplate#exchange` method, specifying the return value with a `ParameterizedTypeReference<Resources<Reservation>>>` as the final argument to the `RestTemplate#exchange` method.
- Take the results of the call and map them from `Reservation` to `Reservation#getReservationName`. Then, confirm that `http://localhost:9999/reservations/names` returns the names.

> The code works, but it assumes that the `reservation-service` will always be up and responding to requests. We need to be a bit more defensive in any code that clients will connect to. We'll use a circuit-breaker to ensure that the `reservation-client` does something useful as a _fallback_ when it can't connect to the `reservation-service`.

- Add `org.springframework.boot`:`spring-boot-starter-actuator` and `org.springframework.cloud`:`spring-cloud-starter-hystrix` to the `reservation-client`
- Add `@EnableCircuitBreaker` to our `DemoApplication` configuration class
- Add `@HystrixCommand` around any potentially shaky service-to-service calls, like `getReservationNames`, specifying a fallback method that returns an empty collection.
- Test that everything works by killing the `reservation-service` and revisiting the `/reservations/names` endpoint
- Go to the [Spring Initializr](http://start.spring.io) and stand up a new service - with an `artifactId` of `hystrix-dashboard` - that uses Eureka Discovery, Config Client, and the Hystrix Dashboard.
- Identify it is as `hystrix-dashboard` in `bootstrap.properties` and point it to config server.
- Annotate it with `@EnableHystrixDashboard` and run it. You should be able to load it at `http://localhost:8010/hystrix.html`. It will expect a heartbeat stream from any of the services with a circuit breaker in them. Give it the address from the `reservation-client`: `http://localhost:9999/hystrix.stream`

