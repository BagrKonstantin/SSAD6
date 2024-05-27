# Доработка архитектуры приложения "Воронка" по документу "Серверное приложение “Воронка” МААК оценка архитектуры проекта"
*Поскольку проект разрабатывается под NDA, нет возможности предоставить доступ к репозиторию с кодом, поэтому 
в документ вставлены куски кода, потерпевшие изменение. 
Так же были скрыты некоторые переменные, что не должно повлиять на восприятие*

Скачать приложение: https://voronka-events.ru
## Репликация баз данных
*П3.2.1 Ранее было показано, что к некоторым базам данных одновременно обращаются
несколько микросервисов, что может значительно уменьшить производительность
Воронки. Для того, чтобы решить эту проблему предлагается настроить репликацию для
некоторых таблиц. В частности, в первую очередь репликация должна быть настроена
для всех таблиц микросервиса мероприятий.*

В сервисы мероприятий, образования, мероприятий пользователя, 
регистрации, утилит были добавлены встроенные базы данных. 
Репликация происходит при запуске сервиса.

Сервис организаторов не нуждается в этих изменениях так как на него нагрузка на порядки меньше.

Neo4jConfig.java
```java
@Configuration
@EnableTransactionManagement
public class Neo4jConfig {

    @Bean
    public Driver driver() {
        return GraphDatabase.driver("bolt://localhost:7689");
    }

    @Bean
    public Neo4jClient neo4jClient(Driver driver) {
        return Neo4jClient.create(driver);
    }

    @Bean
    public Neo4jTransactionManager transactionManager(Driver driver) {
        return new Neo4jTransactionManager(driver);
    }
}
```
В сервис организаторов был добавлен обменник rabbitMQ, отправляющий изменения 
мероприятий очередям сервисов мероприятий, мероприятий пользователя, регистрации, утилит
```java
public void sendEventUpdateMessage(Event event) {
        rabbitTemplate.convertAndSend("event-update-exchange", "", event );
    }
```
В сервис авторизации был добавлен обменник rabbitMQ, 
отправляющий изменения о пользователе сервисам образования, мероприятий пользователя,
регистрации
```java
public void sendEventUpdateMessage(User user) {
        rabbitTemplate.convertAndSend("user-update-exchange", "", user);
    }
```
Благодаря обменникам rabbitMQ очереди в перечисленных сервисах 
получают изменения произошедшие с ивентов или с пользователем, 
за счёт чего снижается количество обращений к базе данных
```java
@RabbitListener(queues = "eventcache.event.update")
    public void processEventChangeQueue(String messageString) throws JsonProcessingException {
        Event event = objectMapper.readValue(messageString, Event.class);
        
        Event updatedEvent = eventService.updateEvent(event);
        
        eventCacheService.updateEvent(updatedEvent);
    }
```
## Ослабление зависимости от сторонних сервисов
*П3.2.2 На текущий момент времени, в Воронке обращения к API
ЕЛК НИУ ВШЭ происходят во многих фрагментах кода, и потенциальное изменение API
ЕЛК НИУ ВШЭ приведет к необходимости внесения большого количества изменений. Для
решения этой проблемы предлагается создать Прокси-класс для обращения к API ЕЛК
НИУ ВШЭ, который будет проксировать запросы к этому API, и тогда в случае изменения
API, изменения будет необходимо внести только в этот прокси-класс*

UserHSEService.java
```java
@Data
public class UserHSEService {
    private static final String redirectURL = "https://api.voronka-events.ru/********";
    private final UserData userData;
    
    public UserHSEService(String code) {
        loginKeyClock(code);
    }

    public String loginKeyClock(String code) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

        MultiValueMap<String, String> map = new LinkedMultiValueMap<>();

        map.add("client_id", "*****************");
        map.add("grant_type", "authorization_code");

        map.add("redirect_uri", redirectURL);
        map.add("code", code);

        HttpEntity<MultiValueMap<String, String>> entity = new HttpEntity<>(map, headers);

        userData = restTemplate.postForObject(
                "https://saml.hse.ru/realms/hse/protocol/openid-connect/token",
                entity,
                ResponseHSEAppX.class,
                headers
        );
        if (userData == null) {
            throw new HSEError("hseResponse in null");
        }

    }
}

```


