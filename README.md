# RFID Security MotorSystem
RFID 기반 사용자 인증과 IR 리모컨 제어를 결합한 RTOS 기반 모터 보안 제어 시스템

## 프로젝트 개요
본 프로젝트는 **RFID UID 기반 사용자 인증**, **IR 리모컨 제어**,  
**FreeRTOS 멀티태스킹 구조**를 결합하여  
**인증된 사용자만 모터를 제어할 수 있도록 설계한 임베디드 보안 시스템** 입니다.

메인 MCU로 **STM32 Nucleo-F411RE**를 사용하였으며, RFID 인증 상태에 따라 모터 제어 권한을 **동적으로 허용/차단**하도록 구현하였습니다.

---

###  1. RFID 기반 사용자 인증
- MFRC522 RFID 리더기를 이용한 UID 기반 사용자 인증
- Flash 메모리에 등록된 UID만 시스템 접근 허용
- 미등록 UID 접근 시 즉시 동작 거부
- 인증 상태를 전역 상태로 관리하여 모든 제어 로직의 기준으로 사용


###  2. 인증 연동 IR 리모컨 모터 제어
- IR 리모컨 입력을 통한 모터 제어
- 인증 상태에서만 IR 명령 처리
- 인증 해제 시 모든 모터 제어 차단

| 버튼 | 기능 |
|----|----|
| 1번 | 정방향 회전 (CCW) |
| 2번 | 역방향 회전 (CW) |
| 3번 | 모터 OFF |
| 5번 | 로그아웃 (인증 해제) |


###  3. RTOS 기반 멀티태스킹 구조
- FreeRTOS 기반 Task 분리 설계
- RFID / IR / Motor 기능을 독립 Task로 구성
- Blocking 최소화를 통한 실시간 응답성 확보


###  4. 메시지 큐 기반 이벤트 처리
- Task 간 직접 제어 제거
- 메시지 큐(`core_msg_queue`) 기반 이벤트 전달
- 이벤트 중심(Event-driven) 구조 설계
---

## 하드웨어 구성

| 부품 | 역할 | STM32 핀 |
|-----|-----|---------|
| STM32 Nucleo-F411RE | 메인 MCU, 시스템 제어 | - |
| RFID 리더기 (MFRC522) | UID 인증 및 등록 | PA5(SCK), PA6(MISO), PA7(MOSI), PB6(SDA), PA9(RST) |
| IR 수신기 | 리모컨 입력 수신 | PB10 |
| 스텝 모터 + ULN2003 | 모터 구동 | PA8(IN1), PB4(IN2), PB5(IN3), PB3(IN4) |

---

## 시스템 동작 흐름
<img width="600" height="493" alt="image" src="https://github.com/user-attachments/assets/41da3f7a-2730-47d7-b69b-5d7c1f68afe6" />

### 사용자 인증
1. RFID 태그 인식
2. UID SPI 통신으로 읽기
3. UID 등록 여부 확인
4. 등록된 UID → 인증 상태 ON  
   미등록 UID → 접근 거부

### 모터 제어
- 인증 상태에서만 IR 리모컨 입력에 따라 모터 동작
  - 1번: 정방향 회전
  - 2번: 역방향 회전
  - 3번: 모터 OFF

### 로그아웃
- IR 리모컨 5번 버튼 입력 시 로그아웃
- 인증 상태 OFF → 모터 제어 차단

### UID 등록 및 삭제
- 4번 버튼: 등록 모드 진입 → RFID 태깅 → UID Flash 저장
- Blue 버튼: 삭제 모드 진입 → RFID 태깅 → UID 삭제

---

## RTOS Task 구성

| Task 이름 | 우선순위 | 역할 |
|----------|---------|-----|
| defaultTask | Normal | 시스템 메인 제어, UID 상태 관리 |
| Motor Task | Low | 스텝 모터 제어 |
| IR Task | Low | IR 신호 수신 및 명령 처리 |
| RC522 Task | Low | RFID 통신 및 UID 관리 |

---

## 소프트웨어 설계

### Core Logic
- 메시지 큐(core_msg_queue) 기반 Task 간 이벤트 처리
- 주요 이벤트 흐름
  - RFID_LOGIN → Motor_Awake()
  - RFID_LOGOUT → Motor_Sleep()
  - IR 신호 수신 → 정/역방향 및 OFF 제어
  - UID 등록 모드 / 삭제 모드 처리

### UID Manager 로직
- UID 관리 모드: Normal(0) / Delete(1) / Register(2)
- 주요 기능
  - UIDManager_Init(): Flash → RAM 캐시 로딩
  - UIDManager_IsRegistered(): UID 존재 여부 확인
  - UIDManager_SaveUID(): UID 등록
  - UIDManager_DeleteUID(): UID 삭제

---

## 시연 결과 요약
- 등록 카드 / 미등록 카드 동작 구분 확인
- 등록 모드에서 UID 저장 성공
- 삭제 모드에서 UID 삭제 성공
- 모터 제어 동작
  - 1번: CCW
  - 2번: CW
  - 3번: OFF

---

## 사용 기술
- C
- STM32F411RE
- FreeRTOS
- SPI (MFRC522), GPIO, UART
- ULN2003
- STM32CubeIDE
