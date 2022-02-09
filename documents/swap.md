# swap, 메모리 증설의 포인트

## swap 영역

swap 영역은 물리 메모리가 부족한 경우를 대비해서 만들어 놓은 영역이다.

메모리가 모자라기 시작하면 프로세스는 더 이상 연산을 위한 공간을 얻을 수 없으므로 장애를 일으킬 수 있다.

이를 방지하기 위해서 비상용으로 디스크 공간에 확보해놓은 메모리 공간이 swap 영역이다.

swap 영역은 물리 메모리가 아니라 디스크 영역이기 때문에 메모리에 비해서 현저히 접근 속도가 떨어진다.

그래서 swap 영역을 사용하고 있다면 시스템 성능이 급격하게 떨어져있다고 생각하면 된다.

리눅스에서 사용중인 swap 영역은 free 명령을 통해서 볼 수 있다.

```
ubuntu@ip-192-168-111-19:~$ free -k
              total        used        free      shared  buff/cache   available
Mem:        3959676     1281980      290516         824     2387180     2383912
Swap:             0           0           0
```
- 여기에 있는 `Swap` 이 스왑영역을 말한다. 스왑 영역을 볼때는 쓰고 있느냐 안쓰고 있느냐가 중요하다.

스왑 영역을 사용하고 있다면 어떤 프로세스에서 메모리를 그렇게 많이 사용되고 있는지를 찾아야한다.

단순하게 메모리를 많이 사용하고 있다면 증설을 하거나 메모리를 줄이는 방법을 찾아야 하고

메모리가 누수인 경우라면 이를 해결할 방법을 찾아봐야한다. 관리 용도의 프로세스가 메모리를 많이 차지하고 있다면 이 프로세스를 죽이는 것만으로도 해결할 수 있다.

모든 프로세스는 `/proc/{PID}` 의 디렉토리에 자신과 관련된 정보들을 저장한다. 

프로세스가 사용하는 메모리도 이곳에 저장하는데 `/proc/{PID}/smaps` 에서 볼 수 있다.

```
$ cat /proc/19325/smaps 
...
7f68aa146000-7f68aa147000 r--p 00000000 103:01 3454                      /usr/lib/x86_64-linux-gnu/libdl-2.31.so
Size:                  4 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                   4 kB
Pss:                   0 kB
Shared_Clean:          4 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:            4 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:         0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
```

이외에 전체 프로세스별로 swap 영역을 사용하고 있는 크기를 보려면 `smem -t` 명령을 통해서 볼 수 있다.

## 버디 시스템 

커널이 메모리를 할당하는 과정을 간단하게 살펴보자.

커널은 버디 시스템을 통해서 프로세스에 메모리를 할당한다.

버디 시스템은 물리 메모리를 연속된 메모리 영역으로 관리하는데 예를 들면 연속 1 개의 페이지 크기별 버디, 연속 2 개의 페이지 크기별 버디, 연속 4 개의 페이지 크기별 버디, 연속 8 개의 페이지 크기별 버디 이런식으로 관리한다.

그러므로 8KB 메모리를 요청하면 연속 1 개 짜리를 두 개 주는게 아니라 연속 2 개 짜리 하나를 준다.

이를 통해서 메모리 단편화 현상을 막아준다.

버디 시스템의 현재 상황은 `/proc/buddyinfo` 에서 볼 수 있다.

```
ubuntu@ip-192-168-111-19:~$ cat /proc/buddyinfo
Node 0, zone      DMA      1      0      0      1      2      1      1      1      0      1      3
Node 0, zone    DMA32   1296    838    470    182    133     70     35     14     14      6     42
Node 0, zone   Normal    238    391    186     76     39     10      3      1      3      1      0
```

- 왼쪽부터 오른쪽으로 개수가 늘어난다고 보면 된다. (연속 1 개, 연속 2 개, 연속 4 개 ...) 
- 즉 여기에 있는 DMA, DMA32, Normal 영역의 버디를 다 합치면 가용영역의 메모리가 계산된다. 
- 여기서 메모리를 달라는 요청이 있으면 `/proc/buddyinfo` 에 있는 메모리의 개수는 줄어들 것이다.

## 메모리 재할당 과정 

이번엔 커널이 메모리를 해제하고 다시 할당하는 재할당하는 과정을 보자. 이 경우는 크게 두 가지 케이스가 있다.

- Page cache, Buffer cache, inode cache, dentry cache 같은 캐시 공간을 해제하고 다시 할당하는 것. (커널은 그냥 메모리를 안쓰고 남겨두는 걸 좋아하지 않는다.)
- Swap 을 위한 재할당. 캐시용도로 사용하고 있는 메모리를 제외하고 프로세스가 사용중인 메모리는 임의로 커널이 해제할 순 없다. 하지만 메모리가 가득 차있는 상황이라면 사용하는 메모리 중 inactive 영역에 있는 메모리는 swap 영역으로 내리고 요청한 메모리를 올리는 과정을 한다.


## vm.swappiness 와 vm.vfs_cache_pressure

이 두 값은 커널 내에서 사용하는 파라미터인데 메모리 재할당에 영향을 준다. 하나씩 살펴보자.

vm.swappiness 의 정의는 다음과 같다.

```
This control is used to define how aggressive the kernel will swap memory pages. Higher values will increase aggressiveness, lower values decrease the amount of swap. The default value is 60.
```

- swap 을 얼마나 적극적으로 할 지 나타내는 변수다. 
- 프로세스마다 I/O 의 성능을 높여주기 위해 Page cache 를 적극적으로 할 필요가 있다. 이 경우에 `vm.swappiness` 를 높여서 사용하면 캐시를 좀 더 적극적으로 쓸 것이니 좋을 수 있다. 

va.vfs_cache_pressure 의 정의는 다음과 같다.

```
This percentage value controls the tendency of the kernel to reclaim the memory which is used for caching of directory and inode objects
```

- 메모리 캐시 중에 inode 에 대한 캐시를 얼마나 할려는지의 여부를 말한다. 즉 page cache 를 더 많이 할 지 inode cache 를 더 많이 할 지 나타내는 값이다.

## 메모리 증설 포인트

메모리가 부족해서 swap 영역을 쓰고 있다면 메모리를 증설해야 한다고 생각할 수 있지만, 메모리가 누수되고 있는 문제일 수도 있다.

메모리가 누수되고 있는 경우는 메모리의 사용량이 시간이 지남에 따라 선형적으로 증가하는 경우다.

이 경우는 메모리 누수를 의심해볼 수 있고 pmap 등의 명령을 이용해서 해당 프로세스가 사용하는 힙 메모리 영역 변화 과정을 보거나 gdb 같은 도구를 이용해서 디버깅을 해보면 메모리 덤프를 생성해 실제 어떤 데이터들이 메모리에 있는지 확인할 수 있다.



