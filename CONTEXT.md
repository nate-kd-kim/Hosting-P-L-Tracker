# 에어비앤비 가계부 — 프로젝트 컨텍스트

## 현재 상태: 코드 완성, Zapier/Sheets 수동 설정 필요

## 아키텍처 결정사항

- **이메일**: 네이버웍스로 수신 (Gmail 아님)
- **Zapier**: 무료 플랜 (2스텝) — 이메일 원본을 Sheets에 그대로 저장만 함
- **파싱**: 가계부 앱 JavaScript에서 정규식으로 이메일 본문 파싱
- **Sheets 구조**: 3열 (email_subject, email_date, email_body)

## 파이프라인

```
네이버웍스 → (자동전달) → Zapier → Google Sheets → 가계부 앱 JS 파싱
```

## 이메일 파싱 패턴 (parseEmailBody 함수)

실제 에어비앤비 이메일 기반으로 작성됨:
- `Confirmation code\s*([A-Z0-9]+)` → booking_id
- `confirmed - (.+?) arrives` → guest_name (제목에서)
- `Check-in\s+\w+,\s*(\w{3})\s+(\d{1,2})` → check_in
- `Checkout\s+\w+,\s*(\w{3})\s+(\d{1,2})` → check_out
- `x\s*(\d+)\s*night` → nights
- `₩([\d,]+)\s*x\s*\d+\s*night` → nightly_rate
- `cleaning fee\s*₩([\d,]+)` → cleaning_fee
- `Host service fee[^₩]*₩([\d,]+)` → service_fee (→ 지출 거래)
- `You earn\s*₩([\d,]+)` → total_payout (→ 수입 거래)

## 파일 목록

| 파일 | 설명 |
|------|------|
| `index.html` | 가계부 앱 (단일 파일) |
| `SETUP_GUIDE.md` | Sheets + Zapier + 테스트 설정 가이드 |
| `CONTEXT.md` | 이 파일 |

## 남은 수동 작업

- [ ] Google Sheets 생성 (3열) + 웹에 게시
- [ ] Zapier 가입 + 2스텝 Zap 생성 (IMAP/Email 트리거 → Sheets 기록)
- [ ] 네이버웍스 자동 전달 규칙 설정
- [ ] 가계부 앱에서 동기화 테스트
