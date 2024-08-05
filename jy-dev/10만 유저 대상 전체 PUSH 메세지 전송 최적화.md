# **10만 유저 대상 전체 PUSH 메시지 전송 최적화**

Legacy 시스템의 PUSH 메시지 전송 로직을 Spring Batch와 FCM을 이용하여 어떻게 개선했는지에 대한 경험을 공유합니다.

# **기존 Legacy 시스템**

기존의 Legacy 시스템은 Firebase Cloud Messaging(FCM)의 HTTP v1 API를 사용하여 RestTemplate을 통해 PUSH 메시지를 동기적으로 전송하고 있어 대략 10만 유저대상으로 요청을 보내는데 2시간이 넘게 걸리는 이슈가 있었습니다.

## **문제점**

1. **응답 지연**: 동기적 전송 로직으로 인해 HTTP 요청이 완료될 때까지 다른 요청을 처리할 수 없어 전체 시스템의 응답성이 저하되었습니다. (대략적으로 2시간 이상)
2. **처리 비효율성**: Spring Batch에서 Processor에서 메시지 전송을 처리하고 있었기 때문에, 각 메시지가 개별적으로 처리되어 일괄 처리의 이점을 활용할 수 없었습니다.
3. **기술적 제한**: FCM의 HTTP v1 API는 유연성이 떨어지고 Deprecated된 API였으므로, 유지 보수와 확장성에 문제가 있었습니다.

# **최적화 방법**

1. **동기 → 비동기 작업 전환**: 메시지 전송을 비동기적으로 처리함으로써 응답 지연을 줄이고 시스템 전체의 처리량을 증가시킬 수 있습니다.
2. **Writer 컴포넌트를 통한 처리**: 메시지 처리 로직을 Processor에서 Writer로 이동시켜 일괄 처리를 가능하게 하여 전송 효율을 높일 수 있습니다.

## **비동기 작업으로 전환 시 발생할 수 있는 문제점**

1. **작업 완료 관리**: 비동기 작업은 작업의 완료를 별도로 관리해야 합니다. 작업이 완료되지 않은 상태에서 시스템이 종료될 위험이 있습니다.
2. **오류 처리 복잡성 증가**: 비동기 작업은 동기 작업에 비해 오류를 감지하고 대응하는 복잡성이 증가합니다.
3. **리소스 관리**: 비동기 작업은 작업을 기다리지 않고 계속 요청을 생성하기 때문에 Thread 생성을 조절하지 않으면 전체 PUSH 메세지 같은 경우는 OOM이 발생할 수 있습니다.

# 비동기 처리 방법

## **Firebase SDK를 이용한 비동기 요청**

Firebase SDK는 이미 비동기 처리를 위한 메서드를 제공하며, 이를 사용하면 비동기 작업을 간단하게 구현할 수 있습니다. 그러나 이 방식에는 몇 가지 고려해야 할 점이 있습니다:

### **장점**

- **간편한 구현**: Firebase SDK가 제공하는 비동기 메서드를 사용하면, 복잡한 비동기 로직을 쉽게 구현할 수 있습니다.
- **자동 리소스 관리**: SDK 내부에서 스레드와 리소스를 관리하므로, 직접적인 스레드 관리가 필요하지 않습니다.

### **단점**

- **제한된 제어**: 비동기 작업 관련 로직이 Firebase SDK 내부에 캡슐화되어 있어, 세부적인 동작을 제어하거나 커스텀하기 어렵습니다.
- **메모리 오버헤드**: 비동기 작업을 제어하기 위해 큐에 저장하고 관리하는 과정에서 추가적인 메모리 사용이 발생할 수 있습니다.
- ThreadExecutor를 통해 직접 FCM HTTP API를 비동기적으로 관리
    - 유연하게 작성하기 어렵고 직접 비동기 관련한 메서드를 만들어야하지만 직접 컨트롤하기가 쉬움

## **ThreadExecutor를 통해 직접 FCM HTTP API 비동기 처리**

ThreadExecutor를 사용하여 FCM HTTP API를 비동기적으로 처리하는 방법은 직접적인 제어와 커스터마이징이 가능하다는 장점이 있습니다. 그러나 이 방법은 구현이 더 복잡할 수 있습니다.

### **장점**

- **세밀한 제어**: 스레드와 작업의 세부 동작을 직접 제어할 수 있어, 특정 요구 사항에 맞게 커스터마이징이 가능합니다.
- **효율적인 리소스 관리**: 스레드 풀을 사용하여 스레드 생성 및 관리를 효율적으로 수행할 수 있습니다.

### **단점**

- **복잡한 구현**: 비동기 처리를 직접 구현해야 하므로, 코드가 더 복잡해질 수 있습니다.
- **추가적인 관리 필요**: 스레드 풀 및 비동기 작업의 예외 처리를 포함한 리소스 관리를 직접 처리해야 합니다.

