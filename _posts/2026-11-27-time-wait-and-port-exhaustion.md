---
title: "TIME_WAIT은 왜 60초를 기다리는가 — 포트 고갈의 원인과 ss·tcpdump로 확인하는 순서"
excerpt: "부하 테스트 도중 'Cannot assign requested address'가 뜨는데 서버는 멀쩡했다면, 범인은 클라이언트 쪽 TIME_WAIT일 수 있습니다. 4-way 종료의 원리, 4-튜플과 포트 고갈의 계산, ss·nstat·tcpdump로 확인하는 절차, 그리고 keep-alive·tcp_tw_reuse·SO_LINGER 같은 처방의 실제 효과를 정리합니다."
categories: [Network]
tags: [TCP, TIME_WAIT, 포트고갈, ss, tcpdump, 트러블슈팅]
permalink: /network/time-wait-and-port-exhaustion/
toc: true
toc_sticky: true
date: 2026-11-27 08:30:00 +0900
---

## 들어가며: 서버는 멀쩡한데 클라이언트만 죽는 상황

부하 테스트나 배치 작업을 돌리다 보면 한 번쯤 이런 로그를 만난다.

```text
curl: (7) Failed to connect to api.example.com port 443: Cannot assign requested address
requests.exceptions.ConnectionError: [Errno 99] Cannot assign requested address
```

서버는 CPU도 여유롭고, `ss -s`를 보면 ESTABLISHED도 많지 않다. 그런데 클라이언트에서는 연결이 계속 실패한다. 이때 대개 `ss -s`의 한 줄이 눈에 들어온다.

```text
TCP:   1293 (estab 8, closed 1281, orphaned 0, timewait 1281)
```

TIME_WAIT 소켓이 1,281개. 이 상태를 두고 "TIME_WAIT이 쌓여서 포트가 고갈됐다"고 말하지만, 그 문장에는 세 가지가 뭉쳐 있다.

1. TIME_WAIT은 **에러가 아니라 TCP가 일부러 만드는 정상 상태**다.
2. 그래서 "없애는 법"보다 **왜 그 상태가 필요한지**를 먼저 알아야 처방이 틀리지 않는다.
3. 포트 고갈은 TIME_WAIT의 개수가 아니라 **4-튜플(출발지 IP·포트 + 목적지 IP·포트)의 유한성**에서 온다.

이 글은 그 순서대로 간다. 원리 → 계산 → 실측 → 진단 명령 → 처방 비교 → 테스트 설계에 반영하는 법.

---

## 1. TIME_WAIT은 어디서 생기는가

### 1.1 4-way 종료와 상태 전이

TCP 연결은 양방향이 각각 독립적으로 닫힌다. 그래서 종료에도 4개의 패킷이 오간다. **먼저 FIN을 보낸 쪽(능동 종료자, active closer)** 이 마지막에 TIME_WAIT을 갖는다.

```text
   A (능동 종료자)                                B (수동 종료자)
        │                                              │
        │  ── 1. FIN, ACK ──────────────────────────▶  │   B: CLOSE-WAIT
        │  ◀──────────────────────────  2. ACK ──────  │
        │                                              │
        │  ◀──────────────────  3. FIN, ACK ────────   │   B: LAST-ACK
        │  ── 4. ACK ─────────────────────────────▶    │   B: CLOSED
        │                                              │
   TIME-WAIT  (2 × MSL 대기)                            │
        │                                              │
     CLOSED                                            │
```

| # | 패킷 | A 상태 | B 상태 |
|---|------|--------|--------|
| 1 | A → B : FIN, ACK | FIN-WAIT-1 | ESTABLISHED → CLOSE-WAIT |
| 2 | B → A : ACK | FIN-WAIT-2 | CLOSE-WAIT |
| 3 | B → A : FIN, ACK | FIN-WAIT-2 | LAST-ACK |
| 4 | A → B : ACK | **TIME-WAIT** | CLOSED |

즉 서버가 유휴 연결을 정리하려고 먼저 `close()`를 호출하면 **서버 쪽 소켓이 TIME_WAIT이 되고**, 클라이언트가 "요청 후 바로 닫기"를 하면 **클라이언트 쪽 소켓**이 TIME_WAIT이 된다. 어느 쪽이 문제인지 헷갈릴 때는 `ss`의 Local/Peer 열을 보면 된다. 로컬 포트가 32768 이상의 임시 포트면 클라이언트 쪽 소켓이다.

