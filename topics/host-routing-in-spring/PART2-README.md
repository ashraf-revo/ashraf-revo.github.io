# Host Routing part 2
previously we discussed host mapping , and we made simple project that route based on host header

now we will continue on this path and make this project convert domain to corresponding userId

### How we can achieve domain mapping to corresponding user

we can build a filter that execute before calling downstream services like other predefined filters in spring (`RewritePath`,`SetPath`,`SetRequestHostHeader`)


we need to build a filter like those in `gateway service`
in `GatewayApplication.java`
```java
package com.asrevo.domainrouting.gateway;

import lombok.Getter;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.net.URI;
import java.util.List;
import java.util.Map;

import static org.springframework.web.util.UriComponentsBuilder.fromUri;

@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

}

interface DomainRepository {
    Mono<String> findByDomainName(String hostName);
}

@Service
class DomainRepositoryImpl implements DomainRepository {
    private final Map<String, String> dbDomainMap = Map.of("john.com", "1");

    @Override
    public Mono<String> findByDomainName(String domain) {
        return Mono.justOrEmpty(dbDomainMap.get(domain));
    }
}

@Component
@Slf4j
class MapHostAsRequestParamGatewayFilterFactory extends AbstractGatewayFilterFactory<MapHostAsRequestParamGatewayFilterFactory.Config> {
    public static final String TEMPLATE_KEY = "template";
    private final DomainRepository repository;
    private final String MapHostAsRequestParamHeader = "Map-Host-As-Request-Param";

    public MapHostAsRequestParamGatewayFilterFactory(DomainRepository repository) {
        super(MapHostAsRequestParamGatewayFilterFactory.Config.class);
        this.repository = repository;
    }

    public List<String> shortcutFieldOrder() {
        return List.of(TEMPLATE_KEY);
    }


    @Override
    public GatewayFilter apply(MapHostAsRequestParamGatewayFilterFactory.Config config) {
        return (exchange, chain) -> {
            URI uri = exchange.getRequest().getURI();
            String hostName = exchange.getRequest().getHeaders().getHost().getHostName();
            if (checkingIfHostMappingNeeded(exchange, config)) {
                Mono<String> domainReference = this.repository.findByDomainName(hostName);
                try {
                    return domainReference.flatMap(itx -> {
                        if (itx != null && !itx.isEmpty()) {
                            ServerHttpRequest request = createHttpRequestWithNewParams(config, exchange, uri, itx);
                            return chain.filter(exchange.mutate().request(request).build());
                        } else {
                            log.warn("will reject " + uri + " with headers " + exchange.getRequest().getHeaders());
                            return response(exchange, HttpStatus.BAD_REQUEST, "invalid host header " + hostName);
                        }
                    });
                } catch (RuntimeException ex) {
                    throw new IllegalStateException("Invalid URI query: \"" + uri.getRawQuery() + "\"");
                }
            } else {
                return chain.filter(exchange);
            }
        };
    }

    private static Mono<Void> response(ServerWebExchange exchange, HttpStatus httpStatus, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(httpStatus);
        return response.writeWith(Mono.just(response.bufferFactory().wrap(message.getBytes())));
    }

    private static ServerHttpRequest createHttpRequestWithNewParams(Config config, ServerWebExchange exchange, URI uri, String itx) {
        StringBuilder query = new StringBuilder();
        String originalQuery = uri.getRawQuery();
        if (StringUtils.hasText(originalQuery)) {
            query.append(originalQuery);
            if (originalQuery.charAt(originalQuery.length() - 1) != '&') {
                query.append('&');
            }
        }
        String value = ServerWebExchangeUtils.expand(exchange, itx);
        query.append(config.getTemplate());
        query.append('=');
        query.append(value);
        URI newUri = fromUri(uri).replaceQuery(query.toString()).build(true).toUri();
        return exchange.getRequest().mutate().uri(newUri).build();
    }

    public boolean checkingIfHostMappingNeeded(ServerWebExchange exchange, MapHostAsRequestParamGatewayFilterFactory.Config config) {
        HttpHeaders headers = exchange.getRequest().getHeaders();
        return headers.containsKey(MapHostAsRequestParamHeader);
    }

    @Getter
    @Setter
    public static class Config {
        private String template;
    }
}
```
as you see 
- we build `MapHostAsRequestParamGatewayFilterFactory` that fill the `userId` that can be retrieved from domain 
  
 for example convert `http://john.com:8080/api/v1/user` to `http://localhost:8081/api/v1/user?userId=1`
- we build `DomainRepository` that could be implemented as db for you domain mapping or redis or in memory cache


and `application.yaml`
```yaml
spring:
  application:
    name: gateway
  cloud:
    gateway:
      default-filters:
        - MapHostAsRequestParam=userId
      routes:
        - id: host_route
          uri: http://localhost:8081
          predicates:
            - Host={domain}.{x},{sub}.{domain}.{x}
server:
  port: 8080
```
as you see
- we added to default-filters `MapHostAsRequestParam` with value `userId` which is the request param that passed to user service

testing it

```http request
GET http://john.com:8080/api/v1/user
Map-Host-As-Request-Param: true
```
you should see response like this 
```json
{
  "id": "1",
  "name": "jhon",
  "email": "jhon@email.com"
}
```

### Summary
now we can say that we build a sas solution 50% that allow any user in your system to serve his account under any domain

but we still have some issue that we will address later like https support because every domain we serve should support https ,and we need a way to address this issues like 
- configure our server to load certificate based on every request domain see [part3](https://blog.asrevo.com/topics/host-routing-in-spring/part3.html)
- caching this certificates for fast tls handshake (out of scope , you could use in memory store like [caffeine](https://github.com/ben-manes/caffeine))
- generate, renew and validate those certificates (out of scope but ,I can suggest using [acme4j&Encrypt](https://github.com/shred/acme4j)) and this article 
- store those certificates in central storage (out of scope , you could use any service like s3 for storing those certificate keys)