# 어떤 방법으로 접근 했냐?

제한된 시간 내에 대량의 푸시 메시지를 효율적으로 전송하기 위해 Firebase SDK를 사용하여 구현하기로 했습니다. Firebase SDK는 비동기 전송 관련 메서드를 지원하고 있어, 이를 활용하면 안전하고 빠르게 처리할 수 있었습니다. 아래는 진행했던 과정입니다.

## **FirebaseMessaging의 `sendAsync` 활용**

Firebase SDK의 **`sendAsync`** 메서드를 사용하여 FCM 요청을 비동기로 처리했습니다.

```java
 @StepScope
 public ItemWriter<PushItem> pushWriter() {
 
		 return items -> { 
				 
				 var messages = messageConverter.convert(items);
				 
				 messages.forEach(FirebaseMessaging.getInstance()::sendAsync);
     };
 }
```

### 고려할 점

- Firebase SDK의 인스턴스는 기본적으로 요청에 대해 항상 새로운 스레드를 생성합니다.
- Spring Batch는 비동기로 요청한 FCM 요청을 기다려주지 않기 때문에, 요청이 완료되었는지 체크하고 Job 종료를 관리해야 합니다.

### 비동기 요청 관리

모든 비동기 요청의 완료 여부를 파악하기 위해 Firebase SDK의 **`sendAsync`** 메서드가 반환하는 **`ApiFuture<?>`** 객체를 활용했습니다. 이 객체는 **`Future<?>`**를 상속하고 있어 비동기 요청을 관리할 수 있습니다.

```java
 private Queue<Future<?>> queue = new LinkedList<>();
 
 @StepScope
 public ItemWriter<PushItem> pushWriter() {
 
		 return items -> { 
				 
				 var messages = messageConverter.convert(items);
				 
				 var list = messages.map(FirebaseMessaging.getInstance()::sendAsync);
				 queue.addAll(list);
     };
 }
```

완료된 비동기 요청은 제거해야 하기 때문에 **`Queue`** 자료구조를 활용하였습니다.

### OOM 관리

Firebase SDK에서 FCM 요청을 비동기적으로 수행할 때 요청마다 새로운 스레드를 생성하기 때문에, 새로운 Chunk가 계속 쌓이게 되면 OOM(Out Of Memory) 문제가 발생할 수 있습니다. 이를 방지하기 위해 스레드 수를 조절했습니다.

```java
 private Queue<Future<?>> queue = new LinkedList<>();
 private int limit = 500;
 
 @StepScope
 public ItemWriter<PushItem> pushWriter() {
		 return items -> { 
				 
				 var messages = messageConverter.convert(items);
				 
				 var list = messages.map(FirebaseMessaging.getInstance()::sendAsync);
				 queue.addAll(list);
				 
     };
 }
 
 public void waitJobIfOverLimit() {
 
		 while(queue.size() > limit) {
		 
				 var future = queue.poll();
				 
				 if(!future.isDone()) {
            queue.add(future);
         }
		 } 
 }
```

**`limit`**을 설정해 작업 수가 **`limit`**을 넘어서면 Job을 중지하고 작업 수가 **`limit`**보다 작아질 때까지 기다리면서 스레드 수를 조절합니다.

### JobExecutionListener를 이용한 모든 FCM 요청 대기

JobExecutionListener를 사용하여 Job이 종료되기 전에 모든 FCM 요청이 완료되도록 대기할 수 있습니다.

```java
public class PushAsyncWaitListener implements JobExecutionListener {

    private final Queue<Future<?>> queue = new LinkedList<>();
    private final int limit;

    public PushAsyncWaitListener(int limit) {
        this.limit = limit;
    }

    public void addFutures(List<? extends Future<?>> futures) {
        queue.add(futures);
    }
    
    public void waitJobIfOverLimit() {
    
			 while(queue.size() > limit) {
		 
					var future = queue.poll();
				 
					if(!future.isDone()) {
             queue.add(future);
          }
			 } 
	 }

    @Override
    public void afterJob(JobExecution jobExecution) {
    
        if (jobExecution.getStatus() != BatchStatus.COMPLETED) {
            return;
        }

        while(!queue.isEmpty()) {
		 
					 var future = queue.poll();
				 
					 if(!future.isDone()) {
              queue.add(future);
           }
			  } 
    }
}

```

```java
private PushAsyncWaitListener listener;

@StepScope
public ItemWriter<PushItem> pushWriter() {
		return items -> { 
				 
				var messages = messageConverter.convert(items);
				 
				var list = messages.map(FirebaseMessaging.getInstance()::sendAsync);
				listener.addFutures(list);
				listener.waitJobIfOverLimit();
    };
}
```

