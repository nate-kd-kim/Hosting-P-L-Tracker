# 숙소 가계부 — 프로젝트 컨텍스트

## 현재 상태: 멀티 OTA 지원 완료 (Airbnb, Agoda, Booking.com, 삼삼엠투)

## 아키텍처 결정사항

- **이메일**: 네이버웍스로 수신 (Gmail 아님)
- **Zapier**: 무료 플랜 (2스텝) — 이메일 원본을 Sheets에 그대로 저장만 함
- **파싱**: 가계부 앱 JavaScript에서 OTA별 파서로 이메일 파싱
  - Airbnb: 정규식 (Plain Text)
  - Agoda: DOMParser (HTML Element ID 기반)
  - Booking.com: 제목에서 예약번호/날짜 추출 (금액은 수동 입력)
  - 삼삼엠투: 1회 수동 입력 (앱 초기화 시 자동 삽입)
- **Sheets 구조**: 3열 (email_subject, email_date, email_body)

## 파이프라인

```
네이버웍스 → (자동전달) → Zapier → Google Sheets → 가계부 앱 JS 파싱
                                                     ↳ detectOta() → OTA별 파서 디스패치
```

## OTA별 이메일 파싱

### Airbnb (parseEmailBody)
- `Confirmation code\s*([A-Z0-9]+)` → booking_id
- `confirmed - (.+?) arrives` → guest_name (제목에서)
- `₩([\d,]+)\s*x\s*\d+\s*night` → nightly_rate
- `You earn\s*₩([\d,]+)` → total_payout
- 거래 ID: `airbnb_{bookingId}_{type}`

### Agoda (parseAgodaEmail, HTML DOMParser)
- `ltrBookingIDValue` → booking_id
- `ltrCustomerFirstNameValue` + `ltrCustomerLastNameValue` → guest_name
- `lblCustomerArrival` / `lblCustomerDeparture` → check_in / check_out
- `lblAmountPayableData` → net_rate
- `referenceCommissionLabel` 행 → commission
- 거래 ID: `agoda_{bookingId}_{type}`

### Booking.com (parseBookingEmail, 제목 전용)
- 제목: `신규 예약 안내! (번호, YYYY년 M월 D일)`
- 금액 없음 → ₩0 플레이스홀더, 사용자 수동 입력
- 거래 ID: `booking_{bookingId}_placeholder`

### 삼삼엠투 (addSamsamTransaction, 1회)
- 2025-12-20, 숙박 ₩1,000,000, 청소 ₩50,000, 수수료 ₩34,650
- 거래 ID: `samsam_20251220_{type}`

## OTA 감지 (detectOta)
- 제목에 `Agoda Booking ID` → agoda
- 제목에 `Booking.com` / `신규 예약 안내` / `취소된 예약 안내` → booking
- 기본값 → airbnb

## 취소 감지 (detectCancellation)
- Airbnb: `cancel` + `Reservation {ID}` or `Confirmation code {ID}`
- Agoda: `CANCELLED` + `Agoda Booking ID {ID}`
- Booking.com: `취소된 예약` + `({ID},...)`

## 파일 목록

| 파일 | 설명 |
|------|------|
| `index.html` | 가계부 앱 (단일 파일, 멀티 OTA 지원) |
| `SETUP_GUIDE.md` | Sheets + Zapier + 테스트 설정 가이드 |
| `CONTEXT.md` | 이 파일 |
| `eml/` | 테스트용 샘플 EML 파일 (Agoda 4건, Booking 4건) |

## 남은 수동 작업

- [ ] 네이버웍스 자동전달 규칙에 Agoda, Booking.com 발신자 추가
- [ ] (선택) Zapier Body 매핑을 `Body HTML`로 변경 (Agoda Sheets 동기화용)
