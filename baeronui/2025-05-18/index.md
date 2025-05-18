# Virtual Thread 살펴보기

### 배경

저희 부서는 다른 서비스들의 자산을 노출하고, 관리할 수 있는 대문격의 서비스를 제공합니다. 광고로 활용할 수 있는 자산이 총 11여곳에 달하며, 각 자산을 관리하는 부서들이 MQ 를 활용한 연동에 부정적이라 api 를 활용해 연동하고 있습니다. 이로 인해 저희가 작성한 각 API 는 상당한 I/O-bounded 한 동작을 하고 있습니다. 당시 Jdk21 이 새로 나왔던 시기였고, 저희 부서에서는 web-flux 의 선택을 고민하고 있던 시기라 그때의 분석을 진행하였습니다. Virtual thraed 와 web flux 를 고민하며 발견한 Virtual thread 의 한계점에 대해 이야기를 하려고 합니다. 

우선, Virtual Thread 에 대해 간단히 살펴봅시다. Virtual Thread 는 JVM 1.3 이전의 Green Thread 와 이후의 Platform Thread 를 엮어 만든 Light weight Thread 입니다. 기존 Green Thread 에서는 JNI 를 사용할 경우, 전체 thread 가 blocking 되는 이슈가 있었습니다. 구조상으로 이를 해결하지는 못했기에, JDK 21 의 virtual thread 도 동일한 이슈가 발생합니다. synchronized 시에도 pinning 이 되는 새로운 이슈가 발생하였지만, 이는 단순 JVM 상에서의 버그여서 JDK 25 에서는 이를 해소하는 패치가 들어옵니다.

```java
private static ForkJoinPool createDefaultScheduler() {
    ForkJoinWorkerThreadFactory factory = pool -> {
        PrivilegedAction<ForkJoinWorkerThread> pa = () -> new CarrierThread(pool);
        return AccessController.doPrivileged(pa);
    };
    PrivilegedAction<ForkJoinPool> pa = () -> {
        int parallelism, maxPoolSize, minRunnable;
        String parallelismValue = System.getProperty("jdk.virtualThreadScheduler.parallelism");
        String maxPoolSizeValue = System.getProperty("jdk.virtualThreadScheduler.maxPoolSize");
        String minRunnableValue = System.getProperty("jdk.virtualThreadScheduler.minRunnable");
        if (parallelismValue != null) {
            parallelism = Integer.parseInt(parallelismValue);
        } else {
            parallelism = Runtime.getRuntime().availableProcessors();
        }
        if (maxPoolSizeValue != null) {
            maxPoolSize = Integer.parseInt(maxPoolSizeValue);
            parallelism = Integer.min(parallelism, maxPoolSize);
        } else {
            maxPoolSize = Integer.max(parallelism, 256);
        }
        if (minRunnableValue != null) {
            minRunnable = Integer.parseInt(minRunnableValue);
        } else {
            minRunnable = Integer.max(parallelism / 2, 1);
        }
        Thread.UncaughtExceptionHandler handler = (t, e) -> { };
        boolean asyncMode = true; // FIFO
        return new ForkJoinPool(parallelism, factory, handler, asyncMode,
                     0, maxPoolSize, minRunnable, pool -> true, 30, SECONDS);
    };
    return AccessController.doPrivileged(pa);
}
```

Virtual thread 는 n 개의 플랫폼 스레드에 virtual thread 를 스케쥴링 하는 구조입니다. 여기를 조금 더 집고 넘어들어가면, ForkJoinPool 를 Virtual Thread scheduling 할때 사용합니다. 설정을 조금 더 들여다 보면, 전체 platform Thread 의 수는 저희가 설정한 parallelism 값부터 시작해서 maxPoolSize 수 까지 증가할 수 있습니다. 그리고, 이 ForkJoinPool 의 설정을 변경할 수 있는 방법은 현 시점에서는 전무한 상태입니다.

그리고 위 코드에서 `maxPoolSize = Integer.max(parallelism, 256);` 를 확인해보면, maxPoolSize 를 parallism 과 동일하게 설정하여도 일반적인 상황에서는 결국 256을 사용하게 됩니다.

Virtual Thread 에 대해 더 이야기하기 전에 Web-flux 를 이 시점에 살펴보려고 합니다.

![netty](./baeronui/2025-05-18/netty.png)

WebFlux 는 netty 를 내장하여 사용하고, 전체적인 구조는 위와 같습니다. 이후, 아래와 같은 동작을 합니다.

