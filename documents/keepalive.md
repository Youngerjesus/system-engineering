# TCP Keepalive 를 이용한 세션 유지

여기서는 TCP Keepalive 옵션을 이용해서 TCP 기반의 통신에서 세션을 유지하는 방법에 대해서 알아보려고 한다.

이전에도 말했지만 keep-alive 는 두 종단 간 맺은 세션을 이용해서 통신이 일어날 때마다 3-way-handshake 와 4-way-handshake 를 하지 않고 유지중인 세션을 이용하는 방식이다.

여기서는 이를 통해 시스템이 얻는 것은 뭐고, 주의해야 할 점이 뭔지 알아보자.

## TCP Keepalive 란?

TCP Keepalive 가 세션을 유지하는 방식은 일정 시간이 지나면 연결된 세션에서 두 종단이 살아있는지 확인하는 아주 작은 양의 패킷을 보내고 수신하는 걸 통해서 일어난다.

양쪽 모두 이 패킷을 보낼 필요는 없고 한 쪽 에서만 보내도 된다. 즉 클라이언트 서버 둘 중 하나라도 이 기능을 사용한다면 세션은 유지된다.

서로 Keepalive 를 확인하기 위한 패킷을 주고 받은 후 다시 타이머는 돌아가는 식으로 주기를 가지면서 확인을 한다.

현재 사용하고 있는 네트워크 소켓이 Keepalive 를 사용하고 있는지 확인하고 싶다면 `netstat` 명령을 통해서 보면 된다.

```bash
ubuntu@ip-192-168-111-182:~$ netstat -napo
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 192.168.111.182:22      61.177.173.12:27908     SYN_RECV    -                    on (2.15/5/0)
tcp        0    404 192.168.111.182:22      223.62.174.78:34873     ESTABLISHED -                    keepalive (70.54/0)
tcp        0      0 192.168.111.182:36638   52.78.32.75:80          TIME_WAIT   -                    timewait (37.96/0/0)
tcp6       0      0 :::22                   :::*                    LISTEN      -                    off (0.00/0/0)
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -                    off (0.00/0/0)
udp        0      0 192.168.111.182:68      0.0.0.0:*                           -                    off (0.00/0/0)
raw6       0      0 :::58                   :::*                    7           -                    off (0.00/0/0)
```

- 여기 TIMER 항목에 keepalive 가 있으면 소켓이 Keepalive 를 사용하고 있다는 뜻이다.

TCP Keepalive 를 사용하고 싶다면 소켓을 생성할 때 소켓 옵션을 주면 된다. 소켓 옵션은 ``setsockopt()`` 라는 함수를 통해서 소켓 옵션을 설정하는데 이 옵션에서 ``SO_KEEPALIVE`` 를 선택하면 TCP Keepalive 를 사용하는게 가능하다.

이건 소켓을 직접 생성하는 경우면 이렇고 대부분의 어플리케이션에서는 TCP Keepalive 를 지원하는 별도의 옵션이 있을 것이다.

## TCP Keepalive 의 파라미터들

TCP Keepalive 를 유지하는 데 필요한 커널 파라미터들은 총 3 개가 있다.

- ``net.ipv4.tcp_keepalive_time`` : 가장 중요한 값인 tcp_keepalive_time 을 보자. 이름이 의미하는 것처럼 소켓의 유지 시간을 말한다. 타이머는 이 시간을 기준으로 동작하며 이 시간이 지나면 keepalive 확인 패킷을 보낸다. 이 값을 직접 지정하는게 가능하고 기본 값은 240 초이다.
- ``net.ipv4.tcp_keepalive_probes``: tcp_keepalive_probe 는 keepalive 패킷을 보낼 최대 전송 횟수를 말한다. keepalive 패킷에 한번 응답하지 않았다고 해서 바로 연결을 종료하는 건 아니다. 네트워크 상의 패킷은 언제든지 다양한 원인으로 유실될 수 있다는 걸 알아야한다. 그렇다고 무한정 재전송하는 건 아니니 tcp_keepalive_probes 로 최대 재전송 횟수를 정해야 한다. 기본 값은 3 으로 최초의 keepalive 패킷을 포함해서 총 3 번의 패킷을 보내고 그 후에도 응답이 없으면 연결을 끊는다.
- `net.ipv4.tcp_keepalive_intvl` : tcp_keepalive_intvl 은 재전송 패킷을 보내는 주기를 말한다.

원래는 소켓이 종료할 때 FIN 패킷을 보내면서 종료를 한다. keepalive 를 사용할 땐 재전송을 한 후에도 응답이 없는 경우에는 FIN 패킷에 대한 응답을 보낼 수 없는 상태일 것이니 커널에서 알아서 끊어진 세션으로 판단하고 소켓을 종료한다.

## TCP Keepalive 와 HTTP Keepalive

흔히 TCP Keepalive 와 HTTP Keepalive 를 혼동하는 경우가 많다. 여기서는 이 두 Keepalive 의 차이점에 대해서 알아보자.

아파치와 nginx 와 같은 웹 어플리케이션에서도 keepalive timeout 이라는 것이 존재한다. HTTP/1.1 에서 지원하는 keepalive 를 위한 설정 항목이다. 용어가 비슷해서 헷갈릴 수 있지만 두 항목은 큰 차이가 있다.

TCP Keepalive 는 두 종단 간의 연결을 유지하기 위함이지만, HTTP Keepalive 는 최대한 연결을 유지하는 것이 목적이다. 만약 두 값 모두 60 초라면 TCP Keepalive 는 60 초 간격으로 연결이 유지되었는지를 확인한다면 HTTP Keepalive 는 60 초 후에도 연결이 안오면 요청을 끊는다.

두 값이 다르면 어떤 일이 벌어질지 걱정할 수 있지만 TCP Keepalive 의 연결을 확인하려는 패킷이 HTTP Keepalive 의 연결 유지에 영향을 주지 않는다. 즉 TCP Keepalive 를 HTTP Keepalive 보다 높게 설정하던 낮게 설정하던 HTTP Keepalive 에서 설정한 시간동안 HTTP 요청이 오지 않는다면 소켓은 종료된다.
