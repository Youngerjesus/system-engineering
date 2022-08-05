# dirty page 가 I/O 에 끼치는 영향

메모리 정보를 보는 방법 중에는 ``/proc/meminfo`` 파일을 통해서 볼 수도 있는데 여기에는 Dirty 라고 표현된 부분이 있다. 이 영역은 dirty page 라고 불리는 커널의 메모리 영역 중 일부를 의미한다.

여기서는 dirty page 가 뭔지, 어떻게 생성되는지 그리고 이 dirty page 가 I/O 에 미치는 영향은 무엇인지에 대해서 알아보겠다.

## dirty page 란

리눅스에서는 파일 I/O 가 일어날 때 커널은 Page Cache 를 이용해서 디스크에 있는 파일의 내용을 메모리에 잠시 저장하고 필요할 때마다 이 Page Cache 를 이용한다.

이를 통해서 디스크에 매번 접근하는게 아니라 더 빠른 메모리의 접근 속도를 활용할 수 있고 전체적으로 시스템의 성능을 향상 시킬 수 있다.

커널은 Page Cache 에 디스크 내용 중 일부를 저장한다. 그럼 만약에 읽기 작업이 아니라 쓰기 작업이 일어나면 어떻게 될까?

Page Cache 의 내용과 디스크에 있는 내용이 서로 달라지게 될 수 있다. 그래서 커널은 해당 메모리 영역에 대해서 디스크 영역과 달라졌다는 표시를 하기 위해서 Dirty 비트를 켠다. 그리고 이 Dirty 비트가 켜져있는 페이지를 dirty page 라고 부른다.

즉 dirty page 는 page cache 에 있는 내용이 쓰기 작업으로 인해서 변경된 내용을 말한다.

쓰기 작업이 일어날 때 바로 디스크에 반영하지 않고 Page cache 에서만 반영을 한다. 그렇기 때문에 현재 상태에서 전원이 나가게 되면 쓰기 작업은 반영되지 않을 수 있다. 하지만 그렇다고 해서 매번 디스크에 쓰기를 하면 상당량의 I/O 때문에 성능 저하가 발생할 수도 있다.

그래서 커널은 몇 가지 조건을 만들어서 해당 조건을 만족시키면 dirty page 를 디스크에 동기화 시킨다. 이 기능을 `page writeback` 이라고 부른다. (dirty page 동기화라고도 부른다.)

커널 버전에 따라서 다르겠지만 flush 라는 단어가 들어간 커널 스레드 `(pdflush, flush, bdflush)` 가 이 작업을 진행한다.

I/O 가 많이 발생하는 서버에서는 dirty page 를 언제 동기화 시키는지가 중요한 성능 튜닝의 요소가 될 수 있다. 그리고 커널에서는 서버의 워크로드에 가장 적합한 동기화 전략을 구사할 수 있도록 파라미터로 조절할 수 있는 인터페이스를 제공해준다. 이를 하나씩 살펴보자.

## dirty page 관련 커널 파라미터

dirty 와 관련된 커널 파라미터는 총 6 개가 있다. 이는 `sysctl -a | grep -i dirty` 명령을 통해서 볼 수 있다. (sysctl 이 커널 매개변수를 보는 명령어다.)

```bash
vm.dirty_background_ratio = 10
vm.dirty_background_bytes = 0
vm.dirty_ratio = 20
vm.dirty_bytes = 0
vm.dirty_writeback_centisecs = 500
vm.dirty_expire_centisecs = 3000
```

먼저 살펴볼 파라미터는 `vm.dirty_background_ratio` 이다. dirty_page 의 내용을 백그라운드로 동기화 할 때 그 기준이 되는 비율을 의미한다. 

- 전체 메모리 양에 해당 파라미터에 설정되어 있는 비율을 곱해서 나온 기준 값보다 dirty page 가 커지면 그때 dirty page 의 동기화를 시작한다. 예로 이 값이 10 이고 16GB 메모리를 쓰고 있다면 dirty page 가 1.6GB 보다 커지는 순간에 동기화를 시작한다.

두 번째는 `vm.dirty_background_bytes` 이다. ratio 는 전체 메모리 대비 비율을 지정한다면 bytes 는 절대적인 bytes 값을 지정한다.

- 만약 이 값이 65535 라면 dirty page 가 65535 bytes 가 되었을 때 동기화를 시작한다.

세 번째는 `vm.dirty_ratio 이다.` `vm.background_ratio` 와 비슷하게 전체 메모리를 기준으로 dirty page 비율을 산정한다. 다만 차이점이 있는데 예로 바로보자. 

- 이 값이 10 이고 전체 메모리가 16GB 라면 프로세스가 I/O 작업을 하던 중 dirty page 가 1.6 GB 가 되는 순간 해당 프로세스의 I/O 작업을 모두 멈추고 dirty page 동기화를 시작하게 된다. 즉 `hard limit` 이라고 보면 된다.

