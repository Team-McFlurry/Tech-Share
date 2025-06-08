# TestContainers 사용기

### 배경

플랫폼 서버 개발은 서비스 개발과 비교했을 때 외부 서비스의 조율에 더 집중하게 됩니다. 하나의 사용자의 요청은 수많은 서비스들에 거쳐 처리가 되는 경우가 많고 이를 어떻게 노출할까에 대해 고민을 하기에, 어쩌면 외부 클라이언트와 내부 서비스간의 인터페이스 역할을 하는 일종의 API 게이트웨이 라고 할 수도 있습니다.

이런 플랫폼 서버의 특징으로 인해 각 서비스의 로직에 대해 분할하여 테스트하는 것보다는 외부 서비스의 변화에 따라 어떤 응답을 내려주는지 더 관심이 생기게 되고, 이런 이유로 통합테스트에 더 관심이 생기게 됩니다.

Spring 환경에서 통합테스트를 설계할때 여러가지 방법이 있지만, 아래의 조합을 고려해볼 수 있습니다.

* junit 5
* testContainer (MySQL)
* DB Rider

Junit 5 는 모두 사용해보셨을테니, DB Rider을 간단히 집고 넘어가고 TestContainer 에 대해 이야기해보려고 합니다.

##### DB Rider

DB Rider 는 일종의 DB Unit 의 래퍼 라이브러리로, 아래와 같이 사용합니다.

```
@DataSet("dataset/common/bizinsight/bizinsight.yml")
@DBRider
@SpringBootTest
public class TestClass {

	@Test
	void getBizInsightContents() {
		// do something
	}
}

```

위와 같이 Class level 에 annotation 을 사용하면 하위 모든 테스트 실행 전에 아래 형식의 yml 파일에서 명시한 데이터로 세팅이 됩니다. 만약 상대적인 날짜나 null 등의 값을 넣고 싶다면, [dataset replacer](https://github.com/database-rider/database-rider?tab=readme-ov-file#310-dataset-replacers) 를 활용할 수 있습니다.

```
table_name:
  - id: 1
  	column_1: aa
  	name: "[null]"
  	end_at: "[DAY,YESTERDAY]"
```



dataset annotation 을 살펴보면 아래와 같습니다.

```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface DataSet {
  String[] value() default "";
  String executorId() default "";

  SeedStrategy strategy() default SeedStrategy.CLEAN_INSERT;
  boolean useSequenceFiltering() default true;

  String[] tableOrdering() default {};
  boolean disableConstraints() default false;
  boolean fillIdentityColumns() default false;
  String[] executeStatementsBefore() default {};
  String[] executeStatementsAfter() default {};
  String[] executeScriptsBefore() default {};
  String[] executeScriptsAfter() default {};

	boolean cleanBefore() default false;
  boolean cleanAfter() default false;

  boolean transactional() default false;
  Class<? extends DataSetProvider> provider() default DataSetProvider.class;
  String[] skipCleaningFor() default {};
  Class<? extends Replacer>[] replacers() default {};
}
```

눈여겨 볼 값들은 다음과 같습니다.

* strategy : CLEAN_INSERT 가 기본
* cleanBefore
* cleanAfter
* Transactional : 테스트 전체가 틀내잭션 내에서 실행, 테스트 중 생성 및 수정된 모든 데이터가 테스트 종료 후 트랜잭션 롤백 수행



원하는 시점에서 Dataset 을 롤백시킬 수 있다는 점을 확인할 수 있습니다. 하지만 눈치 채셨다시피, `AUTO_INCREMENT` 에 대해서는 초기화가 이루어지지 않습니다. 이 경우, Dataset annotation 에 의존하지 않고 보일러 플레이트 class 를 작성하고, 모든 테이블을 TRUNCATE 시키고 `FOREIGN_KEY_CHECKS` 를 1로 설정해주어 해결할 수 있습니다.



### Test Containers

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

@ServiceConnection annotation 을 통해 Spring boot 가 위 컨테이너의 접속 정보를 바탕으로 test property로 등록합니다. @DynamicPropertySource 를 등록하고, 별개 method 를 구현해야하는 보일러 플레이트 코드를 라이브러리화 한 어노테이션입니다.



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

```java
public class ContainerDatabaseDriver implements Driver {
    static {
        load();
    }

    private static void load() {
        try {
            DriverManager.registerDriver(new ContainerDatabaseDriver());
        } catch (SQLException e) {
            LOGGER.warn("Failed to register driver", e);
        }
    }

    @Override
    public boolean acceptsURL(String url) throws SQLException {
        return url.startsWith("jdbc:tc:");
    }

    @Override
    public synchronized Connection connect(String url, final Properties info) throws SQLException {
        /*
          The driver should return "null" if it realizes it is the wrong kind of driver to connect to the given URL.
         */
        if (!acceptsURL(url)) {
            return null;
        }
      ...
    }
}
```





@TestContainers annotation 은 단순 @Import(TestcontainersConfiguration.class) 를 단순화시킨 코드입니다.





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