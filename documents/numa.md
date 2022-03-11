## NUMA, 메모리 관리의 새로운 세계

여기서는 NUMA 아키텍처가 뭐고, NUMA 아키텍처가 메모리 할당에 어떤 영향을 주는지 알아보겠다.****

### ****NUMA 아키텍처****

NUMA 는 Non-Uniform Memory Access 의 약어다. 번역하자면 불균형 메모리 접근이라는 뜻이며 멀티 프로세서 환경에서 적용되는 메모리 접근 방식을 말한다.

NUMA 아키텍처의 반대는 UMA (Uniform Memory Access) 로 모든 프로세서가 공용 BUS 를 통해서 메모리에 접근하는 방식이다.

UMA 아키텍처의 문제점은 버스를 동시에 이용하지 못하는 단점이 있다. 예로 0 번 소켓에 있는 CPU 가 메모리에 접근하는 동안은 1 번 소켓에 있는 CPU 는 메모리에 접근하지 못한다. 즉 싱글 CPU 에 적합한 구조. (TMI: CPU 안에 코어가 있다.)

NUMA 는 CPU 마다 로컬 메모리가 서로 붙어 있는 관계로 이뤄져있다. 그래서 한 CPU 가 자신의 로컬 메모리에 접근하는 동안 다른 CPU 도 자신에게 붙어 있는 로컬 메모리에 접근이 가능하다.

주의할 점은 Remote 에 있는 메모리에 접근할 땐 성능이 떨어질 수 있어서 로컬 메모리를 최대한으로 쓰도록 하는게 중요하다.****

### ****리눅스에서 NUMA 확인****

```python
ubuntu@ip-192-168-111-182:~$ numactl --show
policy: default
preferred node: current
physcpubind: 0 1
cpubind: 0
nodebind: 0
```

- 기본 정책은 default 이고 현재 사용중인 프로세스가 포함된 노드 (= CPU + 로컬 메모리) 에서 메모리를 먼저 가져다가 쓰는 방식이다. 이 정책은 별도의 설정을 하지 않는한 모든 프로세스가 해당된다.
- 두 번째는 bind 정책으로 특정 프로세스를 특정 노드에 바인딩시키는 방식을 취한다. 특정 노드에 할당시켜서 거기에 있는 메모리만 사용하도록 하는 방식이다. 이 경우 메모리의 지역성이 좋아져서 더 빠를수 있지만 해당 로컬 메모리에서 가용 메모리가 부족해진다면 성능이 급격히 나빠질 수 있다.
- 세 번째는 preferred 정책으로 bind 와 비슷하지만 선호하는 노드를 설정한다.
- 마지막은 interleaved 정책으로 다수의 노드에서 거의 동일한 비율로 메모리를 받는 방식이다. Round-Robin 정책에 따라 돌아가면서 할당 받는다.

다음은 `-H` 옵션이다. 이건 노드에 있는 메모리 정보를 확인할 때 사용할 수 있다. 

```python
ubuntu@ip-192-168-111-182:~$ numactl -H
available: 1 nodes (0) # (1) 
node 0 cpus: 0 1       # (2) 
node 0 size: 3866 MB
node 0 free: 1759 MB
node distances:        # (3) 
node   0
  0:  10
```

- (1) 은 NUMA 노드가 1 개로 구성되어 있음을 알 수 있다.
- (2) 은 노드 0 번에 cpu 0 과 1 이 붙어 있음을 알 수 있다. 각 노드에 할당된 메모리 크기와 사용 가능한 메모리 크기도 볼 수 있다.
- (3) 은 메모리에 접근하는데 걸리는 시간이다. 로컬 메모리에 접근할 때 걸리는 시간을 10 이라고 한다면 리모트 메모리에 접근하는 동안은 2 배 정도 차이가 날 수 있다.

다음은 NUMA 환경에서 메모리의 상태를 자세하게 확인할 때 사용하는 명령어인 `numastat` 이다.

주로 노드의 메모리 균형을 확인하고 한쪽에 메모리가 가득차서 스왑을 하는지 확인하기 위해서 사용한다.

```python
ubuntu@ip-192-168-111-182:~$ numastat -cm

Per-node system memory usage (in MBs):
Token Node not in hash table.
Token Node not in hash table.
Token Node not in hash table.
Token Node not in hash table.
Token Node not in hash table.
                 Node 0 Total
                 ------ -----
MemTotal           3867  3867
MemFree            1761  1761
MemUsed            2106  2106
Active              777   777
Inactive           1074  1074
Active(anon)          1     1
Inactive(anon)       77    77
Active(file)        776   776
Inactive(file)      997   997
Unevictable          22    22
Mlocked              18    18
Dirty                 0     0
Writeback             0     0
FilePages          1779  1779
Mapped              102   102
AnonPages            94    94
Shmem                 1     1
KernelStack           2     2
PageTables            2     2
NFS_Unstable          0     0
Bounce                0     0
WritebackTmp          0     0
Slab                195   195
SReclaimable        148   148
SUnreclaim           46    46
AnonHugePages         0     0
HugePages_Total       0     0
HugePages_Free        0     0
HugePages_Surp        0     0
```

