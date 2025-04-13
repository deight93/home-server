# Ubuntu 24.04, DuckDNS, PEM 키 기반 SSH 접속 설정 가이드

목표:  
- 맥에서 아래 한 줄 명령어로 우분투 서버에 접속  
  ```bash
  ssh myserver
  ```  
- 도메인: myserver.duckdns.org  
- SSH 포트: xxxx  
- 인증: PEM 키 기반

-------------------------------------------------

## 1. 우분투 서버 설정

### 1-1. SSH 서버 설치  
- 업데이트 및 설치  
  ```bash
  sudo apt update
  sudo apt install openssh-server
  sudo systemctl enable ssh
  sudo systemctl start ssh
  ```

### 1-2. 내부 IP 고정 (공유기)  
- 우분투 서버 IP 확인  
  ```bash
  ip a
  ```  
- 공유기 관리자 페이지에서 DHCP 예약 또는 고정 IP로 설정

### 1-3. SSH 포트 변경  
- `/etc/ssh/sshd_config` 파일 수정  
  ```bash
  sudo vi /etc/ssh/sshd_config
  ```  
  - 아래 항목 추가 또는 수정  
    ```
    Port xxxx
    PermitRootLogin no
    PasswordAuthentication no
    ```
- 변경 사항 적용  
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart ssh
  ```

### 1-4. UFW 방화벽 설정  
- 포트 xxxx 허용  
  ```bash
  sudo ufw allow xxxx/tcp
  sudo ufw enable
  ```  
- 접속 테스트 후 포트 22 제거  
  ```bash
  sudo ufw delete allow 22/tcp
  ```

-------------------------------------------------

## 2. DuckDNS 설정 (우분투 서버)

### 2-1. 도메인 등록  
- [DuckDNS 사이트](https://www.duckdns.org)에서 회원가입 후  
- `myserver` 도메인 생성 → myserver.duckdns.org

### 2-2. 갱신 스크립트 작성  
- 스크립트 디렉터리 생성 및 스크립트 작성  
  ```bash
  mkdir -p ~/duckdns
  cd ~/duckdns
  nano duck.sh
  ```  
- 스크립트 내용 (토큰은 DuckDNS에서 발급받은 것으로 대체)  
  ```
  echo "url=https://www.duckdns.org/update?domains=myserver&token=YOUR_TOKEN&ip=" > duck.sh
  chmod 700 duck.sh
  ```
  
### 2-3. 크론탭 등록  
- 5분마다 스크립트 실행하도록 등록  
  ```bash
  crontab -e
  ```  
- 아래 항목 추가  
  ```
  */5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
  ```

-------------------------------------------------

## 3. 맥에서 PEM 키 기반 접속 설정

### 3-1. PEM 키 생성 (맥)  
- 아래 명령 실행  
  ```bash
  ssh-keygen -t rsa -b 4096 -f ~/.ssh/my-ubuntu.pem
  ```  
- 생성된 파일:  
  - `~/.ssh/my-ubuntu.pem` (개인 키, 외부 노출 금지)  
  - `~/.ssh/my-ubuntu.pem.pub` (공개 키)

### 3-2. 공개키 등록 (맥 → 우분투)  
- 자동으로 공개키 복사  
  ```bash
  ssh-copy-id -i ~/.ssh/my-ubuntu.pem.pub user@192.168.0.100
  ```  
- 또는 수동 방법  
  1. 공개키 파일 전송  
     ```bash
     scp ~/.ssh/my-ubuntu.pem.pub user@192.168.0.100:~/tempkey.pub
     ```
  2. 우분투 서버에서 등록  
     ```bash
     ssh user@192.168.0.100
     mkdir -p ~/.ssh
     cat ~/tempkey.pub >> ~/.ssh/authorized_keys
     chmod 700 ~/.ssh
     chmod 600 ~/.ssh/authorized_keys
     rm ~/tempkey.pub
     ```

-------------------------------------------------

## 4. 공유기 설정

### 4-1. 포트포워딩  
- 공유기 관리자 페이지에서 포트포워딩 설정  
  - 외부 포트: xxxx  
  - 내부 포트: xxxx  
  - 내부 IP: (우분투 서버의 고정 IP, 예: 192.168.0.100)  
  - 프로토콜: TCP

-------------------------------------------------

## 5. 접속 테스트 및 ssh_config 간소화 (맥)

### 5-1. 개별 접속 테스트  
- 아래 명령어로 접속  
  ```bash
  chmod 600 ~/.ssh/my-ubuntu.pem
  ssh -i ~/.ssh/my-ubuntu.pem -p xxxx user@myserver.duckdns.org
  ```

### 5-2. 접속 명령 간소화를 위한 SSH 클라이언트 설정  
- `~/.ssh/config` 파일 수정  
  ```bash
  nano ~/.ssh/config
  ```  
- 아래 내용 추가  
  ```
  Host myserver
    HostName myserver.duckdns.org
    User user
    Port xxxx
    IdentityFile ~/.ssh/my-ubuntu.pem
  ```  
- 이후, 간단히 다음 명령어로 접속 가능  
  ```bash
  ssh myserver
  ```

-------------------------------------------------

## 6. 추가 보안 권장 사항 (우분투)

- fail2ban 설치  
  ```bash
  sudo apt install fail2ban
  ```
- 정기 시스템 업데이트  
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

-------------------------------------------------

설정 완료 후 맥에서 다음 명령어로 서버 접속을 확인할 수 있습니다.
```bash
ssh myserver
```
