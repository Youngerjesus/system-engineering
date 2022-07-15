## VMSTAT


- procs (Process)
    - r: run Queue, 작업 수행을 CPU 를 기다리는 스레드의 수
    - b: blocking Queue, I/O 를 기다리는 스 드으의
- memory
    - free: 남은 메모리의 양
    - swap: 스왑으로 사용하고 있는 메모리 양
    - buff: 버퍼로 사용하고 있는 메모리의 양
    - cache: 캐시로 사용하고 있는 메모리의 양
- swap
    - si: swap in
    - so: swap out 으로 이 값이 0 보다 높다면 swap 을 쓰고 있다는 뜼
- System
    - sy: System Call 을 말함 (OS 의 작업이 사용자의 작업보다 많아졌는지 알 수 있음.)
    - cs: Process 들의 Context Switch 를 말함. (CPU 자원에 대한 프로세스들의 경쟁)
- CPU
    - cs: CPU 컨택스트 전환 비율
    - us: 사용자에서 사용하는 CPU 비율
    - sy: 시스템 콜 호출에 의해서 사용하는 CPU 비율
    - id: 사용가능한 CPU 비율 (100 - (us + sy) = id)
    - wa: i/o 작업을 기다리는 cpu