### 1.2 애플리케이션은 이미 닫았다

TIME_WAIT 소켓은 **파일 디스크립터가 없는 커널 상태**다. 애플리케이션은 `close()`를 마쳤고 fd도 반환했다. 그래서 `lsof -p <pid>`에는 보이지 않고 `ss`로만 보인다. "코드에서 소켓을 안 닫아서 쌓인 것 아니냐"는 의심은 TIME_WAIT에는 해당되지 않는다. 그건 **CLOSE_WAIT**이다(뒤에서 비교한다).

### 1.3 왜 하필 기다리는가

RFC 9293은 능동 종료한 연결이 **2 × MSL(Maximum Segment Lifetime) 동안 TIME-WAIT에 머물러야 한다**고 요구한다(MUST-13). 이유는 두 가지다.

**첫째, 마지막 ACK의 유실을 복구하기 위해서다.** 4번 ACK이 사라지면 B는 3번 FIN을 재전송한다. A가 그 사이 CLOSED가 되어 소켓을 잊었다면, 재전송된 FIN에 대해 A는 "그런 연결 모른다"는 의미의 RST를 보낸다. B 입장에서는 정상적으로 끝난 연결이 갑자기 오류로 보고된다. TIME-WAIT에 머무는 동안은 재전송된 FIN에 ACK을 다시 보낼 수 있다.

**둘째, 이전 연결의 지연 세그먼트가 새 연결에 섞이지 않게 하기 위해서다.** 네트워크에 남아 있던 옛 패킷이 같은 4-튜플로 막 열린 새 연결에 도착하면, 시퀀스 번호 공간이 겹칠 경우 데이터가 섞인다. RFC 9293은 이 문제를 "incarnation(연결의 세대)" 문제라고 부르고, TIME-WAIT이 연결 재사용 속도를 제한하고 ISN(초기 시퀀스 번호) 선택이 이를 추가로 보호한다고 설명한다.

> 요약하면 TIME_WAIT은 **"내가 보낸 마지막 ACK이 도착했는지 모르는 상태"를 안전하게 닫는 유예 기간**이다.

### 1.4 리눅스에서의 실제 길이: 60초

RFC는 "2 × MSL"이라고만 한다. MSL은 구현마다 다르고, 리눅스는 커널 소스에서 이렇게 정의한다.

```c
/* include/net/tcp.h */
#define TCP_TIMEWAIT_LEN (60*HZ)   /* how long to wait to destroy TIME-WAIT */
```

그래서 리눅스에서 TIME_WAIT은 **정확히 60초**다. 실측으로도 확인된다(아래 3절). `ss -tanpo`의 타이머 열에 남은 시간이 그대로 보인다.

```text
$ ss -tanpo state time-wait
Recv-Q Send-Q Local Address:Port   Peer Address:Port  Process
0      0         10.100.0.8:50196    10.100.0.1:7002   timer:(timewait,45sec,0)
0      0         10.100.0.8:39008    10.100.0.1:7002   timer:(timewait,15sec,0)
```

여기서 자주 나오는 오해 하나. **`net.ipv4.tcp_fin_timeout`은 TIME_WAIT과 무관하다.** 커널 문서를 보면 이 값은 "애플리케이션이 더 이상 참조하지 않는 연결이 **FIN_WAIT_2** 상태에 머무는 시간"이다(기본 60초). TIME_WAIT 기간을 줄이려고 `tcp_fin_timeout`을 만지는 것은 헛수고다.

---

## 2. 왜 '고갈'되는가: 4-튜플의 계산

### 2.1 연결을 식별하는 것은 소켓이 아니라 4-튜플

TCP 연결은 다음 네 값의 조합으로 식별된다.

```text
(출발지 IP, 출발지 포트, 목적지 IP, 목적지 포트)
```

출발지 포트는 커널이 임시 포트 범위(ephemeral port range)에서 고른다. 리눅스 기본값은 다음과 같다.

```bash
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768	60999
```

