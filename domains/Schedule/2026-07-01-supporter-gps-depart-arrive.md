---
domain: Schedule
date: 2026-07-01
tags: [pwa, gps, supporter, schedule, routes-api]
status: captured
related: ["[[_INDEX]]"]
---
# Supporter GPS Depart/Arrive Flow

## 한 일
- `2026.04.28_suho` 브랜치가 PWA 버전 브랜치임을 확인했다.
- PWA/Web 환경에서 고객-서포터 간 GPS 연동 가능성을 검토했다.
- 실시간 백그라운드 추적보다 서포터의 `출발` / `도착` 버튼 기반 GPS 스탬프가 현실적인 방향이라고 정리했다.
- Google Maps Platform Routes API를 사용하면 GPS 좌표 기반 직선거리가 아니라 실제 도로 기준 거리와 예상 이동시간 계산이 가능함을 확인했다.

## 결정/맥락 (왜)
- 브라우저 Geolocation API로 현재 위치 좌표를 얻는 것 자체는 무료이고, HTTPS와 사용자 위치 권한 허용이 필요하다.
- PWA만으로 앱 수준의 백그라운드 GPS 추적은 안정적으로 어렵다.
- CareTalk 업무 흐름에서는 서포터가 서비스 종료 후 다음 서비스 장소로 이동하므로, `출발` 시점 좌표/시간과 `도착` 시점 좌표/시간을 저장하는 방식이 개인정보 부담과 구현 난이도 측면에서 적합하다.
- 도로 기준 거리/예상 시간은 Google Routes API 과금 대상이다. 단순 좌표 저장과 외부 지도 앱 링크 열기는 운영비 부담이 낮다.

## 설계 메모
- 예상 흐름:
  - 현재 서비스 종료
  - 다음 서비스 카드 표시
  - 서포터가 `출발` 클릭
  - 브라우저 GPS 좌표 획득
  - 출발 좌표/시간 저장
  - 필요 시 서버에서 Google Routes API 호출
  - 고객/관리자 화면에 예상 도착 시간 표시
  - 서포터가 현장 도착 후 `도착` 클릭
  - 도착 좌표/시간 저장
- 저장 후보 필드:
  - `ServiceOrderItemId`
  - `SupporterId`
  - `TravelStatus`
  - `DepartedAt`
  - `DepartedLatitude`
  - `DepartedLongitude`
  - `ArrivedAt`
  - `ArrivedLatitude`
  - `ArrivedLongitude`
  - `EstimatedDistanceMeters`
  - `EstimatedDurationSeconds`
  - `ActionUserId`
  - `CreateDate`
  - `UpdateDate`
  - `IsDelete`

## 다음 할 일
- [ ] 서포터 일정 화면에서 `출발` / `도착` 버튼이 들어갈 위치 확인
- [ ] 기존 서비스 오더/스케줄 테이블 구조 확인 후 별도 이동 로그 테이블 또는 기존 테이블 확장 여부 결정
- [ ] 고객 주소 좌표가 이미 저장되어 있는지 확인
- [ ] Google Routes API 사용 여부와 API 키/과금 정책 결정
- [ ] MVP는 좌표/시간 저장부터 구현하고, 도로 거리/ETA는 후속으로 분리 검토