- numastat 명령을 통해서 NUMA 아키텍처에서 메모리 불균형 상태를 확인할 수 있다. 어느 한쪽 노드만의 메모리 사용률이 높은 경우에도 swap 을 사용할 수 있기 때문이다. 분명 전체 메모리 기준으로 free 영역이 많다고 하더라도 메모리 정책에 따라서 한쪽 노드에만 메모리를 많이 할당하도록 했다면 swap 을 사용할 수 있게 된다.

이번에는 프로세스가 어떤 메모리 할당 정책으로 실행되었는지를 보자. `/proc/{PID}/numa_maps` 에서는 현재 동작 중인 프로세스의 메모리 할당 정책과 관련된 정보가 기록된다.

```python
$ sudo cat /proc/92432/numa_maps
558244dbf000 default file=/usr/sbin/sshd mapped=11 mapmax=3 N0=11 kernelpagesize_kB=4
558244dca000 default file=/usr/sbin/sshd mapped=127 mapmax=3 N0=127 kernelpagesize_kB=4
558244e49000 default file=/usr/sbin/sshd mapped=39 mapmax=3 N0=39 kernelpagesize_kB=4
```

- 이 프로세스는 default 정책으로 할당되었다.****

### ****numad 를 이용한 메모리 할당 관리****

리눅스에선 numad 를 통해서 NUMA 메모리 할당 정책을 직접 설정하지 않고도 메모리 지역성을 높일 수 있는 방법을 제공해준다.

numad 는 백그라운드 데몬과 같은 형태로 시스템에 늘 상주해있으면서 프로세스들의 메모리 할당 과정을 지켜보고 프로세스가 필요로 하는 메모리 크기가 노드보다 작다면 한쪽 노드에서만 메모리를 할당하도록 해서 메모리 지역성을 높여 최적화를 해준다.

다수의 프로세스를 사용하는 경우에 default 정책이라면 메모리가 여러 곳으로 흩어지면서 성능이 다소 떨어질 수 있다. 하지만 numad 를 실행시킨다면 하나의 프로세스가 필요로 하는 메모리를 하나의 노드에서만 할당하도록 해준다.

이 경우에 조심해야 할 것은 하나의 노드에만 집중적으로 하기 때문에 메모리 불균형이 생겨서 스왑을 사용하는 경우다. ****

예를 들면 Process 1 이 Node B 의 메모리를 모두 차지하고 있는데 Process 2 의 경우는 NUMA 정책을 interleaved 으로 해서 스왑이 발생하는 경우. 

### ****vm.zone_reclaim_mode 커널 파라미터****

numad 말고도 NUMA 아키텍처에서 메모리 할당에 영향을 줄 수 있는 커널 파라미터가 있는데 이건 `vm.zone_reclaim_mode` 이다.

`vm.zone_reclaim_mode` 에 대해서 이야기 하기 전에 zone 이 무엇인지 먼저 살펴보자.

커널은 메모리를 사용 용도에 따라 zone 이라 불리는 영역으로 구분해서 관리한다.

zone 에 대한 정보는 `/proc/buddyinfo` 에서 살펴볼 수 있다. (buddyinfo 는 메모리 할당 과정에서 이전에 살펴봤다.)

```python
ubuntu@ip-192-168-111-182:~$ cat /proc/buddyinfo
Node 0, zone      DMA      1      0      0      1      2      1      1      1      0      1      3
Node 1, zone    DMA32   3428   2965   1761    328    115     13      5      1      4     22    393
Node 0, zone   Normal   1888   1077    510    103     18     17     10      3      6      0      0
```

- Node 0 은 3 개의 zone 으로 구별된다. DMA, DMA32, Normal.
- DMA 는 Direct Memory Access 의 약자로, 주로 오래된 하드웨어의 동작을 위해서 사용하는 메모리 공간이다. 즉 일부 하드웨어를 위한 공간이므로 필요하지 않을 수 있다.
- Normal 은 커널과 프로세스가 메모리를 필요로 할 때 요청하는 영역이다.

이제 `vm.zone_reclaim_mode` 에 대해서 보자. 이 값은 총 4 개의 값을 가질 수 있지만 실제적으로 중요한 값은 0 과 1 이다.

