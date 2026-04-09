---
title: "[cFS] CFS-101(7): SCH 추가과제"
description: "Learning cFS framework #8"
date: 2026-04-09
update: 2026-04-09
tags:
  - nasa-cfs
  - fsw
series: "NASA cFS로 우주 소프트웨어 개발자 되기"
---

## 🚀 들어가며

CFS-101 트레이닝의 마지막 파트에서는 SCH(Scheduler) 애플리케이션의 동작 주파수를 기본 100Hz에서 40Hz로 변경하는 실습을 혼자 진행할 것을 권유한다. 안 하고 넘어가면 섭하니, 유종의 미를 거두기 위해 마지막 실습까지 꼼꼼히 짚고 넘어가려 한다.

해당 실습 외에도 `/home/dev/Training_workspace/CFS-101/apps/sch/docs` 디렉토리에 SCH 앱에 관한 상세한 설명이 준비되어 있으니, 정독하는 것을 권장한다.

## 🎰 슬롯 수 변경: `sch_platform_cfg.h`

`SCH_TOTAL_SLOTS` 값을 100에서 40으로 변경한다.

```c
/* 변경 전 */
#define SCH_TOTAL_SLOTS         100    /* SCH wake-up rate (Hz) */

/* 변경 후 */
#define SCH_TOTAL_SLOTS         40     /* SCH wake-up rate (Hz) */
```

이 값이 바뀌면 `SCH_TABLE_ENTRIES`도 자동으로 재계산된다.

```c
#define SCH_TABLE_ENTRIES       (SCH_TOTAL_SLOTS * SCH_ENTRIES_PER_SLOT)
```

## 🪜 스케줄 테이블 크기 조정: `sch_def_schtbl.c`

`SCH_TABLE_ENTRIES` 매크로를 사용하는 구조체로 `SCH_DefaultScheduleTable`이 있다.

```c
SCH_ScheduleEntry_t SCH_DefaultScheduleTable[SCH_TABLE_ENTRIES]
```

이 테이블은 `SCH_TABLE_ENTRIES` 크기의 배열이므로, 슬롯 #0~#39만 남기고 #40~#99를 삭제해야 한다. 문제는 슬롯 #40 이후에 활성 엔트리가 존재한다는 점이다.

기존 테이블에서 #40 이후에 있던 활성 엔트리 3개를 #0~#39 범위의 빈 슬롯으로 이동시켜야 한다.

| 원래 슬롯 | 내용 | 이동할 슬롯 |
|---|---|---|
| #43 | ES HK Request | → #4 |
| #49 | SCH HK Request | → #6 |
| #56 | CI HK Request + TO HK Request | → #8 |

#### 슬롯 #4 (← 기존 #43의 ES HK)

```c
  /* slot #4 */
  {  SCH_ENABLED,  SCH_ACTIVITY_SEND_MSG,  4,  3,  CFE_ES_SEND_HK_MIDX, SCH_GROUP_CFE_HK },   /* ES HK Request */
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
```

#### 슬롯 #6 (← 기존 #49의 SCH HK)

```c
  /* slot #6 */
  {  SCH_ENABLED,  SCH_ACTIVITY_SEND_MSG,  4,  2, SCH_SEND_HK_MIDX, SCH_GROUP_CFS_HK },   /* SCH HK Request */
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
```

#### 슬롯 #8 (← 기존 #56의 CI/TO HK)

```c
  /* slot #8 */
  {  SCH_ENABLED,  SCH_ACTIVITY_SEND_MSG,  4,  1, CI_SEND_HK_MIDX, SCH_GROUP_NONE },   /* CI HK Request */
  {  SCH_ENABLED,  SCH_ACTIVITY_SEND_MSG,  4,  2, TO_SEND_HK_MIDX, SCH_GROUP_NONE },   /* TO HK Request */
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
  {  SCH_UNUSED,   0,      0,  0, 0,  SCH_GROUP_NONE},
```

이동할 슬롯은 기존에 비어있던 곳 중 아무 곳이나 골라도 된다. 다만 한 슬롯에 activity를 너무 몰아넣으면 해당 minor frame 안에 처리가 끝나지 않을 수 있으므로 적당히 분산시키는 것이 좋다.

이동 후 슬롯 #40~#99를 전부 삭제하고, 배열의 마지막 원소(슬롯 #39의 마지막 엔트리)에서 trailing comma를 제거하면 된다.

### Frequency / Remainder에 대해

스케줄 테이블의 각 엔트리에는 `Frequency`와 `Remainder` 필드가 있다. 예를 들어 `Frequency=4, Remainder=3`은 "4번째 major frame마다 실행"을 의미한다.

Major frame 1개는 전체 슬롯을 한 바퀴 도는 주기다.

- **100Hz, 100슬롯** → major frame = 100 ÷ 100 = **1초**
- **40Hz, 40슬롯** → major frame = 40 ÷ 40 = **1초**

슬롯 수와 Hz를 함께 변경했으므로 major frame 주기는 동일하게 1초로 유지된다. 따라서 `Frequency=4`인 HK request는 여전히 4초마다 실행되며, 기존과 동작이 달라지지 않는다.

만약 슬롯 수만 줄이고 Hz를 그대로 뒀다면 (40슬롯에 100Hz) major frame이 0.4초가 되어 의도치 않게 주기가 빨라지는 문제가 발생할 수 있으니 주의해야 한다.

## ✨ 마치며

간단한 수정이지만, "어떤 파일을 왜 바꿔야 하는지"를 스스로 찾아내기가 쉽지 않았다. SCH_TOTAL_SLOTS를 바꾸면 테이블 크기가 달라지고, 그러면 기존에 #40 이후에 배치된 활성 엔트리가 잘려나간다는 걸 알아채야 한다. 또한 Frequency/Remainder가 major frame 기준이라는 점을 이해해야 "슬롯 수를 줄여도 HK 주기가 안 변하는 이유"를 설명할 수 있다.

CFS-101 트레이닝 시리즈는 이것으로 마무리되었다. cFE 프레임워크 위에서 앱을 추가하고, 테이블을 수정하고, 스케줄러를 재구성하는 일련의 과정을 통해 cFS의 설계 철학 — 모듈화와 재구성 용이성 — 을 체감할 수 있었길 바란다!