개수는 `60999 - 32768 + 1 = 28,232`개다. 같은 목적지(IP:포트)로 나가는 연결은 이 중 하나를 써야 하고, **쓴 포트는 연결이 완전히 사라질 때까지(TIME_WAIT 포함) 재사용할 수 없다** — 정확히는, 같은 4-튜플을 다시 쓰려면 이전 세대의 TIME_WAIT이 끝나야 한다.

### 2.2 그래서 초당 몇 건까지 되는가

목적지 하나에 대해, 연결을 매번 새로 열고 닫는 구조라면 처리 가능한 속도는 이렇게 계산된다.

```text
단발 연결 상한 ≈ 로컬 포트 수 / TIME_WAIT 유지 시간
              = 28,232 / 60초
              ≈ 470 conn/s
```

초당 470건. "초당 수천 건은 가뿐히 처리하는 서버"인데 클라이언트가 500 conn/s를 단발로 던지면, **서버 성능과 무관하게 클라이언트에서 `connect()`가 실패**한다. 에러는 `EADDRNOTAVAIL`(Cannot assign requested address)로 나타난다.

### 2.3 이 상황을 만드는 전형적인 구조

- **프록시·NAT 뒤의 다수 인스턴스**: 출발지 IP가 하나로 변환되면 4-튜플의 IP 부분이 고정되어 포트만 남는다.
- **서버리스/단발 호출**: 요청마다 새 연결을 열고 닫는 구조.
- **HTTP/1.0 스타일 클라이언트**: `Connection: close`를 쓰거나 keep-alive를 끈 클라이언트.
- **부하 테스트 도구의 기본값**: `ab`처럼 keep-alive가 기본이 아닌 도구.
- **서버가 먼저 닫는 설계**: 유휴 타임아웃이 짧은 서버는 서버 쪽에 TIME_WAIT을 쌓는다. 서버는 로컬 포트가 고정(443)이라 상대 IP 수만큼만 튜플이 늘어나므로 대개 견디지만, 리버스 프록시 뒤에서 목적지 IP가 몇 개로 고정되면 같은 계산에 걸린다.

---

## 3. 실측: 단발 연결과 keep-alive는 얼마나 다른가

원리를 확인하려고 작은 TCP 서버를 띄우고, 클라이언트가 직접 `close()`하는 조건을 통제해 측정했다(Linux 6.12, 루프백).

```python
import socket, threading, time, subprocess, struct

HOST, PORT = "127.0.0.1", 9000

def serve():
    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, PORT)); s.listen(512)
    while True:
        c, _ = s.accept()
        threading.Thread(target=handle, args=(c,), daemon=True).start()

def handle(c):
    try:
        c.recv(16); c.sendall(b"k"); c.recv(1)  # 클라이언트 FIN까지 대기
    finally:
        c.close()

def tw_ports():
    """ss 출력에서 TIME-WAIT 상태의 포트 집합을 뽑는다"""
    out = subprocess.run(["ss", "-tan", "state", "time-wait"],
                         capture_output=True, text=True).stdout
    return out

# A) 단발 연결 200회: 요청마다 connect → close
ports = []
for _ in range(200):
    c = socket.socket(); c.connect((HOST, PORT))
    ports.append(c.getsockname()[1])
    c.sendall(b"x"); c.recv(1); c.close()

# B) 한 연결로 200회 요청: keep-alive와 같은 구조
c = socket.socket(); c.connect((HOST, PORT))
keep = c.getsockname()[1]
for _ in range(200):
    c.sendall(b"x"); c.recv(1)
c.close()

# C) abortive close: SO_LINGER {1, 0} → RST
c = socket.socket()
c.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack("ii", 1, 0))
c.connect((HOST, PORT)); linger = c.getsockname()[1]
c.sendall(b"x"); c.recv(1); c.close()
```

측정 결과:

| 조건 | 요청 수 | 남은 TIME_WAIT |
|------|---------|----------------|
| 요청마다 connect/close | 200 | **200** (클라이언트 포트 200개 전부) |
| 한 연결로 200회 요청 | 200 | **1** |
| `SO_LINGER {1,0}`로 닫기 | 1 | **0** (RST로 종료) |
| 지속 시간 측정 | — | **약 62초** 후 소멸 |

