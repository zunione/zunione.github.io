---
title: "[NeoVim] 플러그인 설치 중 인터넷 연결 유실"
description: "NeoVim configuration #1"
date: 2026-10-29
update: 2025-10-29
tags:
  - neovim
  - ide
series: "NeoVim으로 개발환경 구축하기"
---

# WSL2 인터넷 연결 오류 해결: "Could not resolve host" 완벽 가이드

## 🚨 문제 상황

WSL2(Windows Subsystem for Linux 2)에서 `git clone`을 시도했을 때 다음과 같은 오류가 발생했습니다:

```bash
zunione@101-24031:~$ git clone https://github.com/zunione/nvim
Cloning into 'nvim'...
fatal: unable to access 'https://github.com/zunione/nvim/': Could not resolve host: github.com
```

`apt install`도 불가능하고, 완전히 인터넷 연결이 차단된 상태였습니다.

## 🔍 진단 과정

### 1단계: 기본 네트워크 상태 확인

```bash
# IP 주소 확인
ip addr show

# 라우팅 테이블 확인
ip route show

# DNS 설정 확인
cat /etc/resolv.conf

# 게이트웨이 핑 테스트
ping -c 4 8.8.8.8
```

**진단 결과:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 172.23.214.54/20 brd 172.23.223.255 scope global eth0
    
default via 172.23.208.1 dev eth0 proto kernel
172.23.208.0/20 dev eth0 proto kernel scope link src 172.23.214.54

nameserver 127.0.0.53

PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3064ms
```

**발견된 문제점:**
- ❌ 외부 인터넷 연결 실패 (100% 패킷 손실)
- ❌ 게이트웨이 `172.23.208.1`에 도달 불가

### 2단계: 게이트웨이 연결 테스트

```bash
ping -c 4 172.23.208.1
```

**결과:** 게이트웨이 자체가 응답하지 않음 → WSL 네트워크 브릿지 문제 확정

### 3단계: Windows 네트워크 상태 확인

Windows PowerShell에서 `ipconfig` 실행:

```
IPv4 주소: 10.10.10.34
서브넷 마스크: 255.255.255.0
기본 게이트웨이: 172.25.2.254
```

**결정적 단서 발견:**
- Windows PC: `10.10.10.34` (게이트웨이: `172.25.2.254`)
- WSL: `172.23.214.54` (게이트웨이: `172.23.208.1`)

**→ WSL이 자체 가상 네트워크에 격리되어 Windows의 실제 네트워크에 접근하지 못하고 있음**

## 🎯 근본 원인 분석

### 문제 1: WSL2 네트워크 격리

WSL2는 기본적으로 **NAT(Network Address Translation)** 방식으로 동작하며, Hyper-V를 통해 자체 가상 네트워크를 생성합니다. 이 과정에서 Windows 네트워크와의 브릿지가 제대로 구성되지 않으면 WSL이 외부 인터넷에 접근할 수 없게 됩니다.

**주요 원인:**
- Hyper-V Virtual Ethernet Adapter 오류
- Windows 네트워크 스택 충돌
- VPN 소프트웨어 간섭
- 방화벽/보안 소프트웨어 차단

### 문제 2: DNS 해석 실패

WSL2는 `systemd-resolved`를 통해 DNS를 관리하는데, `/etc/resolv.conf`가 stub resolver(`127.0.0.53`)를 가리키고 있었습니다. 네트워크 브릿지가 끊어진 상태에서는 이 stub resolver가 실제 DNS 서버로 쿼리를 전달할 수 없습니다.

## ✅ 해결 방법

### Step 1: Windows 네트워크 스택 재설정

**Windows PowerShell (관리자 권한)에서 실행:**

```powershell
# Winsock 카탈로그 재설정
netsh winsock reset

# TCP/IP 스택 재설정
netsh int ip reset
```

실행 후 Windows 재부팅이 필요합니다.

### Step 2: WSL2 네트워크 모드 변경

**Windows에서 `.wslconfig` 파일 생성/수정:**

```powershell
notepad $env:USERPROFILE\.wslconfig
```

다음 내용 추가:

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
firewall=true
autoProxy=true
```

**각 옵션 설명:**
- `networkingMode=mirrored`: WSL이 Windows 네트워크를 직접 미러링하여 사용
- `dnsTunneling=true`: DNS 쿼리를 Windows DNS로 터널링
- `firewall=true`: Windows 방화벽 규칙 적용
- `autoProxy=true`: Windows 프록시 설정 자동 적용

저장 후 WSL 재시작:

```powershell
wsl --shutdown
```

### Step 3: WSL에서 네트워크 상태 확인

WSL을 다시 시작한 후:

```bash
route -n
```

