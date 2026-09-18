---
title: "[RTEMS] RTEMS 시작점 init"
description: "RTEMS Open Source #2"
date: 2026-10-30
update: 2026-10-30
tags:
  - rtos
  - rtems
  - beagleboneblack
series: "위성/우주 프로젝트에서 널리 쓰이는 RTEMS"
---

## 🚀 들어가며

![Real-Time Executive for Multiprocessor Systems](sthr.jpg)

RTOS를 처음 시작할 때 FreeRTOS로 입문하는 경우가 가장 많을 것이다. 가볍고 오픈소스이기 때문인데, RTEMS 역시 오픈소스 RTOS이며 NASA 및 ESA 임무에서 오랫동안 쓰여온 헤리티지를 바탕으로 우주항공 분야에서 꾸준히 활용되고 있다.

## 🗿 모노리식 링크형 RTOS

보통의 범용 OS는 먼저 PC 또는 보드에 리눅스 등의 운영체제를 올리고 그 위에서 각종 작업을 한다. 그러나 RTOS는 빌드 실행 파일 안에 OS가 함께 링크되어, 보드에 해당 실행 파일을 올리면 OS와 내부 코드가 함께 돌아가는 구조이다.

별도의 OS 설치 단계가 존재하지 않고, 빌드 자체가 OS 이미지 생성 과정이다.

### RTEMS의 `Init` 함수

따라서 모든 RTOS는 C언어의 `main` 함수와 같은 운영체제 진입 함수를 갖는다. RTEMS의 경우 `Init`이라는 이름의 진입 함수가 있다.

```c
/*
 * Hello world example
 */
#include <rtems.h>
#include <stdlib.h>
#include <stdio.h>

rtems_task Init(
    rtems_task_argument ignored
)
{
    printf( "\nHello World\n" );
    exit( 0 );
}
```

이것이 빌드 결과물 실행 파일의 시작점이자 운영 체제의 시작점이다. 이 소스를 컴파일하여 실행 파일을 만들고 실행하면, 운영체제가 시작되고, `Hello World`를 출력하고, 운영체제가 종료된다.

    RTEMS Testing - Run, @rtems-ver-mjminrev@
    Command Line: $BASE/quick-start/rtems/7/bin/rtems-run --rtems-bsps=erc32-sis build/sparc-rtems7-erc32/hello.exe
    Host: Linux  5.8.0-44-generic #50~20.04.1-Ubuntu SMP Wed Feb 10 21:07:30 UTC 2021 x86_64
    Python: 3.8.5 (default, Jan 27 2021, 15:41:15) [GCC 9.3.0]
    Host: Linux-5.8.0-44-generic-x86_64-with-glibc2.29 (Linux 5.8.0-44-generic #50~20.04.1-Ubuntu SMP Wed Feb 10 21:07:30 UTC 2021 x86_64 x86_64)

    SIS - SPARC/RISCV instruction simulator 2.26,  copyright Jiri Gaisler 2020
    Bug-reports to jiri@gaisler.se

    ERC32 emulation enabled

    Loaded build/sparc-rtems7-erc32/hello.exe, entry 0x02000000

    Hello World

    *** FATAL ***
    fatal source: 5 (RTEMS_FATAL_SOURCE_EXIT)
    fatal code: 0 (0x00000000)
    RTEMS version: 7.0.0.586e06ec6222f1cd1f005aa8f4a34a8b33f5d862
    RTEMS tools: 10.2.1 20210309 (RTEMS 7, RSB 5e449fb5c2cb6812a238f9f9764fd339cbbf05c2, Newlib d10d0d9)
    executing thread ID: 0x08a010001
    executing thread name: UI1
    cpu 0 in error mode (tt = 0x101)
    158479  0200d500:  91d02000   ta  0x0
    Run time     : 0:00:00.259136

## RTEMS Shell

한 실행파일에 운영체제가 다 들어있어서..? 보통 운영체제의 구성인 셸 파일시스템 등을 직접 구성해줘야 한다. 다 구현해야 하는건 아니고 매크로를 정의하면 빌드때 포함됨.원하는 거를 넣거나 뺄 수 잇다.

rtems에서 셸은 어쩌구 매크로를 정의해서 구성할수있다.


rtos는 




















## ✨ 마치며
