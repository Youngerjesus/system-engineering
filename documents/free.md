## free 명령이 숨기고 있는 것들

메모리가 부족하다면 프로세스는 더 이상 작업을 할 수 없고 이는 시스템 장애나 응답 속도가 느려지는 현상을 일으킬 수 있다.

그렇기 때문에 메모리가 어떻게 사용되고 있는지를 파악하는 것은 CPU 사용률과 Load Average 만큼 중요한 포인트다.

여기서는 메모리 상태를 전체적으로 빠르게 살펴 볼 수 있는 `free` 명령어와 `free` 만으로 살펴볼 수 없는 정보는 어떻게 볼 수 있는지 살펴보겠다.

다음은 `free -m` 명령의 출력 결과다.

```bash
ubuntu@ip-172-31-12-59:~$ free -m
              total        used        free      shared  buff/cache   available
Mem:            968         371         135           0         462         443
Swap:             0           0           0
```

- `-m` : 이 옵션을 통해서 출력되는 메모리 양은 MB 단위로 출력한다. `-b` 를 주면 byte 단위, `-k` 를 주면 KB 단위 `-g` 를 주면 GB 단위로 표시된다.
- `total`: 현재 시스템에 설치되어있는 전체 메모리양을 의미한다.
- `used`: 시스템에서 사용하고 있는 메모리양을 의미한다.
- `free`: 시스템에서 아직 사용하고 있지 않은 메모리 양을 의미한다. 그러므로 애플리케이션이 사용할 수도, 커널이 사용할 수도 있다.
- `shared`: 프로세스 사이에 공유하고 있는 메모리양이다.
- `buffred`: 버퍼 용도로 사용하고 있는 메모리 영역을 말하며 시스템의 성능 향상을 위해서 커널에서 사용하고 있는 영역이다.
- `cached`: 페이지 캐시라고 불리는 캐시 영역에 있는 메모리양을 의미한다. 페이지 캐시는 디스크 블록을 캐싱해놓음으로써 I/O 작업의 성능향상을 위해서 사용된다.
- 그리고 `-/+ buffers/cached` 가 출력되는 경우도 있는데 이는 메모리 사용량에서 bufferd 와 cached 를 제외하고, 사용하고 있는 영역과 사용하지 않는 영역을 표시해준다.
- `Swap`: Swap 의 전체 영역과 실제로 사용하고 있는 영역 그리고 사용하지 않은 영역을 표시해준다.****

### ****Buffers 와 cached 영역****

커널이 블록 디바이스라고 불리는 디스크에서 데이터를 읽어올 때는 상당히 느리다. 그래서 이를 통해 시스템 부하가 일어나기도 한다.  

 그래서 커널은 상대적으로 느린 디스크 영역을 좀 더 빠르게 하기 위해서 캐싱 영역으로 한번 읽은 파일은 Page cache 인 cached 영역에 저장하고 여기서 읽어 오도록 한다. 

파일의 내용이 아닌 파일 시스템을 관리하기 위한 메타 데이터 (super block, inode block) 을 읽어올 때는 Buffer 영역을 사용한다.

그리고 이전에 메모리에서 buffer 과 cached 영역을 제외하고 보여주는 부분이 있는데 이 이유는 이 영역의 경우에는 프로세스가 메모리를 더 필요로 하는 경우에 언제든지 해제될 수 있는 영역이기 때문이다.****

### /proc/meminfo 읽기

`free` 명령은 전체 시스템이 사용하고 있는 메모리와 가용 메모리를 빠르게 읽을 수 있지만 시스템의 메모리 전체 사용량을 알려주지는 않는다.

자세하게 보고 싶다면 `/proc/meminfo` 파일을 읽으면 된다.