**변경 후 라우팅 테이블:**

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.25.2.254    0.0.0.0         UG    291    0        0 eth0
10.10.10.0      0.0.0.0         255.255.255.0   U     291    0        0 eth0
172.25.2.0      0.0.0.0         255.255.255.0   U     291    0        0 eth0
172.25.2.254    0.0.0.0         255.255.255.255 UH    291    0        0 eth0
```

✅ 이제 Windows의 실제 게이트웨이 `172.25.2.254`를 사용하고 있습니다!

### Step 4: DNS 문제 해결

네트워크 모드가 변경되었지만 여전히 DNS 해석이 안 될 수 있습니다:

```bash
curl -I https://github.com
# curl: (6) Could not resolve host: github.com
```

**원인:** `/etc/resolv.conf`가 여전히 systemd-resolved의 stub resolver를 가리키고 있음

**해결:**

```bash
# 1. WSL 자동 DNS 설정 비활성화
sudo nano /etc/wsl.conf
```

다음 내용 추가:

```ini
[network]
generateResolvConf = false
```

```bash
# 2. 기존 resolv.conf 삭제 (심볼릭 링크)
sudo rm /etc/resolv.conf

# 3. 새로운 DNS 설정 파일 생성
sudo bash -c 'cat > /etc/resolv.conf << EOF
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 1.1.1.1
EOF'
```

**DNS 서버 설명:**
- `8.8.8.8` / `8.8.4.4`: Google Public DNS
- `1.1.1.1`: Cloudflare DNS

### Step 5: 연결 확인

```bash
# DNS 해석 테스트
curl -I https://github.com

# 성공 시 HTTP/2 200 응답
HTTP/2 200
date: Tue, 28 Oct 2025 00:04:38 GMT
content-type: text/html; charset=utf-8
...

# git clone 테스트
git clone https://github.com/username/repository

# 패키지 설치 테스트
sudo apt update
sudo apt install net-tools
```

## 🔧 추가 트러블슈팅

### 게이트웨이 Ping이 실패하는 경우

많은 라우터는 보안상 ICMP ping을 차단합니다. 따라서 게이트웨이 ping이 실패하더라도 실제 인터넷 연결은 정상일 수 있습니다.

```bash
# ping 대신 실제 연결 테스트
curl -I https://google.com
wget --spider https://google.com
```

### VPN 사용 시

일부 VPN 소프트웨어가 WSL 네트워킹을 방해할 수 있습니다:

1. VPN 연결 해제 후 테스트
2. VPN 설정에서 "로컬 네트워크 허용" 옵션 활성화
3. WSL 트래픽을 VPN 터널에서 제외

### Hyper-V 가상 스위치 재생성

극단적인 경우 가상 스위치를 수동으로 재생성:

```powershell
# PowerShell (관리자)
wsl --shutdown
Get-VMSwitch
Remove-VMSwitch "WSL" -Force
```

WSL 재시작 시 자동으로 가상 스위치가 재생성됩니다.

## 📋 완전한 해결 체크리스트

1. ✅ Windows 네트워크 스택 재설정
   ```powershell
   netsh winsock reset
   netsh int ip reset
   ```

2. ✅ `.wslconfig`에 mirrored 네트워크 모드 설정
   ```ini
   [wsl2]
   networkingMode=mirrored
   ```

3. ✅ Windows 재부팅

4. ✅ `/etc/wsl.conf`에서 자동 DNS 비활성화
   ```ini
   [network]
   generateResolvConf = false
   ```

5. ✅ `/etc/resolv.conf` 수동 설정
   ```bash
   sudo rm /etc/resolv.conf
   sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
   ```

6. ✅ WSL 재시작 및 연결 테스트
   ```powershell
   wsl --shutdown
   ```

## 🎓 핵심 요약

**문제의 본질:**
- WSL2가 자체 가상 네트워크에 격리되어 Windows 호스트 네트워크와 브릿지 실패
- systemd-resolved의 DNS stub resolver가 실제 DNS 서버와 통신 불가

**해결의 핵심:**
1. **Mirrored 네트워크 모드**: WSL이 Windows 네트워크를 직접 사용하도록 변경
2. **수동 DNS 설정**: systemd-resolved 대신 직접 DNS 서버 지정

**예방 방법:**
- `.wslconfig`와 `/etc/wsl.conf`를 미리 설정
- VPN 사용 시 WSL 호환성 확인
- Windows 네트워크 변경 시 WSL 재시작

## 💡 참고 자료

- [Microsoft WSL Documentation](https://learn.microsoft.com/en-us/windows/wsl/)
- [WSL2 Networking](https://learn.microsoft.com/en-us/windows/wsl/networking)
- [systemd-resolved](https://www.freedesktop.org/software/systemd/man/systemd-resolved.service.html)

---

이 가이드로 동일한 문제를 겪는 분들에게 도움이 되길 바랍니다! 🚀