Job이 종료될 때 모든 비동기 요청을 기다릴 수 있게 PushAsyncWaitListener로 Future를 관리하는 로직을 이동시키고 afterJob에서 Job이 종료되기 전에 모든 비동기 요청이 끝날때까지 기다립니다.

### **결과**

결과적으로, 기존에 2시간 이상 걸리던 작업을 스레드 수를 조절하면서 16분으로 단축시킬 수 있었습니다. 하지만 매번 스레드를 생성하는 방식은 비효율적이었고, 더 나은 방법이 있을 것으로 생각됩니다.

## ThreadPoolTaskExecutor 이용하기

기존 방식과 다르게 스레드를 재활용할 수 있는 방법으로 **`ThreadPoolTaskExecutor`**를 이용할 수 있습니다. 이는 새로운 스레드를 매번 생성하는 대신, 스레드를 재사용하여 자원을 효율적으로 관리할 수 있습니다. 다음은 **`ThreadPoolTaskExecutor`**를 이용하여 비동기 작업을 직접 만들어 관리하는 과정입니다.

### **비동기 작업 관리**

**`ThreadPoolTaskExecutor`**를 사용하여 비동기 작업을 관리하고, 완료 여부를 추적합니다.

```java
public class PushAsyncWaitListener implements JobExecutionListener {

    private final Integer chunkSize;
    private final ThreadPoolTaskExecutor executor;

    public PushAsyncWaitListener(ThreadPoolTaskExecutor threadPoolTaskExecutor, Integer chunkSize) {
        this.chunkSize = chunkSize;
        this.executor = threadPoolTaskExecutor;
    }

    public void waitJobIfMemoryLimit() {
        int remainCnt = getWaitQueueRemainingCapacity();

        while (remainCnt < chunkSize) {
            remainCnt = getWaitQueueRemainingCapacity();
        }
    }

    @Override
    public void afterJob(JobExecution jobExecution) {

        if (jobExecution.getStatus() != BatchStatus.COMPLETED) {
            return;
        }

        waitAllPushSend();
    }

    private void waitAllPushSend() {
        log.info("Waiting for all push send to complete...");
        while (executor.getActiveCount() > 0 || executor.getQueueSize() > 0) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        executor.shutdown();
    }

    private int getWaitQueueRemainingCapacity() {
        return executor.getQueueCapacity() - executor.getQueueSize();
    }
}
```

```

private final PushAsyncWaitListener listener;
private final ThreadPoolTaskExecutor executor;

@StepScope
public ItemWriter<PushItem> pushWriter() {
		return items -> { 
		
				var messages = messageConverter.convert(items);
				 
				var list = messages.map(message -> executor.submit(() -> {
					 FirebaseMessaging.getInstance().send(message);
				}).toList();
				
				listener.waitJobIfMemoryLimit();
    };
}

```

### **ThreadPoolTaskExecutor 설정**

가장 중요한 건 적절한 기본 스레드 수와 큐의 용량을 정하는 것 입니다. ThreadPoolTaskExecutor은 아래와 같이 설정할 수 있습니다.

```java
java코드 복사
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
public class TaskExecutorConfig {

    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(100); // 기본 스레드 수
        executor.setMaxPoolSize(500);  // 최대 스레드 수
        executor.setQueueCapacity(10000); // 큐 용량
        executor.setThreadNamePrefix("PushExecutor-"); // 스레드 이름 접두사
        executor.setKeepAliveSeconds(60); // 유휴 스레드 유지 시간
        executor.initialize();
        return executor;
    }
}

```

여러 가지 세팅으로 10만 유저를 기준으로 테스트한 결과는 아래와 같습니다.

Core Thread / Max Thread / Queue Capacity

- **`3000/3000/5000`** → 6분 16초
- **`500/3000/5000`** → 2분 48초
- **`200/500/10000`** → 2분 28초
- **`170/500/10000`** → 2분 1초
- **`160/500/10000`** → 1분 58초
- **`150/500/10000`** → 2분 9초
- **`100/500/10000`** → 2분 14초
- **`80/500/10000`** → 2분 39초
- **`50/500/10000`** → 4분 3초

너무 많은 스레드를 사용하면 오히려 오래 걸렸고, 너무 적어도 오래 걸렸습니다. 따라서 세팅 값은 실제로 테스트를 통해 최적화해야 합니다.

### **결론**

여러 과정을 거치면서 로직의 동작 시간을 2시간에서 16분으로, 다시 2분대로 줄일 수 있었습니다. 이는 **`ThreadPoolTaskExecutor`**의 적절한 설정과 비동기 작업의 효율적인 관리 덕분이었습니다. 최적의 성능을 위해서는 실제 환경에서 다양한 설정을 테스트하며 최적화하는 것이 중요합니다.