# TestContainers 사용기

### 배경

플랫폼 서버 개발은 서비스 개발과 비교했을 때 외부 서비스의 조율에 더 집중하게 됩니다. 하나의 사용자의 요청은 수많은 서비스들에 거쳐 처리가 되는 경우가 많고 이를 어떻게 노출할까에 대해 고민을 하기에, 어쩌면 외부 클라이언트와 내부 서비스간의 인터페이스 역할을 하는 일종의 API 게이트웨이 라고 할 수도 있습니다.

이런 플랫폼 서버의 특징으로 인해 각 서비스의 로직에 대해 분할하여 테스트하는 것보다는 외부 서비스의 변화에 따라 어떤 응답을 내려주는지 더 관심이 생기게 되고, 이런 이유로 통합테스트에 더 관심이 생기게 됩니다.

Spring 환경에서 통합테스트를 설계할때 여러가지 방법이 있지만, 아래의 조합을 고려해볼 수 있습니다.

* junit 5
* testContainer (MySQL)
* DB Rider



이중, Test Container 사용을 다뤄볼까 합니다.



### Test Container 설정

[Spring 공식문서](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html)에서, Bean 등록을 통한 테스트컨테이너 설정을 아래와 같이 할 수 있다 안내하고있습니다.

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

        @Bean
        @ServiceConnection
        MySQLContainer<?> mysqlContainer() {
                return new MySQLContainer<>(DockerImageName.parse("mysql:8.0.39"))
                                .withReuse(true);
        }
}

```

이후, 테스트 class 에서는 아래와 같이 사용합니다.

```java
@SpringBootTest
@Testcontainers
class AsdfTest {

    @Autowired
    private AsdfService AsdfService;
    
    @Test
    void testSomething() {
                                
    }
}
```

이 외에도 jdbc-url 을 아래와 같이 설정하여 사용할 수 있습니다.

```java
jdbc:tc:mysql:8.0.36:///databasename
```



#### init sql

여기서, 다음과 같은 상황이 있다고 해봅시다.

1. DB table 에 대한 세부 세팅이 DBA 리뷰 과정에서 수정되었고 코드에 반영되지 않아, 실제 DB의 설정과 JPA annotation의 설정이 일부 상이하다.
2. 코드 배포 없이 동적으로 요구사항을 반영하기 위해 일부 Enum 성격의 값을 DB로 관리한다. 그렇기에, 초기값들이 존재한다.

이런 상황의 경우, init.sql 등의 SQL 을 작성하여 테스트코드 수행 전 일괄로 테이블을 생성하고 데이터를 추가하는 것이 좋고, TestContainers 는 이런 경우 2가지 방법을 제공합니다.



1. jdbc url

```java
jdbc:tc:mysql:8.0.39:///databasename?TC_INITSCRIPT=sql/init.sql
```

2. MySQLContainer 설정시 initScript 사용

```java
mysqlContainer = new MySQLContainer<>("mysql:8.0.39")
        .withInitScript("sql/init.sql");
```



#### System call filtering

Github Actions 과 같이 worker를 여러대 사용하고, 이 worker들에 대한 설정을 접근하고 수정할 수 없을 경우가 종종 있습니다. 이런 환경에서 만약 TestContainer 를 실행하는데 실패하였지만 리소스의 문제가 아니라면 Linux 의 secure computing mode 에서 컨테이너를 띄울 때 필요한 시스템 콜이 필터링 되었을 가능성이 있습니다.

이런 경우에 일반적으로는 `seccomp=undefined` 와 같은 옵션을 추가해서 실행해볼 수 있습니다. 위에서 계속 보여드린 TestContainers 의 MySQLContainer 에는 `withCreateContainerCmdModifier` 를 제공해서 이를 이용해 위 옵션을 추가할 수 있습니다.

```
mysqlContainer = new MySQLContainer<>("mysql:8.0.39")
        .withCreateContainerCmdModifier(cmd -> cmd.getHostConfig().withSecurityOpts(List.of("seccomp=unconfined")))
        .withInitScript("sql/init.sql");
```

물론 테스트 환경이라 할지라도 이렇게 시스템 콜 보안 옵션을 모두 해제하는 것은 좋은 선택이 아닙니다. `module_request` 나 얼마전 bpf door 로 개발자 아닌 사람들도 많이 알게된 `bpf`, `reboot` 등과 같은 시스템 콜을 모두 권한을 주는 것이기에

```
.withCreateContainerCmdModifier(cmd -> 
    cmd.getHostConfig()
        .withCapAdd(Capability.SYS_NICE, Capability.SYS_RESOURCE)
)
```

이렇게 필요한 권한만 추가하는 것이 더 좋습니다. 