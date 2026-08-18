---
title: "[BBB] BeagleBone Black 로컬 웹 IDE로 보드 제어"
description: "BeagleBone Black Embedded #4"
date: 2026-08-30
update: 2026-08-30
tags:
  - beagleboneblack
  - arm
  - cortex-a8
  - uart
  - ftdi
series: "BeagleBone Black 보드로 산업용 임베디드 입문"
---

## 🚀 들어가며

아두이노, 라즈베리파이, 비글본블랙 등 소형 보드로 임베디드에 입문하면 가장 먼저 여러 센서를 달아 아날로그와 디지털 입출력을 살펴보게 된다. 이때 사용하는 것이 각종 핀들인데, 각 핀이 어떤 기능을 하는지 알아야 올바른 방법으로 연결할 수 있다.


https://docs.beagleboard.org/books/beaglebone-cookbook/01basics/basics.html





<aside>
💡

RTEMS를 Source Builder로부터 설치해 봅니다.

</aside>

**사전 준비**

---

- [ ]  Google Cloud Console Virtual Machine

**학습 목표**

---

- [ ]  RTEMS 6.1 설치 (2시간 이상 소요)
- [ ]  예제 소스코드 살펴보기

**자료실**

---

https://docs.rtems.org/docs/main/user/start/preparation.html

https://docs.rtems.org/docs/main/user/start/prefixes.html

https://blog.thelunatic.dev/getting-started-bbb-1/

https://blog.thelunatic.dev/getting-started-bbb-2/

---

## RTEMS 설치 방법

### 1. Obtain the Sources

```bash
mkdir /opt/rtems
cd /opt/rtems

git clone https://gitlab.rtems.org/rtems/tools/rtems-source-builder.git
git clone https://gitlab.rtems.org/rtems/rtos/rtems.git
git clone https://gitlab.rtems.org/rtems/pkg/rtems-libbsd.git
```

### 2. Build and install the tools

```bash
cd /opt/rtems/rtems-source-builder/rtems
git checkout 6.1
../source-builder/sb-set-builder --prefix="/opt/rtems/6.1" 6/rtems-arm
```

### 3. Build and install the RTEMS Board Support Packages

```bash
cd /opt/rtems/rtems
git checkout 6.1

# The official command
echo -e "[arm/beagleboneblack]" > config.ini

# In our case
sudo tee config.ini << EOF
[arm/beagleboneblack]
RTEMS_POSIX_API = True
EOF

./waf configure --prefix "/opt/rtems/6.1"
./waf
./waf install
```

### 4. Populate the `rtems_waf` git submodule

```bash
cd /opt/rtems/rtems-libbsd
git checkout 6.1
git submodule init
git submodule update rtems_waf
```

### 5. Run Waf's configure to build libbsd

```bash
cd /opt/rtems/rtems-libbsd
./waf configure --prefix="/opt/rtems/6" \
    --rtems-bsps=arm/beagleboneblack \
    --buildset=buildset/default.ini
./waf
./waf install
```

## Sample source code directory

```bash
# Pre-configured executable files
/opt/rtems/rtems/build/arm/beagleboneblack/testsuites/samples/hello.exe

# Source code to sample apps
/opt/rtems/rtems/testsuites/samples/hello/init.c

# Copy sample directory to ${HOME}
cp -r /opt/rtems/rtems/build/arm/beagleboneblack/testsuites/samples/ ~/rtems-samples
```





































## ✨ 마치며