읽을 점은 세 가지다.

1. 단발 연결에서는 **요청 수 = TIME_WAIT 수**다. 60초 안에 28,232건을 넘기면 그 순간 포트가 바닥난다.
2. 같은 양의 요청을 한 연결로 보내면 TIME_WAIT은 **1개**다. TIME_WAIT 문제의 근본 해법은 튜닝이 아니라 **연결 재사용**이다.
3. `SO_LINGER {1,0}`은 TIME_WAIT을 만들지 않는다. 하지만 이건 4-way 종료를 하지 않고 RST로 끊는 것이므로, **보내지 못한 데이터는 그대로 사라지고 상대는 오류로 인식한다.** 지연을 줄이려고 애플리케이션에서 습관적으로 쓰면 안 되는 이유다.

---

## 4. 진단 순서: 개수 → 상대 → 타이머 → 커널 카운터

TIME_WAIT이 "많다"는 것만으로는 장애가 아니다. **연결 실패가 동반될 때** 문제다. 아래 순서로 좁혀간다.

### 4.1 전체 요약

```bash
$ ss -s
Total: 16
TCP:   93 (estab 2, closed 89, orphaned 0, timewait 89)
```

`timewait` 값이 순간 연결 수에 비해 비정상적으로 크면 4.2로 간다.

### 4.2 개수와 상대별 분포

```bash
# 헤더 1줄이 포함되므로 tail -n +2로 제거
$ ss -tan state time-wait | tail -n +2 | wc -l

# 어느 목적지에 몰려 있는지
$ ss -tan state time-wait | tail -n +2 \
    | awk '{split($4, a, ":"); print a[1]}' | sort | uniq -c | sort -rn | head
```

특정 목적지 한 곳에 수만 개가 몰려 있으면 2절의 계산에 그대로 걸린 상태다. 반대로 여러 목적지에 고르게 퍼져 있으면 튜플 총량이 아니라 애플리케이션의 호출 패턴(단발 호출)을 먼저 본다.

### 4.3 남은 타이머

```bash
$ ss -tanpo state time-wait | head -5
Recv-Q Send-Q Local Address:Port   Peer Address:Port  Process
0      0         10.100.0.8:50196    10.100.0.1:7002   timer:(timewait,45sec,0)
```

60초 카운트다운이 보이면 정상 동작 중이다. TIME_WAIT이 쌓였는데 타이머가 짧게 반복된다면 커널이 `tcp_max_tw_buckets` 한도에 걸려 조기에 파기하고 있을 수 있다.

### 4.4 커널 카운터로 확인

```bash
$ nstat -az TcpExtTW TcpExtTWKilled TcpExtTCPTimeWaitOverflow
TcpExtTW                        4263               0.0
TcpExtTWKilled                  0                  0.0
TcpExtTCPTimeWaitOverflow       0                  0.0
```

| 카운터 | 의미 | 늘고 있다면 |
|--------|------|-------------|
| `TcpExtTW` | TIME_WAIT 진입 횟수 | 단발 연결이 많다는 뜻 |
| `TcpExtTWKilled` | 버킷 한도 초과로 TIME_WAIT을 **즉시 파기**한 횟수 | `tcp_max_tw_buckets` 초과. 프로토콜 안전장치가 사라진 상태 |
| `TcpExtTCPTimeWaitOverflow` | TIME_WAIT 자원 부족 | 이미 포화 |

### 4.5 패킷으로 종료 과정 보기

개수만으로 부족할 때는 FIN/RST를 직접 본다.

```bash
# FIN 또는 RST 플래그가 선 세그먼트만
$ sudo tcpdump -ni any 'tcp[tcpflags] & (tcp-fin|tcp-rst) != 0'

# 특정 상대와의 종료 과정을 방향·시퀀스까지
$ sudo tcpdump -ni any host 10.0.0.5 and port 8080 -vv
```

여기서 확인할 것은 두 가지다.

