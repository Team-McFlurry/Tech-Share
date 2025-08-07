k8s 의 network layer 인 cilium, tcpdump 등 우리가 평소에 사용하는 수많은 기술들을 포함해서 모 통신사를 공격하는데 사용한 BPF door 또한 이름에서 알 수 있다시피 bpf 를 사용하는 등, 우리 생각보다 많은 곳에서 BPF 를 활용하고 있다. 이런 BPF이 무엇이고, 어떤 환경에서 탄생했는지, 그리고 활용 범위를 알아보자.

### Classic BPF

`Berkeley packet filter`

네트워크 패킷을 트리거로 활용하여 운영체제 커널 수준에서 효율적으로 캡쳐하고, 필터링할 수 있게 해주는 기술로, 아래의 특징을 갖는다.

1. 성능적으로 효율적
2. 시스템 안정성
3. 실시간성



BPF 를 사용하지 않으면서 네트워크 트래픽을 분석하려면 user-context 에서 application 을 작성해야 한다. 그럴 경우 아래의 flow를 탄다.

>1. 패킷 도착 -> kernel interrupt
>2. kernel 에서 user-context 로 패킷을 복사
>3. user-context 에서 패킷 필터링
>4. kernel-context 로 복귀

 이럴 경우, context-switch 로 성능상 불이익이 있을수밖에 없다. context-switch 하지 않는 케이스를 생각하자면 아래와 같다.

>1. 패킷 도착 -> kernel interrupt
>2. 작성한 필터링 규칙을 kernel 에서 실행

사용자가 작성한 필터링 규칙을 kernel 에 직접 들어갈 경우 kernel 의 안정성을 보장할 수 없다. 안정성을 보장하기 위해 도입한 기능이 syscall 이지만, 이를 활용하여 안정성을 도모하려는 경우 다시 user-context 로 context-switch 가 발생하는 overhead 가 생기게 된다.

BPF 는 커널 내 작은 가상머신과 전용 Instruction set 을 갖추고 있어 안정성을 도모하였다. BPF 프로그램을 작성하여 컴파일 하면 BPF 가상머신 위에 돌아갈 수 있는 바이트 코드로 변환되어 동작한다. BPF 를 실제로 사용하는 tcpdump 의 -d flag 를 통해 BPF 필터링 규칙을 컴파일해서 확인하면 아래와 같은 바이트코드를 확인할 수 있다.

```
~/test/bpf$ sudo tcpdump -i any -d 'tcp port 80'

tcpdump: data link type LINUX_SLL2
(000) ldh      [0]
(001) jeq      #0x86dd          jt 2	jf 8
(002) ldb      [26]
(003) jeq      #0x6             jt 4	jf 19
(004) ldh      [60]
(005) jeq      #0x50            jt 18	jf 6
(006) ldh      [62]
(007) jeq      #0x50            jt 18	jf 19
(008) jeq      #0x800           jt 9	jf 19
(009) ldb      [29]
(010) jeq      #0x6             jt 11	jf 19
(011) ldh      [26]
(012) jset     #0x1fff          jt 19	jf 13
(013) ldxb     4*([20]&0xf)
(014) ldh      [x + 20]
(015) jeq      #0x50            jt 18	jf 16
(016) ldh      [x + 22]
(017) jeq      #0x50            jt 18	jf 19
(018) ret      #262144
(019) ret      #0
```

이를 통해, 모든 패킷을 kernel 에서 user-context 로 복사하는 것이 아닌, 필터링 규칙에 맞는 패킷들만 user-context 에 복사하여 활용할 수 있게 되었다.

지금까지 이야기한 BPF 는 classic BPF 라고 하고, 줄여서 cBPF 라고 부른다. cBPF 는 32bit 레지스터 2개만 가지고 있고, 함수를 정의, 호출할 수 없었으며 네트워크 패킷만을 트리거로 활용할 수 있었다.

### eBPF

extended BPF, eBPF 는 이러한 한계를 극복하였다. 64bit 레지스터 11개를 가지고, 함수를 정의하고 호출할 수 있으며, 다른 프로세스/스레드와 데이터를 공유할 수 있는 BPF Map 을 정의하여 활용할 수 있다. 또한, 트리거를 네트워크 패킷뿐만 아니라 다른 이벤트들도 활용할 수 있게 되었다.

아래는 bcc 를 활용한 eBPF 코드이다. process 마다 파일 읽기 카운트를 저장하고, 출력하는 간단한 코드이다.

