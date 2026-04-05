# 과제 1.2 보고서
# GCS-PULSE CLI + Google Calendar 통합 예약 Skill: `/book-meeting`

**작성일:** 2026-04-05  
**환경:** Claude Code (claude-sonnet-4-6), macOS  
**사용 도구:** GCS-PULSE MCP, gcalcli 4.5.1

---

## 1. 과제 목표

> GCS-PULSE CLI와 GWS CLI를 이용해 GCS 회의실 예약과 구글 캘린더를  
> 한 번에 처리(일정 등록, 팀원 초대, 초대 메일 보내기)하는 Skill을 만들어 보세요.

---

## 2. 사전 준비

### 2.1 사용 도구 선택

| 역할 | 도구 | 비고 |
|------|------|------|
| GCS 회의실 예약 | GCS-PULSE MCP | `claude mcp list` 로 연결 확인 |
| Google Calendar 연동 | gcalcli 4.5.1 | `pip install gcalcli` |
| 팀원 정보 조회 | GCS-PULSE MCP | `users_search` 툴 활용 |

### 2.2 gcalcli 설치 및 인증

```bash
# 설치
pip install gcalcli

# Google OAuth 인증 (Client ID/Secret은 Google Cloud Console에서 발급)
gcalcli --client-id "CLIENT_ID" --client-secret "CLIENT_SECRET" init
```

인증 완료 후 `~/Library/Application Support/gcalcli/config.toml` 에 자격증명 저장:

```toml
client_id = "859440639105-j21st0loqefhhiva5qgjbs0vg68s1otq.apps.googleusercontent.com"
client_secret = "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"
```

인증 확인:
```bash
$ gcalcli list
 Access  Title
 ------  -----
  owner  beomsu9665@gachon.ac.kr
 reader  대한민국의 휴일
 reader  GCS 학기제 일정
```

---

## 3. Skill 구성

### 3.1 파일 위치

```
~/.claude/commands/book-meeting.md
```

Claude Code는 `~/.claude/commands/` 디렉토리의 `.md` 파일을 슬래시 커맨드로 자동 등록합니다.  
`/book-meeting` 입력 시 이 파일의 내용이 프롬프트로 전달됩니다.

### 3.2 Skill 실행 흐름

```
사용자: /book-meeting

    │
    ▼
Step 1. 예약 정보 수집
        회의 제목 / 날짜·시간 / 참석자 / 안건

    │
    ▼
Step 2. GCS 회의실 가용 현황 확인
        mcp__gcs-pulse__meeting_rooms_list
        mcp__gcs-pulse__meeting_rooms_reservations (날짜별)

    │
    ▼
Step 3. GCS 회의실 예약
        mcp__gcs-pulse__meeting_rooms_reserve
        → 예약 ID 반환

    │
    ▼
Step 4. 참석자 이메일 조회
        mcp__gcs-pulse__users_search (이름 → 이메일 변환)

    │
    ▼
Step 5. Google Calendar 일정 생성 + 초대 메일 발송
        gcalcli add --calendar ... --who EMAIL ...

    │
    ▼
Step 6. 완료 요약 출력
```

### 3.3 Skill 파일 전문

```markdown
# /book-meeting — GCS 회의실 + 구글 캘린더 통합 예약

GCS-PULSE 회의실 예약과 Google Calendar 일정 등록, 팀원 초대 메일을 한 번에 처리합니다.

## 사용법
/book-meeting
/book-meeting 제목: 스프린트 회의, 날짜: 4월 7일 오후 2시~3시, 참석자: 홍길동 김철수

## 실행 순서

### Step 1 — 예약 정보 수집
회의 제목 / 날짜 및 시작·종료 시간 / 참석자 이름 / 목적·안건

### Step 2 — GCS 회의실 가용 현황 확인
meeting_rooms_list → meeting_rooms_reservations 으로 빈 슬롯 확인 후 선택

### Step 3 — GCS 회의실 예약
meeting_rooms_reserve(room_id, start_at, end_at, purpose)

### Step 4 — 참석자 이메일 조회
users_search 로 이름 → 이메일 변환

### Step 5 — Google Calendar 일정 생성 + 초대 메일 발송
gcalcli add \
  --calendar "beomsu9665@gachon.ac.kr" \
  --title "TITLE" \
  --when "YYYY-MM-DD HH:MM" \
  --duration MINUTES \
  --where "ROOM_NAME (ROOM_LOCATION)" \
  --description "PURPOSE\nGCS 예약 ID: RESERVATION_ID" \
  --who "EMAIL1" --who "EMAIL2" \
  --reminder "30 email" --reminder "10 popup" \
  --noprompt

### Step 6 — 완료 요약 출력
```

---

## 4. 실제 사용 예시

### 4.1 입력

