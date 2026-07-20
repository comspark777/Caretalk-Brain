---
domain: Infra-DB
date: 2026-07-20
tags: [db, roles, permission, insert]
status: complete
related: ["[[_INDEX]]", "CareTalkData/20260720_insert_role_310_coordi_assistant.sql"]
---
# Role 310 COORDI_ASSISTANT 추가

## 한 일
- 권한 테이블 9종 파악 후, `CARETALK_Roles`에 **RoleId 310 / COORDI_ASSISTANT** 신규 추가.
- 테스트서버·실서버 **양쪽 모두 적용** 완료(에릭 직접 요청, 단건 INSERT라 실서버 예외 진행).
- 패치 파일: `CareTalkData/20260720_insert_role_310_coordi_assistant.sql` (`IF NOT EXISTS` 가드).

## 결정/맥락 (왜)
- RoleName 철자는 `COORDI_ASSISTNAT`(입력 오타) 대신 **`COORDI_ASSISTANT`** 로 교정해 삽입(에릭 확인).
- RoleId는 identity 아님 → 명시 삽입. 기존 최대 300, 10단위 증분 관례에 맞춰 310.

## 권한 테이블 구성 (참고)
- Roles(정의) / UserRoles(사용자 배정, 6838) / RoleClaims(기능권한, 57) / Menu(59) / Menu_Role(메뉴노출, 174) / Menu_Quick(즐겨찾기, 355) / AuthInfo(세션1·폐기) / Log_Auth(URL인가로그, 7) / Customer_Chater_AgedCare_Right(권리헌장 문서, 접근제어 무관).
- 권한 3축: Roles → UserRoles(배정) + RoleClaims(기능) + Menu_Role(메뉴). 관리화면 `/Admin/RoleClaims`, `/Admin/Manage/MenuManage`.

## 다음 할 일
- [ ] COORDI_ASSISTANT 실사용 시 UserRoles(배정)·Menu_Role(메뉴노출)·필요시 RoleClaims(기능) 매핑 추가 필요.