# Развёртывание
*П3.2.3 Существуют отдельные
микросервисы деплой которых отдельно (без деплоя других микросервисов) невозможен.
Например, задеплоить сервис авторизации невозможно без деплоя сервиса пользователя.
Для устранения такой проблемы необходимо уменьшить зависимости таких
микросервисов друг от друга, а также обернуть эти микросервисы в разные dockerконтейнеры, а также переписать docker-compose файлы. Кроме того, на текущий момент
деплой работает в полуавтоматическом формате, и с помощью изменения docker-compose
файлов можно автоматизировать процесс деплоя до конца.*

Проблема невозможности деплоя микросервисов была вызвана конфигом OpenFeign.
Для исправления был добавлен Resilience4j, а именно предохранитель, 
который вместо того что бы уронить сервис при старте будет пытаться
подключиться раз за разом. 
При этом статус сервиса будет unhealthy, до тех пор пока нужный сервис не запустится.

application.yaml
```yaml
...
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        retryer:
          period: 1000
          maxPeriod: 5000
          maxAttempts: 3
resilience4j:
  circuitbreaker:
    instances:
      exampleServiceClient:
        registerHealthIndicator: true
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowType: COUNT_BASED
        minimumNumberOfCalls: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        eventConsumerBufferSize: 10
...
```
Модифицируем FeignClient добавляя в него предохранитель

Resilience4jFeignClient.java
```java
public class Resilience4jFeignClient extends Client.Default {

    private final CircuitBreaker circuitBreaker;

    public Resilience4jFeignClient() {
        super(null, null);
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofMillis(10000))
                .build();
        this.circuitBreaker = CircuitBreaker.of("UserServiceFeignClient", config);
    }

    @Override
    public Response execute(Request request, Request.Options options) throws IOException {
        return CircuitBreaker.decorateCheckedSupplier(circuitBreaker, () -> super.execute(request, options)).get();
    }
}
```
FeignConfig.java
```java
@Configuration
public class FeignConfig {
    @Bean
    public Feign.Builder feignBuilder() {
        return Feign.builder()
                    .client(new Resilience4jFeignClient());
    }
}
```
Так же был добавлен компонент выкидывающий ошибку если нет подключения к сервису.

UserServiceFallback.java
```java
@Component
public class UserServiceFallback implements UserServiceFeignClient {
    @Override
    public User getUser() {
        throw new ServiceUnavaliableException("user-service is currently unavailable");
    }
}
```
И обработчик этой самой ошибки.
```java
@ExceptionHandler(ServiceUnavaliableException.class)
    @ResponseStatus(value = HttpStatus.SERVICE_UNAVAILABLE)
    public String handleException(ServiceUnavaliableException exception) {
        return exception.getMessage();
    }
```
Был сделан общий docker-compose для развертывания, обращающий внимание на здоровье сервисов
```yaml
...
  authentication-service:
    build:
      context: ./authentication-service
      dockerfile: Dockerfile
    restart: always
    ports:
      - "127.0.0.1:${AUTH_SERVICE_PORT}:${AUTH_SERVICE_PORT}"
    networks:
      - voronka
    environment:
      server.port: ${AUTH_SERVICE_PORT}

      spring.neo4j.uri: bolt://${NEO4J_IP}:${NEO4J_PORT}
      spring.neo4j.authentication.username: ${NEO4J_USERNAME}
      spring.neo4j.authentication.password: ${NEO4J_PASSWORD}
      spring.data.neo4j.database: ${NEO4J_DB}

    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://127.0.0.1:${AUTH_SERVICE_PORT}/api/v1/auth/actuator/health || exit 1
      interval: 10s
      timeout: 10s
      retries: 15
    depends_on:
      neo4j-db:
        condition: service_started
      user-service:
        condition: service_healthy
...
```

Такое же решение было добавлено в сервисы ивентов и ивентов пользователя (зависимость от медиа сервиса)
