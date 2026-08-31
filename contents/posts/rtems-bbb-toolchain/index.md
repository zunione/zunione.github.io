---
title: "[BBB/RTEMS] RTEMS를 BBB BSP로 빌드하고 부팅하기"
description: "BeagleBone Black Embedded #4"
date: 2026-09-30
update: 2026-09-30
tags:
  - rtos
  - rtems
  - beagleboneblack
series: "BeagleBone Black 보드로 산업용 임베디드 입문"
---

## 🚀 들어가며

![Real-Time Executive for Multiprocessor Systems](sthr.jpg)

RTOS를 처음 시작할 때 FreeRTOS로 입문하는 경우가 가장 많을 것이다. 가볍고 오픈소스이기 때문인데, RTEMS 역시 오픈소스 RTOS이며 NASA 및 ESA 임무에서 오랫동안 쓰여온 헤리티지를 바탕으로 우주항공 분야에서 꾸준히 활용되고 있다.

RTEMS는 다양한 아키텍처와 보드를 지원하며, 각 보드에 맞는 BSP(Board Support Package)를 선택해서 빌드하게 된다. 예를 들어 x86 아키텍처의 pc686, ARM의 Raspberry Pi, SPARC의 leon3 등이 있고, 당연히 비글본 시리즈 보드를 위한 BSP도 준비되어 있다.

단점이라면 보드가 바뀌면 장장 두 시간이 걸리는 빌드 과정을 또 거쳐야 한다는 점.. 정도가 되겠다. 그러나 이 부분은 하드웨어와 긴밀한 상호작용을 해야 하는 RTOS 특성상 어쩔 수 없는 부분인 것 같다.

**참고자료**

- [RTEMS Quick Start: Preparation](https://docs.rtems.org/docs/main/user/start/preparation.html)
- [Getting started with RTEMS on Beaglebone black - part I](https://blog.thelunatic.dev/getting-started-bbb-1/)
- [Getting started with RTEMS on Beaglebone black - part II](https://blog.thelunatic.dev/getting-started-bbb-2/)

## ⏱️ RTEMS 및 추가 패키지 설치

### 1. RTEMS 소스 트리 준비

RTEMS 빌드를 위해서는 RTEMS Source Builder(RSB)와 본체 RTEMS 소스코드가 필요하다. RSB는 크로스 컴파일 툴체인을 소스부터 빌드해주는 도구이고, rtems는 커널과 각 보드용 BSP 소스가 들어 있는 메인 저장소이다.

또한 네트워크 스택이나 디바이스 드라이버 같은 추가 기능을 쓰기 위해 rtems-libbsd도 함께 받는다. rtems-libbsd는 FreeBSD에서 포팅한 네트워크 스택·디바이스 드라이버 등을 RTEMS에서 쓸 수 있게 해주는 라이브러리이다.

```bash
mkdir /opt/rtems
cd /opt/rtems

git clone https://gitlab.rtems.org/rtems/tools/rtems-source-builder.git
git clone https://gitlab.rtems.org/rtems/rtos/rtems.git
git clone https://gitlab.rtems.org/rtems/pkg/rtems-libbsd.git
```

### 2. RSB로 크로스 컴파일 툴체인 빌드

ARM 타겟용 크로스 컴파일 툴체인을 빌드한다. 이제부터는 꼭 6.1 버전 브랜치로 체크아웃해 버전을 맞춰 주어야 한다.

빌드 결과물은 `/opt/rtems/6.1`에 설치된다.

```bash
cd /opt/rtems/rtems-source-builder/rtems
git checkout 6.1
../source-builder/sb-set-builder --prefix="/opt/rtems/6.1" 6/rtems-arm
```

### 3. BeagleBone Black BSP로 RTEMS 설치

다음은 실제 RTEMS를 설치하는 과정인데, 이때 BSP를 지정해 줌과 동시에 POSIX API를 활성화해야 한다. POSIX 없이 소스를 빌드하면 나중에 libBSD 설치 과정에서 에러가 발생한다.

```bash
export PATH=/opt/rtems/6.1/bin:"$PATH"

cd /opt/rtems/rtems
git checkout 6.1

# The official command
#echo -e "[arm/beagleboneblack]" > config.ini

# In our case
sudo tee config.ini << EOF
[arm/beagleboneblack]
RTEMS_POSIX_API = True
EOF
```

RTEMS 6 계열부터는 Python 기반의 Waf라는 빌드 시스템을 사용한다. 이전 RTEMS 5.x까지는 GNU autoconf/automake 기반이었기 때문에 bootstrap 스크립트를 별도로 실행해야 했는데, 이제는 간편하게 여러 타겟 아키텍처를 빌드할 수 있게 되었다.

Waf는 단일 Python 스크립트로 배포되는 빌드 시스템으로, 프로젝트 저장소에 직접 포함시켜 별도 설치 없이 바로 사용할 수 있다. RTEMS도 마찬가지로 waf 스크립트가 루트 디렉터리에 이미 존재한다.

```bash

./waf configure --prefix "/opt/rtems/6.1"
./waf
./waf install
```

### 4. libBSD 라이브러리 빌드

libBSD도 마찬가지로 6.1 버전으로 체크아웃하고, Waf 업데이트까지 진행해 준다.

```bash
cd /opt/rtems/rtems-libbsd
git checkout 6.1
git submodule init
git submodule update rtems_waf
```

올바른 BSP와 설정 파일을 사용해 빌드한다.

```bash
./waf configure --prefix="/opt/rtems/6" \
    --rtems-bsps=arm/beagleboneblack \
    --buildset=buildset/default.ini
./waf
./waf install
```

## RTEMS 부팅 파일 구성

### RTEMS 커널 이미지

```bash
# Pre-configured executable files
/opt/rtems/rtems/build/arm/beagleboneblack/testsuites/samples/hello.exe

# Source code to sample apps
/opt/rtems/rtems/testsuites/samples/hello/init.c

# Copy sample directory to ${HOME}
cp -r /opt/rtems/rtems/build/arm/beagleboneblack/testsuites/samples/ ~/rtems-samples
```


```bash
arm-rtems5-objcopy hello.exe -O binary app.bin
gzip -9 app.bin
mkimage -A arm -O linux -T kernel -a 0x80000000 -e 0x80000000 -n RTEMS -d app.bin.gz rtems-app.img
```


### `uEnv.txt`



```bash
setenv bootdelay 5
uenvcmd=run boot
boot=fatload mmc 0 0x80800000 rtems-app.img ; fatload mmc 0 0x88000000 am335x-boneblack.dtb ; bootm 0x80800000 - 0x88000000
```




### device tree blob (dtb) 

```bash
git clone https://github.com/freebsd/freebsd.git
MACHINE='arm' freebsd/sys/tools/fdt/make_dtb.sh \
freebsd/sys \
freebsd/sys/gnu/dts/arm/am335x-boneblack.dts \
$(pwd)
```
































## ✨ 마치며