```
/book-meeting  (테스트 실행 — 오늘 22:00~23:00, N.MR1 회의실)
```

---

### 4.2 Step 2 — 회의실 목록 조회

**호출:**
```
mcp__gcs-pulse__meeting_rooms_list()
```

**결과 (일부):**
```json
[
  {"id": 1, "name": "N.MR1", "location": "AI관 6층 North", "description": "최대 4인"},
  {"id": 2, "name": "N.MR2", "location": "AI관 6층 North", "description": "최대 4인"},
  {"id": 3, "name": "N.MR3", "location": "AI관 6층 North", "description": "최대 8인 화상회의실"},
  ...총 14개
]
```

---

### 4.3 Step 2 — 해당 날짜 예약 현황 확인

**호출:**
```
mcp__gcs-pulse__meeting_rooms_reservations(room_id=1, date="2026-04-05")
```

**결과:**
```json
{"items": []}   ← 빈 슬롯 확인 완료
```

---

### 4.4 Step 3 — 회의실 예약

**호출:**
```
mcp__gcs-pulse__meeting_rooms_reserve(
  room_id=1,
  start_at="2026-04-05T22:00:00+09:00",
  end_at="2026-04-05T23:00:00+09:00",
  purpose="book-meeting Skill 테스트 회의"
)
```

**결과:**
```json
{
  "id": 180,
  "meeting_room_id": 1,
  "reserved_by_name": "김범수/AI·소프트웨어학부(인공지능전공)",
  "start_at": "2026-04-05T13:00:00+00:00",
  "end_at": "2026-04-05T14:00:00+00:00",
  "purpose": "book-meeting Skill 테스트 회의",
  "can_cancel": true
}
```

→ **GCS 예약 ID: 180** 발급 완료

---

### 4.5 Step 5 — Google Calendar 등록 + 초대 메일

**실행 명령:**
```bash
gcalcli add \
  --calendar "beomsu9665@gachon.ac.kr" \
  --title "book-meeting Skill 테스트 회의" \
  --when "2026-04-05 22:00" \
  --duration 60 \
  --where "N.MR1 (AI관 6층 North)" \
  --description "GCS 회의실 예약 완료
장소: N.MR1 (AI관 6층 North)
예약 ID: 180" \
  --who "beomsu9665@gachon.ac.kr" \
  --reminder "30 email" \
  --reminder "10 popup" \
  --noprompt
```

**캘린더 등록 확인:**
```bash
$ gcalcli agenda "2026-04-05" "2026-04-06"

일  4 05   22:00   book-meeting Skill 테스트 회의
```

→ Google Calendar 일정 생성 완료  
→ `--who` 에 지정된 이메일로 **초대 메일 자동 발송**  
→ `--reminder "30 email"` 로 30분 전 이메일 알림 설정

---

### 4.6 최종 결과 요약

```
✅ 예약 완료
─────────────────────────────────────
회의 제목  : book-meeting Skill 테스트 회의
일시       : 2026-04-05  22:00 ~ 23:00
GCS 회의실 : N.MR1 (AI관 6층 North)
GCS 예약 ID: 180
참석자     : beomsu9665@gachon.ac.kr
캘린더     : 일정 생성 완료
초대 메일  : 발송 완료 (30분 전 이메일 알림 포함)
─────────────────────────────────────
```

---

## 5. Skill 설계 포인트

| 항목 | 설명 |
|------|------|
| **자연어 파라미터** | "4월 7일 오후 2시" → Claude가 `2026-04-07T14:00:00+09:00` 자동 변환 |
| **가용 현황 먼저 확인** | 예약 충돌 방지 — 빈 슬롯만 사용자에게 표시 |
| **이름 → 이메일 자동 조회** | `users_search` 로 GCS 팀원 이메일을 API에서 가져옴 |
| **초대 메일 자동 발송** | gcalcli `--who EMAIL` 이 Google Calendar 초대 메일 처리 |
| **알림 설정** | 30분 전 이메일 + 10분 전 팝업 기본 설정 |
| **GCS 예약 ID 캘린더 포함** | description에 예약 ID 기록 → 추후 취소 시 참조 가능 |

---

## 6. 한계 및 개선 방향

| 한계 | 개선 방향 |
|------|----------|
| GWS CLI 미설치 — gcalcli 대체 사용 | 공식 GWS CLI 설치 후 `gws calendar events create` 로 교체 |
| 다중 회의실 비교가 수동 | 모든 회의실의 빈 슬롯을 한 번에 조회해 자동 추천 |
| 캘린더 계정 하드코딩 | config에서 기본 캘린더 동적 조회로 개선 |

---

*Claude Code claude-sonnet-4-6 / GCS-PULSE MCP gcs-pulse-mcp v1.26.0 / gcalcli 4.5.1 / 2026-04-05*