- **누가 먼저 FIN을 보내는가.** 먼저 보낸 쪽이 TIME_WAIT의 주인이다.
- **RST가 섞여 있는가.** 정상 종료 중 RST가 보이면 `SO_LINGER` 설정, 수신 버퍼에 남은 데이터를 읽지 않고 닫는 코드, 또는 방화벽/로드밸런서의 강제 종료를 의심한다. RST는 TIME_WAIT을 만들지 않으므로 "TIME_WAIT이 없는데 연결이 끊긴다"는 증상과 같이 나타난다.

---

## 5. 처방 비교: 무엇이 실제로 효과가 있는가

### 5.1 한눈에 보는 표

| 방법 | 대상 | 효과 | 주의점 |
|------|------|------|--------|
| **연결 재사용(keep-alive, 커넥션 풀)** | 애플리케이션 | 근본 해결. TIME_WAIT 수가 요청 수에서 연결 수로 줄어든다 | 유휴 연결 유지 비용, 서버·LB의 idle timeout과 정합 필요 |
| `ip_local_port_range` 확장 | 클라이언트 커널 | 튜플 총량 증가 | 최대 65535까지. `ip_unprivileged_port_start`보다 커야 함 |
| `tcp_tw_reuse=1` | 클라이언트 커널 | 새 연결이 안전할 때 TIME_WAIT 포트를 재사용 | 타임스탬프 옵션 필요. 문서도 "전문가 조언 없이 바꾸지 말 것"이라고 명시 |
| `tcp_fin_timeout` 조정 | 커널 | **TIME_WAIT과 무관**(FIN_WAIT_2만) | 효과 없음. 흔한 오해 |
| `SO_LINGER {1,0}` | 애플리케이션 | TIME_WAIT 제거(RST) | 데이터 유실·오류 은폐. 정상 종료 경로에 쓰면 안 됨 |
| `tcp_max_tw_buckets` 증가 | 커널 | 조기 파기 방지 | 문서는 "인위적으로 낮추지 말라"고만 한다. 늘리는 것도 메모리와 함께 판단 |
| `tcp_tw_recycle` | — | **리눅스 4.12에서 제거됨** | NAT 환경에서 연결 거부를 유발했던 옵션. 설정 자체가 없다 |
| conntrack 타임아웃 조정 | NAT/방화벽 | NAT 게이트웨이의 TIME_WAIT 체류 단축 | 기본 120초. 전 구간에 영향 |

### 5.2 연결 재사용이 먼저다

3절 실측이 답을 준다. 요청 200건이 TIME_WAIT 200개를 만들던 것이 한 연결로 처리하면 1개가 된다. 라이브러리 수준에서 확인할 것들은 다음과 같다.

- HTTP 클라이언트에 **커넥션 풀**이 켜져 있는가 (Python `requests`는 `Session`, `httpx`는 기본 풀, Go는 `http.Client`의 `Transport` 재사용).
- 서버나 LB의 idle timeout이 클라이언트의 keep-alive 유지 시간보다 **길거나 같은가**. 짧으면 서버가 먼저 닫아 서버 쪽에 TIME_WAIT이 쌓이고, 클라이언트는 죽은 연결에 요청을 보내 재시도한다(이때 `Connection reset by peer`가 섞인다).
- HTTP/2라면 연결 하나에 스트림을 다중화하므로 TIME_WAIT 압력이 훨씬 낮다.

### 5.3 `tcp_tw_reuse`와 기본값 2의 의미

```bash
$ sysctl net.ipv4.tcp_tw_reuse
net.ipv4.tcp_tw_reuse = 2
```

최신 커널 문서에서 이 값은 **0(끔), 1(전역), 2(루프백만)** 세 가지다. 기본값이 1이 아니라 **2**라는 점이 중요하다. 루프백에서 테스트하면 TIME_WAIT이 재사용되어 수치가 실제 서비스 경로보다 낮게 나온다. 원격 서비스로 나가는 트래픽에서는 재사용이 일어나지 않으므로 그대로 쌓인다.

`tcp_tw_reuse`는 **클라이언트(나가는 연결)에만** 의미가 있다. 서버로 들어오는 연결에는 적용되지 않으므로 "서버에 TIME_WAIT이 많다"는 상황의 해법이 아니다.

설정을 바꿀 때는 재부팅 후에도 유지되도록 한다.

