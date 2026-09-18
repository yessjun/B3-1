# 트러블슈팅

아래 기록에서 작업에 사용한 회선의 공인 IP는 가렸습니다.

## 외부에서 접속이 되지 않는 경우

구성을 마친 뒤, 보안 그룹 인바운드 규칙 하나가 빠졌을 때 어떤 증상이 나타나고 어떤 순서로 원인을 좁히는지 확인하기 위해 HTTP 규칙을 제거한 상태를 재현했습니다.

### 증상

로컬에서 퍼블릭 IP로 요청하면 응답이 오지 않고 타임아웃됩니다.

```bash
$ curl -sS --max-time 15 -o /dev/null -w "%{http_code}\n" http://43.201.149.54
curl: (28) Connection timed out after 15002 milliseconds
000
```

규칙을 제거한 직후 첫 요청은 200을 돌려줬고, 몇 초 뒤 재시도부터 타임아웃으로 바뀌었습니다. 규칙 변경이 반영되는 데 시간이 걸리므로 변경 직후 한 번의 결과만으로 판단하지 않았습니다.

### 원인 가설

연결 거부가 아니라 타임아웃이라는 점이 출발점입니다. 포트가 닫혀 있는데 요청이 서버까지 도달하면 서버가 RST를 보내 즉시 거부되고, 중간에서 패킷이 버려지면 응답 자체가 없어 타임아웃됩니다. 따라서 요청이 인스턴스에 도달하지 못했다고 보고 후보를 셋으로 잡았습니다.

1. 라우트 테이블의 0.0.0.0/0 경로가 사라졌거나 인터넷 게이트웨이 연결이 끊어진 경우
2. 보안 그룹 인바운드에서 80번이 막힌 경우
3. 인스턴스 안에서 nginx가 죽은 경우

### 검증

SSH(22번)는 그대로 접속됩니다. 같은 인스턴스, 같은 라우팅 경로를 쓰는 접속이 되므로 1번과 인스턴스 자체의 정지는 후보에서 빠집니다.

인스턴스 안에서 확인한 결과 nginx는 80번을 열고 있고 자기 자신에 대한 요청에 200을 돌려줍니다.

```bash
$ ssh -i b3-1-key.pem ubuntu@43.201.149.54 'curl -sS -o /dev/null -w "localhost:%{http_code}\n" http://localhost; ss -tlnp 2>/dev/null | grep ":80"; sudo tail -3 /var/log/nginx/access.log'
localhost:200
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*
LISTEN 0      511             [::]:80           [::]:*
xxx.xxx.xxx.xx - - [18/Sep/2026:03:04:49 +0000] "GET /favicon.ico HTTP/1.1" 404 196 "http://43.201.149.54/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/153.0.0.0 Safari/537.36"
xxx.xxx.xxx.xx - - [18/Sep/2026:03:11:42 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.7.1"
::1 - - [18/Sep/2026:03:12:29 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0"
```

액세스 로그의 마지막 외부 요청은 03:11:42에 200으로 끝난 것이고, 그 뒤 타임아웃된 요청은 로그에 남지 않았습니다. 요청이 nginx까지 도달하지도 못했다는 뜻이므로 남은 후보는 보안 그룹입니다. 실제 규칙을 확인하니 22번만 남아 있습니다.

```bash
$ aws ec2 describe-security-groups --group-ids sg-0d48678d6a69187e0 \
    --query 'SecurityGroups[0].IpPermissions[].{Protocol:IpProtocol,From:FromPort,Cidr:IpRanges[0].CidrIp}' --output table
-------------------------------------------
|         DescribeSecurityGroups          |
+--------------------+-------+------------+
|        Cidr        | From  | Protocol   |
+--------------------+-------+------------+
|  xxx.xxx.xxx.xx/32 |  22   |  tcp       |
+--------------------+-------+------------+
```

### 조치

80번 인바운드 규칙을 다시 추가했습니다.

```bash
$ aws ec2 authorize-security-group-ingress --group-id sg-0d48678d6a69187e0 \
    --protocol tcp --port 80 --cidr 0.0.0.0/0 \
    --query 'SecurityGroupRules[0].SecurityGroupRuleId' --output text
sgr-0b78ba5b81553ed38
```

### 결과

로컬에서 다시 요청하면 200이 돌아옵니다.

```bash
$ curl -sS --max-time 15 -o /dev/null -w "%{http_code}\n" http://43.201.149.54
200
```

### 재발 방지

- 접속 불가 증상은 바깥에서 안쪽 순서로 좁힙니다. 라우팅, 보안 그룹, 퍼블릭 IP, 서버 프로세스와 로그 순입니다. 서버 로그에 요청 기록 자체가 없으면 네트워크 구간, 기록은 있는데 응답이 이상하면 서버 구간으로 나눠서 봅니다.
- 타임아웃과 연결 거부를 구분합니다. 거부는 서버까지 도달한 것이고 타임아웃은 도달하지 못한 것이라, 첫 화면에서 이미 범위가 갈립니다.
- 보안 그룹을 변경한 뒤에는 `describe-security-groups`로 실제 규칙 목록을 확인하고, 반영에 몇 초가 걸리므로 변경 직후 한 번의 요청만으로 판단하지 않습니다.