```
1. 클라이언트가 요청을 하면 Boss EventLoopGroup이 connection establish 하고, NioSocketChannel 을 만든다.

2. 이 Channel 을 Worker EventLoopGroup(이하 worker)의 Selector 에 등록한다.

3. worker는 Selector.open() 을 통해 selector를 열고 다음의 과정을 반복한다.

3-1. select() 메소드를 호출하여 준비된 채널 수를 확인한다.

3-2. 준비된 채널이 있다면 processSelectedKeys() 를 수행한다.

3-3. selectedKeys(Set)의 값을 대상으로 다음을 반복한다.

3-3-1. 선택한 키에서 채널 및 관련 정보를 얻는다.

3-3-2. I/O 이벤트가 들어왔는지 확인하고 있을 경우 처리한다.

3-3-3. 필요에 따라 리스닝 이벤트를 변경한다.

3-3-4. 처리된 경우 해당 키를 Set 에서 제거한다.
```

2줄 요약하면
1. Worker EventLoop 는 CPU * 2 개가 있다. 이는 변하지 않는다.
2. 하나의 EventLoop에 들어간다면, 해당 Channel 은 다른 EventLoop에 들어가지 않는다.

이렇게 됩니다.

저희가 사용하는 Spring MVC 는 내장 tomcat 을 사용하고, virtual thread 를 지원하는 Spring Boot 3.2.0은 Tomcat 10.1.16 을 사용합니다.

내부 코드를 보면 아래와 같이 설정합니다.
```java
    @Override
    protected void startInternal() throws LifecycleException {
        executor = new VirtualThreadExecutor(getNamePrefix());
        setState(LifecycleState.STARTING);
    }
```

```java
public class VirtualThreadExecutor extends AbstractExecutorService {
    // ...

    public VirtualThreadExecutor(String namePrefix) {
        this.threadBuilder = this.jreCompat.createVirtualThreadBuilder(namePrefix);
    }

    public void execute(Runnable command) {
        if (this.isShutdown()) {
            throw new RejectedExecutionException(sm.getString("virtualThreadExecutor.taskRejected", new Object[]{command.toString(), this.toString()}));
        } else {
            this.jreCompat.threadBuilderStart(this.threadBuilder, command);
        }
    }
    // ...
}
```

```java
public class Jre21Compat extends Jre19Compat {
    // ...

    static boolean isSupported() {
        return supported;
    }

    public Object createVirtualThreadBuilder(String name) {
        try {
            Object threadBuilder = ofVirtualMethod.invoke((Object)null, (Object[])null);
            nameMethod.invoke(threadBuilder, name, 0L);
            return threadBuilder;
        } catch (IllegalArgumentException | InvocationTargetException | IllegalAccessException e) {
            throw new UnsupportedOperationException(e);
        }
    }
    // ...
}
```

리플렉션을 활용해서 `Thread.ofVirtual().name(name, 0);` 의 코드를 실행하게 됩니다. 결국, Tomcat 도 다른 별다른 추가설정 없이 Virtual Thread 를 그대로 활용하고 있습니다.

결론을 내자면, OpenJDK 에서는 Virtual Thread 는 Java 를 사용하는 환경, network-bounded 환경에서 Web-flux 를 완전히 몰아낼 수 있는 실버불릿인것 처럼 소개하고 있습니다.

maxPoolSize 를 통해 전체 platform thread의 수를 컨트롤 할 수 있는 것처럼 이야기를 하지만, 실상은 그렇지 않습니다. 저희 부서와 같이 I/O 작업에 parallism 을 꽤 많이 사용할 경우에는 platform thread 의 수가 256 까지 늘어날 위험성이 충분히 있습니다. 이 경우 기존의 thread pool executor 를 사용하는 경우보단 context-change 의 비용이 어느정도 줄 수 는 있겠지만, 상황에 따라 platform thread 의 수를 조정하는 룰도 조정하지 못해 오히려 더 큰 overhead 를 줄 수 있다고 생각됩니다.

추가로, tomcat 의 virtual thread의 스케쥴러 와 저희가 bean 으로 설정하는 virtual thread pool 의 스케쥴러, 더 나아가 jdk 25 에 나올 gatherer의 병렬처리에도 virtual thread를 사용하는데 그 스케쥴러도 모두 동일한 하나의 fork join pool 을 사용합니다. 이 때문에 예상치 못한 platform thread 의 증가도 가능하다고 생각됩니다.