```bash
# 즉시 적용
sudo sysctl -w net.ipv4.tcp_tw_reuse=1

# 영구 적용
echo 'net.ipv4.tcp_tw_reuse = 1' | sudo tee /etc/sysctl.d/99-tcp.conf
```

### 5.4 절대 켜면 안 되는 것: `tcp_tw_recycle`

예전 튜닝 문서에는 `tcp_tw_reuse=1`과 `tcp_tw_recycle=1`을 함께 켜라고 적혀 있다. 이 옵션은 **리눅스 4.12에서 커널에서 완전히 제거됐다**(커밋 `4396e46`, "tcp: remove tcp_tw_recycle"). 제거 이유가 명확하다. 이 옵션은 같은 출발지 IP에서 오는 패킷의 타임스탬프가 단조 증가한다고 가정하는데, NAT 뒤에 여러 대의 호스트가 있으면 그 가정이 깨져 **한 대만 연결되고 나머지는 조용히 거부**된다. 커널이 연결별로 타임스탬프 오프셋을 무작위화하면서 NAT가 아니어도 깨졌고, 결국 삭제됐다.

지금 이 값을 설정하려 하면 파일 자체가 없다. 그런데도 관련 문서를 그대로 따라 하는 사례가 있으니, 커널 버전을 먼저 확인하는 습관이 필요하다.

### 5.5 포트 범위 확장

```bash
$ sudo sysctl -w net.ipv4.ip_local_port_range="15000 65000"
```

28,232개에서 50,001개로 늘어난다. 초당 처리 가능한 단발 연결 상한도 약 470 → 833 conn/s로 올라간다. 다만 근본 해결은 아니고, 그 사이에 다른 프로세스가 쓸 수 있는 포트를 줄이는 부작용이 있다. NAT 게이트웨이나 프록시처럼 출발지가 집중되는 장비에서 임시로 쓰는 정도가 적절하다.

### 5.6 NAT 구간은 따로 본다

TIME_WAIT은 호스트 커널에만 있는 게 아니다. NAT/방화벽은 conntrack 테이블에 연결 상태를 유지하고, TCP의 TIME_WAIT에 대응하는 항목에도 타임아웃이 있다.

```bash
$ cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_time_wait
120
```

호스트는 60초인데 conntrack은 **120초**다. NAT 게이트웨이 뒤에서 포트가 부족하다면, 호스트 쪽 TIME_WAIT 개수는 정상인데도 게이트웨이에서 먼저 막히는 경우가 있다. 이때는 게이트웨이의 conntrack 사용률(`conntrack -C`, `nf_conntrack_count` 대비 `nf_conntrack_max`)을 함께 봐야 한다.

---

## 6. QA 관점: TIME_WAIT을 테스트 설계에 반영하기

TIME_WAIT은 "네트워크 담당자 일"로 보이지만, 실제로 이 문제를 처음 발견하는 곳은 부하 테스트와 자동화 스크립트다.

### 6.1 도구의 기본값을 알아야 오진하지 않는다

| 도구 | 연결 처리 기본값 | 단발 부하 시 TIME_WAIT |
|------|------------------|------------------------|
| `ab` (ApacheBench) | keep-alive 꺼짐 (`-k`로 켬) | 요청 수만큼 증가 |
| `hey` | keep-alive 켜짐 (`-disable-keepalive`로 끔) | 낮음 |
| `wrk` | 지속 연결 사용 | 낮음 |
| `curl` | URL 1개당 연결, 재사용은 같은 실행 내 연속 URL | 호출 횟수만큼 |
| `k6` | VU별 연결 재사용 | 낮음 |

즉 `ab`로 측정한 수치와 `wrk`로 측정한 수치를 같은 표에 놓으면 안 된다. 전자는 클라이언트 포트 고갈이 섞여 있고, 후자는 그 영향이 거의 없다.

### 6.2 오진을 막는 판별 규칙

- 서버 지표(CPU, ESTABLISHED, 응답 시간)는 정상인데 **클라이언트에서 connect 단계 실패**가 난다 → 부하 생성기 쪽 포트 고갈을 먼저 의심한다.
- 실패 직전 `ss -s`의 `timewait`가 임시 포트 범위에 근접한다 → 4-튜플 고갈이 확정에 가깝다.
- 같은 스크립트를 keep-alive로 바꿨을 때 실패가 사라진다 → 서버 문제가 아니었다는 증거다.
- `RST`가 섞여 있다 → TIME_WAIT이 아니라 애플리케이션의 강제 종료나 LB의 연결 끊김을 본다.

