# Ansible - NTP & Mail Server Setup 1.0.3

## 📌 프로젝트 개요
이 프로젝트는 Ansible을 사용하여 `ansible3` 서버에 **NTP 서버(chrony)** 와 **Mail 서버(Postfix)** 를 자동으로 설치하고 설정하기 위한 Playbook입니다.  
내부 네트워크 환경에서 **시간 동기화(NTP)** 와 **메일 서비스(MTA: Postfix)** 를 안정적으로 운영하기 위해 작성되었습니다.  

특히 **DNS 서버와 유기적으로 연동**하여 NTP 및 Mail 서버를 도메인 기반으로 접근할 수 있게 구성합니다.

---

## 🖥️ 환경 정보
- **Ansible 서버**: 192.168.10.10 (`ansible`)
- **대상 서버(NTP & Mail 서버)**: 192.168.10.13 (`ansible3`)
- **운영체제**: CentOS Stream 9 / RHEL 9 계열
- **NTP 패키지**: chrony
- **Mail 패키지**: postfix, mailx
- **내부 DNS 도메인**: `psw.com`
  - NTP 서버 도메인: `ntp.psw.com`
  - Mail 서버 도메인: `mail.psw.com`

---

## ⚙️ NTP와 DNS의 유기적 관계
- NTP 클라이언트는 IP 주소 대신 `ntp.psw.com` 으로 동기화 가능.
- IP 변경 시 DNS 레코드만 수정하면 되므로 관리가 간단.
- 다수의 NTP 서버를 운영할 경우 DNS 라운드로빈으로 부하분산 가능.

DNS 레코드 예시:
```
ntp    IN   A   192.168.10.13
```

---

## ⚙️ Mail과 DNS의 유기적 관계
- 메일 서버는 **SMTP 프로토콜(25번 포트)** 로 송수신을 수행합니다.
- DNS의 **MX 레코드**가 메일 서비스의 핵심 역할을 담당합니다.
- 클라이언트는 `user@mail.psw.com` 같은 주소로 메일을 송수신하며, 내부 DNS가 이를 `192.168.10.13`으로 해석합니다.
- 외부 도메인과 연동하려면 **SPF, DKIM, DMARC** 같은 DNS 레코드도 추가할 수 있습니다.

DNS 레코드 예시:
```
mail    IN   A    192.168.10.13
@       IN   MX   10 mail.psw.com.
```

---

## ⚙️ 사전 준비
1. Ansible 서버에서 대상 서버로 SSH 접속 가능해야 합니다.
   ```bash
   ssh-copy-id ansible@192.168.10.13
   ```

2. 내부 DNS에 `ntp.psw.com`, `mail.psw.com` 레코드 등록.

3. `inventory` 파일에 대상 서버 추가:
   ```ini
   [ntp]
   ansible3 ansible_host=192.168.10.13

   [mail]
   ansible3 ansible_host=192.168.10.13
   ```

---

## 🚀 Playbook 실행
1. **NTP 서버 설치**
   ```bash
   ansible-playbook -i inventory ntp.yml
   ```

   주요 작업:
   - chrony 패키지 설치
   - `/etc/chrony.conf` 템플릿 배포
   - `server ntp.psw.com iburst` 설정
   - chronyd 서비스 enable & start
   - 방화벽에서 UDP 123 허용

2. **Mail 서버 설치**
   ```bash
   ansible-playbook -i inventory mail.yml
   ```

   주요 작업:
   - postfix, mailx 패키지 설치
   - `/etc/postfix/main.cf` 설정 (도메인 `psw.com` 반영)
   - TLS 인증서 생성 (self-signed)
   - postfix 서비스 enable & start
   - 방화벽에서 TCP 25 허용

---

## ✅ 동작 확인
### NTP
```bash
systemctl status chronyd
chronyc sources -v
dig ntp.psw.com
```

### Mail
```bash
systemctl status postfix
echo "Test Mail" | mail -s "Hello" user@psw.com
```

---

## 📂 파일 구조
```
├── inventory
├── ntp.yml
├── mail.yml
├── roles/
│   ├── ntp/
│   │   ├── tasks/main.yml
│   │   ├── templates/chrony.conf.j2
│   │   └── handlers/main.yml
│   └── mail/
│       ├── tasks/main.yml
│       ├── templates/main.cf.j2
│       └── handlers/main.yml
└── README.md
```

---

## ✨ 참고
- NTP 서버는 UDP 123, Mail 서버는 TCP 25 포트를 사용합니다.
- DNS 서버와의 통합은 시스템 관리의 핵심이며, IP 기반보다 **가독성**과 **확장성**이 뛰어납니다.
- 메일 서비스는 보안을 위해 TLS/SSL 인증서와 스팸 방지 레코드(SPF, DKIM, DMARC) 구성을 권장합니다.