네 번째는 `vm.dirty_bytes` 이다. `vm.dirty_ratio 와 같은 방식을 이용하지만 절대적인 bytes 를 기준으로 동작한다.

다섯 번째는 `vm.dirty_writeback_centisecs` 이다. 이 값은 flush 커널 스레드를 몇 초 간격으로 꺠울 것인지를 결정한다. 

- 단위는 1/100 (초) 이므로 5 초로 설정하고 싶다면 500 값을 주면 된다. 500 을 주면 5 초에 한 번 flush 커널 스레드가 꺠어나서 동기화를 시작한다.

마지막은 `vm.dirty_expire_centisecs` 이다. 이 값도 flush 커널 스레드의 동작에 영향을 미친다. `vm.dirty_writeback_centisecs` 로 인해서 깨어날 flush 스레드가 동기화 할 페이지를 찾을 때 이 값을 사용하는데 이 값도 단위는 1/100 (초) 이다. 

- 즉 3000 으로 설정되어 있다면 30 초동안 디스크로 동기화 되지 않은 페이지들을 디스크에 동기화 시킨다.

## 백그라운드 동기화

dirty page 동기화는 백그라운드 동기화와 주기적인 동기화 그리고 명시적인 동기화 세 가지로 구분이 가능하다.

백그라운드 동기화는 `vm.dirty_background_ratio` 와 `vm.dirty_ratio` 를 통해서 조절이 가능하다.

사실 `vm.dirty_ratio` 를 넘어서 발생하는 동기화는 백그라운드 동기화는 아니지만 (그냥 동기화 임.) 백그라운드에서 감시하고 있다가 동기화를 하니까 백그라운드 동기화라고 표현했다.

두 번째로 주기적인 동기화는 동기화 작업이 주기적으로 진행되는 것을 말한다. 커널 파라미터에서 `vm.dirty_writeback_centisec`, `vm.dirty_expire_centisecs` 를 통해서 조절이 가능하다.

마지막으로 명시적인 동기화는 명령어를 통해 명시적으로 동기화가 되는 걸 말한다. sync, fsync 등의 명령어를 이용하면 현재 생성되어 있는 dirty page 를 명시적으로 디스크에 쓴다.

여기서는 백그라운드 동기화와 주기적인 동기화를 살펴볼 것. 

커널의 동작을 추적하기 위해서는 ftrace 를 이용하면 된다. (ftrace 는 커널 동작을 추적하기 위해서 사용한다.)

ftrace 설정하기 

- ``mount -t debugfs debugfs /sys/kernel/debug``
- ``echo function > ./current_tracer``

설정 후에 cat 명령을 통해서 커널 함수가 잘 찍히는지 확인한다. 

trace_pipe 파일을 통해서 커널 함수 호출 확인하기 

```
$ cat -v ./trace_pipe
```

dirty page 의 변화를 보고 싶다면 ``/proc/meminfo | grep -i dirty`` 를 스크립트로 만들어서 보자.

커널 파라미터 값 조절하기 (주기적인 동기화를 보고 싶지 않다면.)

```
$ sysctl -w vm.dirty_writeback_centisecs=0
$ sysctl -w v.dirty_background_ratio=1
```

여기서는 테스트 할 때 C 로 파일을 만들어서 작업을 하네. (초당 1MB 의 쓰기 작업을 함으로써.)


## 요약

ftrace 를 통해서 실제 내부에서 호출되는 함수를 분석해보면 다음과 같은 결로론을 얻을 수 있다.

- vm.dirty_ratio 의 최소값은 5 이다. 5 보다 작은 값으로 설정해도 강제로 5 로 재설정된다.
- vm.dirty_background_ratio 가 vm.dirty_ratio 보다 크다면 강제로 vm.dirty_ratio 의 절반 값으로 수정된다.
- vm.dirty_background_bytes, vm.dirty_bytes 가 설정되어 있다면 각각 vm.dirty_background_ratio, vm.dirty_ratio 가 무시된다.
- vm.dirty_writeback_centisecs 가 0 이면 주기적인 동기화를 실행시키지 않는다.
- vm.dirty_ratio 에 설정한 값 이상으로 dirty_page 가 생성되면 성능 저하가 발생한다.

그리고 dirty page 를 너무 빨리 동기화 시키도록 하면 flush 커널 스레드가 너무 자주 깨어나게 되고 dirty page 를 너무 늦게 동기화 시키면 동기화 해야 할 dirty page 가 너무 많아져서 vm.dirty_ratio 에 도달할 가능성이 커진다. 이러면 성능 저하가 일어날 수 있기 때문에 워크로드와 시스템 구성에 맞게 dirty page 동기화 수준을 설정해줘야 한다.
