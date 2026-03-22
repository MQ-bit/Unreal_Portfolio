# 🎮 Dungeon Warrior - Unreal Engine Portfolio

- 언리얼 엔진 개인 포트폴리오 프로젝트
- 프로젝트명: Warrior
- 개발 형태: 1인 개발
- 사용 엔진: Unreal Engine 5.4
- 개발 언어: C++

* * *

## 📌 프로젝트 소개

Dungeon Warrior는 Unreal Engine 5.4 기반으로 제작한 3D 액션 RPG / 웨이브 서바이벌 프로토타입입니다.

입력 처리, 전투 시스템, 상태 관리, UI 연동, 웨이브 진행 구조를 중심으로 구현했습니다.

📽[시연 영상 확인]  

📜[기술 문서 확인]

* * *

## 🛠 기술 스택

- Unreal Engine 5.4
- C++
- Gameplay Ability System
- Gameplay Tags
- Enhanced Input
- UMG
- AI
- DataTable


* * *

## 📌 주요 구현 내용

### 전투 시스템

- Enhanced Input 기반 입력 처리
- AbilitySystemComponent를 통한 Ability 활성화 및 취소
- Gameplay Tag 기반 상태 관리
- AttributeSet 기반 체력, Rage, 대미지 후처리
- HeroUIComponent delegate 기반 UI 갱신

### 타겟 락 시스템

- 타겟 탐색
- 카메라 회전 보정
- 이동 속도 변경
- 입력 컨텍스트 전환
- 종료 시 상태 복구 처리

### 웨이브 서바이벌 시스템

- DataTable 기반 웨이브 정의
- Soft Class + Async Load 기반 적 클래스 선로딩
- 웨이브 상태 전이 관리
- Spawn Point 및 NavigationSystem 기반 위치 보정
- 웨이브 완료 후 다음 상태 전환 처리

### 데이터 기반 구성

- StartUpData로 초기 Ability / Gameplay Effect 주입
- 무기별 Ability / 입력 / 애니메이션 레이어 데이터 분리
- 캐릭터 코드 수정 없이 데이터 변경으로 기능 확장 가능하도록 구성

* * *

## 🚀 프로젝트 목표

- 상태 기반 전투 구조 설계
- GAS / Gameplay Tags 연동
- 웨이브 시스템 설계
- 런타임 안정화 및 디버깅 경험