```bash
ubuntu@ip-172-31-12-59:~$ cat /proc/meminfo
MemTotal:         992204 kB
MemFree:          138264 kB
MemAvailable:     454700 kB
Buffers:           37596 kB
Cached:           384688 kB
SwapCached:            0 kB
Active:           310688 kB
Inactive:         384112 kB
Active(anon):       1868 kB
Inactive(anon):   274412 kB
Active(file):     308820 kB
Inactive(file):   109700 kB
Unevictable:       23064 kB
Mlocked:           18528 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                16 kB
Writeback:             0 kB
AnonPages:        295616 kB
Mapped:            68360 kB
Shmem:               956 kB
KReclaimable:      51116 kB
Slab:              94424 kB
SReclaimable:      51116 kB
SUnreclaim:        43308 kB
KernelStack:        3712 kB
PageTables:         3976 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:      496100 kB
Committed_AS:    1668184 kB
VmallocTotal:   34359738367 kB
VmallocUsed:       19556 kB
VmallocChunk:          0 kB
Percpu:            13632 kB
HardwareCorrupted:     0 kB
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      110592 kB
```

여기서는 전부 보지말고 중요한 부분만 살펴보자.

- `SwapCached`: swap 으로 빠진 메모리 영역 중 다시 메모리로 돌아온 영역을 말한다. swap 으로 한번 메모리가 빠졌어도 메모리가 확보되면 다시 돌아올 수 있다. 그리고 커널은 한번 swap 이 되면 다시 swap 이 될 가능성이 많아서 swap 영역에 있는 메모리는 메모리로 돌아올 때도 삭제되지는 않는다. 이를 통해 I/O 를 조금이나마 줄일 수 있기 떄문에.
- `Active(anon)`: anon 은 anonymous 로 Page cache 영역을 제외한 메모리 영역을 말한다. active 는 최근까지 메모리를 사용하고 있어서 swap 영역으로 넘어가지 않는 영역을 뜻한다.
- `inactive(anon)`: swap 영역으로 넘어갈 수 있는 anon 영역을 말한다.
- `Active(file)`: buffers 와 cached 영역을 포함한 영역이고 active 이므로 최근에 참조해서 swap 영역으로 넘어가지 않는다.
- `inactive(file)`: swap 영역으로 넘어갈 수 있는 buffers 와 cached 영역을 말한다.
- `Dirty`: I/O 성능 향상을 위해 커널이 캐시 목적으로 사용하는 영역으로 그 중 쓰기 작업의 영역을 말한다. 커널은 I/O 작업으로 쓸 때 바로 블록 디바이스로 명령을 내리지 않고 이 영역에 모아서 한 번에 쓴다.

### ****slab 메모리 영역****

/proc/meminfo 에서 아직 설명하지 않은 영역이 있는데 slab 영역들이다. 

이 영역들은 커널이 내부적으로 사용하는 영역을 뜻한다.

주로 I/O 작업을 더 빠르게 하기 위해서 inode cache, dentry cache 등에 사용하거나 네트워크 소켓을 위한 메모리 영역을 확보하는 작업들을 한다.

inode_cache 는 파일의 inode 에 대한 정보를 지정해두는 캐시라면 dentry 는 디렉터리의 계층 관계를 저장해두는 캐시다.

즉 dentry 는 ls 명령으로 디렉터리를 살펴보기만 해도 값이 증가한다. 만약 파일에 자주 접근하고 디렉터리의 생성/삭제가 빈번한 시스템이라면 Slab 메모리가 늘어날 수 있다.

`/proc/meminfo` 에 있는 영역 중 `slab` 과 관련된 영역은 다음과 같다. 

- `Slab`: 메모리 영역 중 커널이 직접 사용하고 있는 영역을 말한다.
- `SReclaimable`: Slab 영역 중 캐시용도로 사용하고 있는 영역이며 메모리 부족 현상이 일어나면 프로세스에게 할당될 수 있는 영역이다.
- `SUnreclaim`: 커널이 현재 사용중인 영역을 말하며 해제할 수 없는 영역이다.

그리고 추가로 말하는건데 Slab 메모리는 buffers/cached 영역에 포함되지 않고 free 명령으로 조회했을 때 used 에 포함된다.
