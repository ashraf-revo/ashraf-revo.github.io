# Host Routing
 when you make your backend server or load-balancer support `subdomain` and `custom domain` routing dynamically, so it could support unlimited number of domains 

all of those predicates are very interesting, but we need to focus on  `HOST` routing because it deserves more focusing especially when you know use cases for this feature 
### Host routing use cases
- [help.shopify.com](https://help.shopify.com/en/manual/domains/add-a-domain)  use host routing to build `E-Commerce Website` for every user
- [help.medium.com](https://help.medium.com/hc/en-us/articles/115003053487-Setting-up-a-custom-domain-for-your-profile-or-publication)  use host routing to build `Blog` for every user
- [docs.github.com](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages)  use host routing to build `GitHub Pages` for every user

all of those use cases support `Subdomain` and `Custom domain`

### Subdomain vs Custom domain 
- Subdomain when you support domains from your main domain like `user1.github.com` and `user2.github.com` all of those share `github.com` domain suffix
- Custom domain when you support any domain like `example.com` they are not restricted to any domain suffix

# How we can build this solution?
spring cloud gateway has a feature called Route Predicate which is useful when you want to forward request to downstream servers based on some parameters

### Route Predicate
there a lot of route predicate that `spring-cloud-starter-gateway` support like 
- ### `PATH`
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: https://example.com
        predicates:
        - Path=/user
```
- ### `HOST`
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.com
        predicates:
        - Host={domain}.{x},{sub}.{domain}.{x}
```

all of those predicates are very interesting, but we need to focus on  `HOST` routing because it deserves more focusing

        - Host={domain}.{x},{sub}.{domain}.{x}
as you see it support `URI template variables` like `{sub}.{domain}.{x}` and multiple pattern using `,` separator


## Let's play with real code
we will build 3 microservices for this application
- gateway  (main entry point)
- user      (simple downstream service that react based on different hosts)
- ui        (service for showing how it works in browser)

first we need to configure out pc to forward some domains to localhost this will help us in local developments

```shell
sudo gedit /etc/hosts
```
then insert those content in the beaning of the file

```text
127.0.0.1	localhost
127.0.0.1	localhost.me
127.0.0.1	www.localhost.me
127.0.0.1	dev.localhost.me
```

this hack will make our pc network to forward `dev.localhost.me`,`www.localhost.me`,`localhost.me`  and other domain to `localhost`

you can find source code for next steps in `step1` branch

- then we will create `gateway service` with this content
`application.yaml`
```yaml
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
        - id: host_route
          uri: http://localhost:8081
          predicates:
            - Host={domain}.{x},{sub}.{domain}.{x}
server:
  port: 8080
```
`GatewayApplication.java`
```java
package com.asrevo.domainrouting.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

}
```
- and `user service` with this content
`application.yaml`
```yaml
spring:
  application:
    name: user
server:
  port: 8081
```
`UserApplication.java`
```java
package com.asrevo.domainrouting.user;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@SpringBootApplication
@RestController
@RequestMapping("api/v1/user")
public class UserApplication {
    @Autowired
    private UserRepository userRepository;

    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }

    @GetMapping("{id}")
    public User findOne(@PathVariable("id") String id) {
        return userRepository.findOne(id);
    }
}

record User(String id, String name, String email) {
}

interface UserRepository {
    User findOne(String id);
}

@Service
class UserRepositoryImpl implements UserRepository {
    private final Map<String, User> dbUserMap = Map.of("1", new User("1", "jhon", "jhon@email.com"));

    @Override
    public User findOne(String id) {
        return dbUserMap.get(id);
    }
}
```
testing it