```python
#!/usr/bin/python
from bcc import BPF

prog = """
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>

BPF_HASH(counts, u32, u32);

int kprobe____x64_sys_read(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid();
    u32 *count = counts.lookup(&pid);

    if (count) {
        (*count)++;
    } else {
        u32 new_count = 1;
        counts.update(&pid, &new_count);
    }

    bpf_trace_printk("PID %d created new process (count: %d)\\n", pid,
                     count ? *count : 1);
    return 0;
}
"""

b = BPF(text=prog)
try:
    b.trace_print()
except KeyboardInterrupt:
    counts = b.get_table("counts")
    for k, v in counts.items():
        print(f"PID {k.value}: {v.value}회")
```



앞서 말했듯, eBPF 는 이벤트로 트리깅 된다.

eBPF의 이벤트 종류로는 다음과 같은 종류가 있다.

1. 동적 포인트
2. 정적 포인트
3. HW/SW 이벤트

동적 - 특정 영역의 함수 시작/끝에 함수를 삽입할 수 있다는 점에서 동적

정적 - 커널 개발자들이 관리한다는 의미로 정적

* 동적 포인트

  * kprobe - kernel 레벨에서 동작
    * kprobe : 커널 함수의 실행 시점에 원하는 프로그램 동적 추가
    * kretprobe: 커널 함수의 종료 시점에 원하는 프로그램 동적 추가

  * uprobe - 사용자 레벨에서 동작

* 정적 포인트

  * 트레이스포인트
    * `/sys/kernel/debug/tracing/events/`
    * io / cpu event (amd 의 경우 amd_cpu)  등의 포인트를 제공
  * USDT (User level statically defiend tracing)
    * tracepoint 의 사용자 영역 버전
    * BCC tools 의 tplist 를 통해 USDT list 를 가져올 수 있다.
    * `tplist-bpfcc -l /lib64/ld-linux-x86-64.so.2`

* HW/SW 이벤트

  * `perf list`
    * SW event
      * Cpu-clock / major-faults / minor-faults / page-faults
    * HW event
      * Cache-misses / cache-refrences / instructions

#### BPF Map

위 코드에서 나왔던 BPF_HASH 등을 통해 메모리를 할당하는 동작은 BPF 맵을 이용하는 것이다. BPF 맵은 커널 공간의 heap 메모리를 활용하고, cgroup 기반으로 관리되며, 실행시킨 eBPF 프로그램이 종료되면 해제된다.

사실 eBPF 프로그램이 종료되면 해제되는 것이 아닌, BPF Map 에 대한 reference count 가 0이 되면 해제된다.

위에서 이야기했던 다른 프로그램과의 공유도 이를 활용한 동작이다. 리눅스는 모든것이 파일이니, file descriptor 도 BPF Map 에 대한 refence 를 유지할 수 있다. eBPF 에서 BPF Map 을 공유상태로 바꾸기 위해선 pinning 이라는 작업을 하는데, 이 작업이 바로 file 을 생성하고, fd가 BPF Map 에 대한 reference count 를 증가시키는 동작을 한다.

이후, 다른 eBPF 나 프로세스 에서는 `bpf_obj_get` 라는 BPF syscall 을 이용해 이 map 에 접근할 수 있는데, file 에 관련 메모리 정보가 있는 것이 아니라 해당 파일의 `inode->i_private` 에서 BPF map 포인터를 가져와 활용한다.([코드](https://github.com/torvalds/linux/blob/master/kernel/bpf/inode.c))



### Cilium

cilium 은 XDP + eBPF 조합을 통해 k8s 의 Container Network Interface 를 제공한다.

XDP 는 리눅스 커널의 네트워크 패킷 처리 프레임워크로, 패킷이 들어오는 순서로 보면 아래의 순서에 위치한다.

```
NIC -> Network driver -> XDP hook -> kernel network stack
```

이런 XDP를 트리거로 활용하기에, kernel network stack 에 패킷이 들어가기 전에 네트워크 패킷을 조작할 수 있다.

XDP 를 활용하여 eBPF 코드를 작성할때는 XDP action 을 반환하므로서 패킷을 전달할 방식을 결정할 수 있다.

```
XDP_ABORTED
XDP_DROP
XDP_PASS
XDP_TX // 같은 NIC 로 패킷 재전송
XDP_REDIRECT
```

이를 통해 아래와 같이 eBPF 프로그램을 작성할 수 있다.

참고로, 위에서 부터 계속 사용하는 BCC(BPF Compiler Collection) 이라는 툴이 `SEC("xdp")`  매크로를 처리해준다.

```python
#!/usr/bin/python3
from bcc import BPF

prog = """
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>

BPF_HASH(blocked_ips, u32, u8);

SEC("xdp")
int block_ips(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    struct iphdr *ip = (void *)(eth + 1);
    u8 *blocked = blocked_ips.lookup(&ip->saddr);
    if (blocked) {
        return XDP_DROP;
    }
    return XDP_PASS;
}
"""

try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
		print("exit")
```