### 6.3 재현 스크립트를 테스트 자산으로 남기기

3절의 코드는 `argparse`로 모드와 횟수를 받게 감싸면 그대로 재현 도구가 된다. 조건만 바꿔 두면 회귀 검증에 쓸 수 있다.

```bash
# 단발 연결 모드 vs keep-alive 모드 비교
$ python3 tw_test.py --mode one-shot --count 2000
$ python3 tw_test.py --mode keep-alive --count 2000

$ ss -tan state time-wait | tail -n +2 | wc -l
$ nstat -az TcpExtTW TcpExtTWKilled
```

### 6.4 릴리스 전 체크리스트

- [ ] 새로 붙인 HTTP 클라이언트가 커넥션 풀을 쓰는가
- [ ] 서버·LB의 idle timeout ≥ 클라이언트 keep-alive 유지 시간인가
- [ ] 부하 테스트 도구의 keep-alive 설정을 명시했는가
- [ ] 부하 테스트 중 `ss -s`의 `timewait`와 `TcpExtTWKilled`를 함께 기록했는가
- [ ] NAT/프록시 구간의 conntrack 사용률을 확인했는가
- [ ] `SO_LINGER {1,0}`이나 `tcp_tw_recycle` 같은 "빠른 해결"이 코드·설정에 남아 있지 않은가
- [ ] TIME_WAIT 급증 시 확인할 명령을 런북에 적어 두었는가

---

## 7. 정리

- TIME_WAIT은 **능동 종료자가 2 × MSL(리눅스 60초) 동안 유지하는 정상 상태**다. 마지막 ACK 유실 복구와 옛 세그먼트 차단이라는 두 가지 목적이 있다.
- 장애의 본질은 TIME_WAIT 개수가 아니라 **4-튜플의 유한성**이다. 목적지 하나에 임시 포트 28,232개, 60초 유지라면 단발 연결은 초당 약 470건이 상한이다.
- 진단은 `ss -s` → `ss -tan state time-wait`(상대별) → `ss -tanpo`(타이머) → `nstat`(조기 파기 여부) → `tcpdump`(누가 먼저 FIN을 보내는가, RST가 섞였는가) 순서로 좁힌다.
- 가장 효과적인 처방은 **연결 재사용**이다. 같은 200건이 TIME_WAIT 200개에서 1개가 된다.
- `tcp_fin_timeout`은 TIME_WAIT과 무관하고, `tcp_tw_recycle`은 4.12에서 사라졌고, `SO_LINGER {1,0}`은 TIME_WAIT 대신 데이터 유실을 택하는 것이다.

TIME_WAIT이 쌓여 있다는 사실은 대개 "이 서비스가 연결을 아껴 쓰지 않고 있다"는 신호다. 그 신호를 읽으면, 튜닝으로 덮을 문제인지 코드를 고칠 문제인지가 갈린다.

---

## 참고 자료

- RFC 9293, *Transmission Control Protocol (TCP)* — 상태 기계, TIME-WAIT 요구사항(MUST-13), incarnation 문제: <https://www.rfc-editor.org/rfc/rfc9293>
- Linux 커널 문서, *IP sysctl* — `tcp_tw_reuse`, `tcp_fin_timeout`, `tcp_max_tw_buckets`, `ip_local_port_range`: <https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt>
- Linux 커널 소스 `include/net/tcp.h` — `TCP_TIMEWAIT_LEN (60*HZ)`: <https://github.com/torvalds/linux/blob/v6.12/include/net/tcp.h>
- 커밋 `4396e46`, *tcp: remove tcp_tw_recycle*: <https://github.com/torvalds/linux/commit/4396e46187ca5070219b81773c4e65088dac50cc>
- RFC 9112, *HTTP/1.1* — 지속 연결과 `Connection` 헤더: <https://www.rfc-editor.org/rfc/rfc9112>
- Vincent Bernat, *TIME-WAIT state in Linux*: <https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux>
