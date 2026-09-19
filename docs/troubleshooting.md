# 트러블슈팅

## 1. 인스턴스 시작 화면에서 권한 오류가 뜨는 경우

### 증상

보안 그룹을 고르자 콘솔이 노란 경고를 띄웁니다. `ec2:GetSecurityGroupsForVpc` 작업을 수행할 권한이 없어 선택한 보안 그룹의 유효성을 검사할 수 없다는 내용입니다.

![인스턴스 시작 화면의 권한 경고](assets/console-launch-settings.png)

인스턴스 세부 정보 화면에서도 비슷한 오류가 하나 더 나옵니다. `compute-optimizer:GetEnrollmentStatus` 권한이 없다는 메시지입니다.

### 원인 가설

이 실습의 IAM 사용자에게는 EC2와 VPC를 만들고 지우는 데 필요한 액션만 허용했습니다. 콘솔은 화면을 그리면서 실제 작업에 필요한 API 외에도 부가 정보를 가져오는 API를 함께 호출하므로, 정책에 없는 호출이 섞여 있을 것이라고 보았습니다. 그렇다면 경고는 화면 표시에만 영향을 주고 인스턴스 시작 자체는 진행될 것입니다.

### 검증

메시지에 찍힌 액션 이름을 정책과 대조했습니다. 정책에 넣은 것은 `ec2:Describe*`와 생성/삭제/연결 계열이고, `ec2:GetSecurityGroupsForVpc`와 `compute-optimizer:GetEnrollmentStatus`는 둘 다 없습니다. 두 액션 모두 조회용이고 인스턴스를 만드는 `ec2:RunInstances`와는 별개입니다.

경고를 그대로 두고 인스턴스 시작을 눌러 실제로 시작되는지 확인했습니다.

### 조치

권한을 추가하지 않았습니다. 경고가 뜬다고 `ec2:*`를 붙이면 실습에 필요 없는 권한까지 한꺼번에 열리고, 왜 그 권한이 있는지 설명할 수 없게 됩니다. 권한 부족이 실제로 작업을 막았다면 그때 메시지에 찍힌 액션 하나만 정책에 추가하는 것이 순서입니다.

### 결과

인스턴스가 정상적으로 시작됐고 보안 그룹도 의도한 `b3-1-web-sg`가 붙었습니다.

![인스턴스 세부 정보](assets/console-instance-detail.png)

### 재발 방지

- 권한 오류를 만나면 먼저 그 호출이 목표 작업에 필요한 것인지, 화면 표시용 부가 호출인지 구분합니다. 후자면 권한을 늘리지 않습니다.
- 늘려야 할 때는 오류 메시지의 액션 이름 하나만 추가합니다. 서비스 전체 권한이나 관리자 정책으로 해결하지 않습니다.
- 콘솔에서 이런 경고가 보이는 것은 권한이 실제로 좁다는 뜻이기도 합니다. 넓게 열어두면 경고 자체가 나오지 않습니다.

## 2. 헬스체크 경로가 404로 응답한 경우

### 증상

nginx 기본 사이트 설정에 `/health`를 추가하고 곧바로 호출했더니 404가 돌아왔습니다.

```bash
$ ssh -i b3-1-key.pem ubuntu@43.201.49.58 'curl -sS http://localhost/health'
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.28.3 (Ubuntu)</center>
</body>
</html>
```

### 원인 가설

nginx가 응답을 돌려주고 있으므로 서버는 살아 있고, 요청도 서버까지 도달했습니다. 404는 그 경로에 대한 처리가 없다는 뜻이므로 후보는 셋입니다.

1. 설정 파일이 의도한 내용으로 저장되지 않은 경우
2. 저장은 됐지만 그 파일이 읽히지 않는 경우(사이트가 활성화되어 있지 않거나 다른 server 블록이 먼저 잡는 경우)
3. 저장도 됐고 읽히기도 하지만 변경이 아직 반영되지 않은 경우

### 검증

파일 내용을 확인했습니다. `/health` 블록이 그대로 들어 있습니다.

```bash
$ ssh -i b3-1-key.pem ubuntu@43.201.49.58 'cat /etc/nginx/sites-available/default'
server {
    listen 80 default_server;

    root /var/www/html;
    index index.nginx-debian.html;

    location = /health {
        default_type text/plain;
        return 200 'OK';
    }
}
```

사이트가 활성화되어 있는지와 다른 설정이 끼어 있는지도 확인했습니다. `sites-enabled/default`가 이 파일을 가리키는 심볼릭 링크이고, `nginx.conf`가 `sites-enabled/*`를 포함하며, `conf.d`에는 파일이 없습니다. 문법 검사도 통과합니다.

```bash
$ ssh -i b3-1-key.pem ubuntu@43.201.49.58 'ls -l /etc/nginx/sites-enabled/; ls /etc/nginx/conf.d/; grep -n "include" /etc/nginx/nginx.conf | tail -2'
total 0
lrwxrwxrwx 1 root root 34 Sep 19 02:48 default -> /etc/nginx/sites-available/default
60:	include /etc/nginx/conf.d/*.conf;
61:	include /etc/nginx/sites-enabled/*;
$ ssh -i b3-1-key.pem ubuntu@43.201.49.58 'sudo nginx -t'
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

1번과 2번이 빠지므로 남은 것은 3번입니다. 설정을 쓴 직후 같은 명령줄에서 reload와 curl을 연달아 실행했기 때문에, 워커 프로세스가 새 설정으로 교체되기 전에 요청이 처리된 것으로 보았습니다.

### 조치

reload를 다시 실행하고 잠시 뒤 확인했습니다.

```bash
$ ssh -i b3-1-key.pem ubuntu@43.201.49.58 'sudo systemctl reload nginx; sleep 2; curl -sS -o /dev/null -w "localhost:%{http_code}\n" http://localhost; curl -sS http://localhost/health'
localhost:200
OK
```

### 결과

`/health`가 200과 고정 응답 `OK`를 돌려줍니다. 외부에서 호출한 결과도 같습니다.

### 재발 방지

- 설정을 바꾼 뒤에는 reload가 반영될 시간을 두고 확인합니다. 변경과 확인을 한 줄에 붙여 실행하면 반영 전 상태를 보고 잘못 판단하게 됩니다.
- 404와 타임아웃을 구분합니다. 404는 서버까지 도달해 서버가 응답한 것이므로 네트워크 구간이 아니라 서버 설정을 봅니다. 타임아웃이면 반대로 보안 그룹이나 라우팅부터 확인합니다.
- 설정 파일을 고쳤을 때는 파일 내용, 활성화 여부, 문법 검사, 반영 순서로 좁힙니다. 이 순서를 지키면 어디까지 정상인지가 매 단계에서 확정됩니다.
