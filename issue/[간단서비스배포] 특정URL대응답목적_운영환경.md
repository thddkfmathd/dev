spring boot라 jar로 묶어서 buid → 스냅샷으로 배포 ( jar - plain은 라이브러리 포함 안됨)

#### 시스템 서비스 등록

```
# mock_res.service

[Unit]
Description=Mock Response Service
After=network.target

[Service]
# 실행 사용자
User=user
# JAR이 있는 디렉토리
WorkingDirectory=/home/user/mock_response_service
# java 실행 절대경로 + JAR 실행
ExecStart=/usr/bin/java -jar /home/user/mock_response_service/mockRes-0.0.1-SNAPSHOT.jar
# 비정상 종료 시 자동 재시작
Restart=always
RestartSec=10
# 정상 종료 코드
SuccessExitStatus=143
# 출력은 systemd journal로
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload
//서비스 자동 시작 등록
systemctl enable mock_res.service
systemctl start mock_res.service
systemctl status mock_res.service

//중지
systemctl stop mock_res.service
//시작
systemctl restart mock_res.service

//저널로그에서 확인하도록 설정함 StandardOutput,StandardError
journalctl -u mock_res.service -f
```

#### 방화벽해제

```
#체크 
firewall-cmd --list-all
firewall-cmd --permanent --add-port=19880/tcp
firewall-cmd --reload
```