- 0 은 disable 을 의미하며 해당 Zone 안에서 메모리 재할당을 하지 않겠다는 의미다. 즉 메모리가 부족하면 다른 Zone 에서 가져와서 사용하겠다느 뜻이다.
- 1 은 enable 을 말하며 Zone 안에서 메모리 재할당을 하겠다는 뜻이다. 해당 Zone 에서 메모리가 부족하면 재할당을 하기 위해서 필요없는 메모리를 제거하고 재사용해보고 그래도 부족하다면 다른 Zone 에서 메모리를 할당 받아서 사용한다.

0 을 이용하면 page cache 와 같은 재할당이 가능한 메모리들도 재할당 하지않고 다른 노드에 있는 메모리를 할당 받아서 사용하므로 I/O 가 중요한 프로세스의 경우에는 vm.zone_reclaim_mode 를 0 으로 설정해서 사용하는 것이 더 좋을 수 있다.

반대로 page cache 보다 로컬 메모리의 접근이 더 유리할 때는 vm.zone_reclaim_mode 를 1 로 할당해서 가능한 동일한 노드에서 메모리를 할당 받을 수 있게 해주는 것이 좋다.

### ****NUMA 아키텍처의 메모리 할당 정책과 워크로드****

이번에는 지금까지의 내용을 바탕으로 NUMA 의 메모리 할당 정책을 어떻게 사용하는게 최적화를 할 수 있을지를 알아보자.

이것은 정답이 없는 문제이기도 해서 그냥 이런 이유로 사용할 수 있다 정도만을 알아두자.

NUMA 시스템에서 워크로드를 확인할 때 고려해봐야 할 것이 두 개가 있다.

- 프로세스가 노드의 메모리를 뛰어 넘는지
- 프로세스가 여러 CPU 가 필요한 멀티 스레드인지, 싱글 스레드인지가 중요하다.

표로 나타내면 다음과 같다.

| 스레드 개수 | 메모리 크기 |
| --- | --- |
| 싱글 스레드 | 메모리가 노드 하나를 넘지 않음 |
| 멀티 스레드 | 메모리가 노드 하나를 넘지 않음 |
| 싱글 스레드 | 메모리가 노드 하나를 넘음 |
| 멀티 스레드 | 메모리가 노드 하나를 넘음 |

먼저 싱글 스레드 + 메모리가 노드 하나를 넘지 않는 경우를 보자. 이 경우는 흔하진 않을 건데 하나의 코어만 받으면 되므로 바인딩을 통해서 로컬 메모리 엑세스를 하도록 하는게 좋다. 그리고 vm.zone_reclaim_mode 도 1 로 둬서 가능한 로컬 메모리를 향하도록 하자.

두 번째는 멀티 스레드 + 메모리가 노드 하나를 넘지 않는 경우인데 이 경우에도 cpunodebind 모드를 이용해서 여러 개의 코어에 프로세스를 바인딩 하도록 해서 로컬 메모리 엑세스를 하도록 하자.

이 경우에는 CPU Usage 에 대한 모니터링이 필요한데 특정 노드에 위치함으로써 CPU 를 독차지 할 수 있다. 모니터링 할 땐 그러므로 전체 CPU 가 아닌 개별 CPU 를 봐야한다. vm.zone_reclaim_mode 도 가능한 1 로 둬서 로컬 메모리를 사용하도록 하자.

세 번째는 싱글 스레드이고 메모리가 노드 하나를 넘는 경우인데 이 경우 부터는 어쩔 수 없이 리모트 메모리 엑세스가 일어난다. 하지만 이를 최소화 하기 위해서는 로컬 메모리 사용 비중을 높이기 위해서 동일한 cpu 에 계속해서 바인딩 하도록 하는게 좋다. 그래서 첫 번째와 마찬가지로 cpunodebind 정책을 쓰자. 그리고 어짜피 노드 하나의 메모리를 넘을 것이니 계속해서 재할당 하는 것보다는 그냥 골고루 메모리를 할당 받는게 나을수 있으니 vm.zone_reclaim_mode 는 0 으로 하는게 나을 수 있다.
네 번째는 멀티 스레드에 메모리가 노드 하나를 넘는 경우인데 실제로 이 경우가 가장 많을 수 있다. 어짜피 리모트 메모리 엑세스도 일어나고 한쪽으로 CPU 를 할당하는 것도 그렇게 좋지 않다. 그러므로 그냥 interleave 모드로 골고루 받는게 가장 나을 수 있다. 이 경우에도 vm.zone_reclaim_mode 는 0 으로 하는게 더 나아보인다.
