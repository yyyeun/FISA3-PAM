# Linux PAM(Pluggable Authentication Modules) Configuration

## ✋ PAM이란?
> **PAM**은 리눅스 및 유닉스 계열 운영체제에서 **인증(authentication) 시스템**을 모듈화하고 유연하게 구성할 수 있는 시스템으로, **다양한 인증 방식(예: 비밀번호, 생체 인증, 토큰 등)을 쉽게 추가하거나 변경할 수 있도록 도와줍니다.**

<br>

## 🏆 학습 목표
PAM 리눅스 시스템에서 사용자의 인증을 담당하는 모듈을 사용하여 비밀번호를 8자리 이상으로 규제하기

<br>

## 🔧 VM 환경 설정
### 1. myserver01 복제
- VM 복제 > MAC 주소 정책 : 모든 네트워크 어댑터의 새 MAC 주소 생성 선택
  ![image](https://github.com/user-attachments/assets/77ddb923-d3af-4f50-a588-6d64abccd3b4)
- 완전한 복제 선택
  ![image](https://github.com/user-attachments/assets/a4b121d4-d944-4438-9b5e-984254db9f8e)

### 2. IP 고정 (10.0.2.21)
- IP 주소 확인
  ```
  $ifconfig
  ```
  ![image](https://github.com/user-attachments/assets/ebb304ae-8e6b-4c64-b601-6ea586dc9c58)

  - MAC 주소를 변경했으나 IP 주소가 그대로인 상황
  - IP 주소를 고정함으로써 직접 변경

- IP 고정
  ```
  $sudo vi /etc/netplan/00-installer-config.yaml
  $sudo netplan apply
  ```
  ```
  # /etc/netplan/00-installer-config.yaml
  
  network:
    version: 2
    renderer: networkd
    ethernets:
      enp0s3:
        addresses:
          - 10.0.2.21/24  # 변경된 고정 IP 주소
        routes:
          - to: default
            via: 10.0.2.1  # 게이트웨이
        nameservers:
          addresses:
            - 8.8.8.8
        dhcp4: false
  ```
  ![image](https://github.com/user-attachments/assets/172144a0-0154-4ed5-9178-a4309d23f129)
  - 변경된 IP 주소 확인

### 3. NatNetwork 설정
- VM 네트워크 > NatNetwork 설정
  ![image](https://github.com/user-attachments/assets/533a0b94-41aa-4649-890e-24aff6a3d066)
- 도구 > NatNetwork에서 포트포워딩 설정
  ![image](https://github.com/user-attachments/assets/a1dcb828-9422-4dd6-bfed-ce202589cf8c)

### 4. MobaXterm에서 세션 재설정 후 접속
- 포트 번호 VM에서 설정한 번호로 설정 후 접속
  ![image](https://github.com/user-attachments/assets/854887d1-9614-46fa-9a00-b7eb0f0833a7)

<br>

## 🔐 PAM 비밀번호 규제 설정 
### 1. pam_pwquality 모듈 설치
```
$sudo apt install libpam-pwquality
```

### 2. PAM 설정 파일 수정
```
$ls /etc/pam.d
chfn      common-account   common-session                 login     passwd    runuser-l  sudo    vmtoolsd
chpasswd  common-auth      common-session-noninteractive  newusers  polkit-1  sshd       sudo-i
chsh      common-password  cron                           other     runuser   su         su-l
```
- `/etc/pam.d` : PAM(Pluggable Authentication Modules) 설정 파일들이 저장되어 있는 디렉터리
- `/etc/pam.d/common-password` : 비밀번호 정책을 설정하는 파일
  - 이 파일에서 pam_pwquality.so 라인의 설정을 수정하거나 추가해야 함

```
$sudo vi /etc/pam.d/common-password
```
```
#
# /etc/pam.d/common-password - password-related modules common to all services
#
# This file is included from other service-specific PAM config files,
# and should contain a list of modules that define the services to be
# used to change user passwords.  The default is pam_unix.

# Explanation of pam_unix options:
# The "yescrypt" option enables
#hashed passwords using the yescrypt algorithm, introduced in Debian
#11.  Without this option, the default is Unix crypt.  Prior releases
#used the option "sha512"; if a shadow password hash will be shared
#between Debian 11 and older releases replace "yescrypt" with "sha512"
#for compatibility .  The "obscure" option replaces the old
#`OBSCURE_CHECKS_ENAB' option in login.defs.  See the pam_unix manpage
#for other options.

# As of pam 1.0.1-6, this file is managed by pam-auth-update by default.
# To take advantage of this, it is recommended that you configure any
# local modules either before or after the default block, and use
# pam-auth-update to manage selection of other modules.  See
# pam-auth-update(8) for details.

# here are the per-package modules (the "Primary" block)
password        requisite                       pam_pwquality.so retry=3 minlen=8
password        [success=1 default=ignore]      pam_unix.so obscure use_authtok try_first_pass yescrypt
# here's the fallback if no module succeeds
password        requisite                       pam_deny.so
```
- `retry=3`: 사용자가 비밀번호 입력에 실패했을 때 다시 시도할 수 있는 횟수
- `minlen=8`: 비밀번호의 최소 길이를 8자리로 설정

### 3. 설정 적용 확인
```
$passwd
Changing password for username.
Current password:
New password:
BAD PASSWORD: The password fails the dictionary check - it is too simplistic/systematic
New password:
BAD PASSWORD: The password fails the dictionary check - it is based on a dictionary word
New password:
Retype new password:
passwd: password updated successfully
```
- 현재 로그인한 계정의 비밀번호를 재설정
  - 단순한 비밀번호를 설정했을 경우 : `it is too simplistic/systematic`
  - 비밀번호에 단어가 포함된 경우 : `it is based on a dictionary word`
  - 조건에 맞게 비밀번호를 설정한 경우 : `passwd: password updated successfully`
- 사용자를 지정해 비밀번호를 변경할 경우
  - passwd (username) : username에 해당하는 사용자의 비밀번호를 변경
