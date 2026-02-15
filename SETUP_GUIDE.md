# 숙소 가계부 — 자동화 설정 가이드

## 전체 파이프라인

```
Airbnb / Agoda / Booking.com → 예약 확인 이메일 (네이버웍스)
    ↓
Zapier 무료 (2스텝) → 이메일 감지 → Google Sheets에 원본 저장
    ↓
Google Sheets (Bookings 시트, 3열: 제목/날짜/본문)
    ↓
가계부 앱 JS → OTA 자동 감지 → 이메일 파싱 → 수입/지출 거래 생성
```

> **핵심**: Zapier는 이메일을 시트에 그대로 전달만 하고, 파싱은 가계부 앱 JavaScript에서 수행합니다. 무료 플랜(2스텝)으로 충분합니다.
>
> **지원 OTA**: Airbnb, Agoda, Booking.com, 삼삼엠투(1회 수동 입력 완료)

---

## A. Google Sheets 설정

### A-1. 스프레드시트 만들기

1. [Google Sheets](https://sheets.google.com) → **빈 스프레드시트** 생성
2. 파일 이름: `Airbnb 예약 데이터` (자유 지정)

### A-2. 시트 이름 설정

시트 하단 탭 이름을 **`Bookings`**로 변경 (대소문자 정확히)

### A-3. 헤더 입력 (1행)

| 셀 | 값 | 설명 |
|----|-----|------|
| A1 | `email_subject` | 이메일 제목 |
| B1 | `email_date` | 이메일 수신 날짜 |
| C1 | `email_body` | 이메일 본문 (Plain Text) |

이게 전부입니다. 3열만 있으면 됩니다.

### A-4. 웹에 게시 (필수)

1. 메뉴 **파일** → **공유** → **웹에 게시**
2. **`Bookings`** 시트 선택 → **게시** 클릭
3. 확인 대화상자에서 **확인**

### A-5. Sheet URL 복사

브라우저 주소창에서 URL 복사:
```
https://docs.google.com/spreadsheets/d/여기가_시트ID/edit...
```

---

## B. Zapier 설정 (무료 플랜, 2스텝)

### B-1. Zapier 가입

1. [zapier.com](https://zapier.com) 가입/로그인
2. **+ Create** → **New Zap**

### B-2. 트리거: 네이버웍스 이메일 수신

1. **App**: `Email by Zapier` 검색 → 선택
2. **Event**: `New Inbound Email`

> **참고**: `Email by Zapier`는 Zapier 전용 이메일 주소를 생성합니다. 네이버웍스에서 이 주소로 자동 전달 규칙을 설정하면 됩니다.
>
> **대안**: 네이버웍스가 IMAP을 지원하면 `Email by Zapier` 대신 IMAP 트리거를 사용할 수도 있습니다:
> - App: `IMAP by Zapier`
> - Event: `New Email`
> - IMAP 서버 정보: 네이버웍스 관리자 설정에서 확인

### B-3. 네이버웍스 자동 전달 설정 (Email by Zapier 사용 시)

1. 네이버웍스 메일 → **설정** → **메일 필터/전달 규칙**
2. 새 규칙 생성 (OTA별):
   - **Airbnb**: 보낸 사람이 `automated@airbnb.com` 포함
   - **Agoda**: 보낸 사람이 `no-reply@agoda.com` 포함
   - **Booking.com**: 보낸 사람이 `noreply@booking.com` 또는 `noreply@mailer.booking.com` 포함
   - **동작**: Zapier 전용 이메일 주소로 전달
3. 규칙 저장

### B-4. 액션: Google Sheets에 행 추가

1. **App**: `Google Sheets`
2. **Event**: `Create Spreadsheet Row`
3. **Account**: Google 계정 연결
4. **Spreadsheet**: A단계에서 만든 스프레드시트 선택
5. **Worksheet**: `Bookings`
6. 필드 매핑:

| Sheets 열 | Zapier 필드 |
|-----------|-------------|
| email_subject | 트리거의 `Subject` |
| email_date | 트리거의 `Date` (또는 `Received At`) |
| email_body | 트리거의 `Body Plain` (또는 Agoda 대응 시 `Body HTML`) |

> **Agoda 참고**: Agoda 이메일은 HTML-only입니다. Agoda 동기화를 위해 Body 매핑을 `Body Plain` 대신 `Body HTML` 또는 `Body`로 변경하세요. Airbnb는 두 매핑 모두 호환됩니다.

### B-5. 테스트 & 활성화

1. **Test** 클릭 → Sheets에 행이 추가되는지 확인
2. 문제 없으면 **Publish** 클릭

---

## C. 가계부 앱 연동

1. `index.html`을 브라우저에서 열기
2. 우측 상단 **⚙** 클릭
3. Sheet URL 붙여넣기
4. **동기화** 클릭
5. "완료! N개 새 거래 추가" 확인

### 앱이 이메일에서 자동 추출하는 항목

#### Airbnb

| 항목 | 이메일 내 패턴 | 예시 |
|------|---------------|------|
| 예약 코드 | `Confirmation code` 다음 줄 | `HM2YCYXW8J` |
| 게스트 이름 | 제목: `confirmed - OOO arrives` | `규빈 김` |
| 체크인 | `Check-in` 다음의 날짜 | `Feb 8` |
| 체크아웃 | `Checkout` 다음의 날짜 | `Feb 9` |
| 1박 요금 | `₩OOO x N night` | `₩30,000` |
| **정산금** | `You earn ₩OOO` | **`₩62,855`** → 수입 거래 |
| **수수료** | `Host service fee ... ₩OOO` | **`₩2,145`** → 지출 거래 |

#### Agoda (HTML 파싱, Element ID 기반)

| 항목 | HTML Element ID | 예시 |
|------|----------------|------|
| 예약 번호 | `ltrBookingIDValue` | `1688376481` |
| 게스트명 | `ltrCustomerFirstNameValue` + `ltrCustomerLastNameValue` | `Adrian Paul Lim` |
| 체크인 | `lblCustomerArrival` | `April 7, 2026` |
| 체크아웃 | `lblCustomerDeparture` | `April 10, 2026` |
| **정산금** | `lblAmountPayableData` (Net Rate) | **`KRW 348,800`** |
| **수수료** | `referenceCommissionLabel` 행 | **`KRW 61,200`** → 지출 거래 |
| 청소비 | "Cleaning Service Fee" 행 | `KRW 50,000` |

#### Booking.com (제목에서만 추출)

| 항목 | 제목 패턴 | 예시 |
|------|----------|------|
| 예약 번호 | `(숫자, ...)` | `6309203132` |
| 체크인 | `YYYY년 M월 D일` | `2026년 4월 8일` |
| 금액 | 이메일에 없음 → ₩0 플레이스홀더 | 사용자 수동 입력 |

---

## D. 테스트

### 수동 테스트 (Zapier 없이)

1. Google Sheets Bookings 시트에 직접 입력:
   - A2: `Reservation confirmed - 테스트 arrives Feb 8`
   - B2: `2026-02-06`
   - C2: 에어비앤비 이메일 본문 전체 복사-붙여넣기
2. 가계부 앱 → 동기화 → 거래 2건 생성 확인 (수입 + 지출)
3. 다시 동기화 → "추가: 0건" (중복 없음)

### Zapier 연동 테스트

1. Zapier Zap 히스토리에서 실행 확인
2. Sheets에 새 행 추가됐는지 확인
3. 가계부 앱 동기화 → 새 거래 나타나는지 확인

---

## E. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| "응답 파싱 실패" | 시트가 웹에 게시 안 됨 또는 시트 이름 불일치 | 웹에 게시 재확인, 시트 이름 `Bookings` 확인 |
| 동기화 0건 | 이메일 본문에 `Confirmation code`가 없음 | Sheets에서 C열(email_body) 내용 확인 |
| 금액이 0원 | 이메일 형식이 예상과 다름 | 이메일 본문에서 `You earn`, `Host service fee` 검색하여 형식 확인 |
| 날짜가 잘못됨 | 연도 판단 오류 (11~12월 → 1~2월 경계) | B열(email_date)에 정확한 날짜가 있는지 확인 |
