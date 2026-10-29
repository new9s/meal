# Aside 연결용 식단 JSON 규격

## 역할

- Hermes: 원본 수집·추출·검증, 이 폴더의 JSON 게시.
- Aside: 식단앱 개발·배포. Hermes는 앱의 `index.html`을 수정하지 않습니다.
- JSON은 공개 데이터입니다. 인증정보·실행 로그·사설 경로는 담지 않습니다.

## 조회 주소

- 주차 목록: `https://new9s.github.io/meal/data/index.json`
- 주차별 식단: `https://new9s.github.io/meal/data/weeks/<월요일 YYYY-MM-DD>.json`
- 예: `weeks/2026-10-26.json`, `weeks/2026-11-02.json`

목록 형식:

```json
{
  "schema_version": "1.0",
  "timezone": "Asia/Seoul",
  "weeks": [
    {
      "start": "YYYY-MM-DD",
      "end": "YYYY-MM-DD",
      "path": "weeks/YYYY-MM-DD.json",
      "board_idx": 0,
      "image_sha256": "원본 이미지의 SHA-256",
      "sha256": "주차 JSON 바이트의 SHA-256"
    }
  ]
}
```

위 블록은 형식 설명용 예시이며 실제 공지·해시값이 아닙니다. 운영 목록에는 검증된 실제 값이 들어갑니다.

`path`는 **인덱스 URL 기준 상대 경로**입니다. 사이트 루트 기준으로 붙이지 않습니다.

```javascript
const indexUrl = new URL('data/index.json', location.href);
const index = await fetch(indexUrl, {cache: 'no-store'}).then(r => {
  if (!r.ok) throw new Error('주차 목록 조회 실패');
  return r.json();
});
// row는 사용자가 선택한 기간 또는 한국 날짜에 해당하는 주차입니다.
const weekUrl = new URL(row.path, indexUrl);
weekUrl.searchParams.set('v', row.sha256);
const week = await fetch(weekUrl, {cache: 'no-store'}).then(r => {
  if (!r.ok) throw new Error('식단 조회 실패');
  return r.json();
});
```

## 주차별 식단

첨부 정답과 같은 `schema_version: "1.0"` 구조입니다.

- `source`: `board_idx`, `notice_url`, `notice_title`, `notice_content`, `image_url`, `original_filename`, `image_width`, `image_height`.
- `period`: `start`, `end`, `year_basis`.
- `days[]`: `date`, `weekday`, `date_label`, `meals[]`.
- `meals[]`: `date`, `weekday`, `meal`, `menus`, `main_menus`, `plus`, `self_ramen`.
- `main_menus`: `menus`와 같은 구획 순서의 배열의 배열. 원본의 **빨간 글씨가 포함된 메뉴 행**만 메인으로 표시합니다. 첫 항목이라는 이유로 메인으로 추정하지 않습니다. 색상 혼합 행도 메뉴 문자열 전체를 보존합니다.

```json
{
  "menus": [["바베큐닭살볶음", "쌀밥/현미밥", "호박된장국"], ["산채나물볶음밥", "건새우미역국"]],
  "main_menus": [["바베큐닭살볶음"], ["산채나물볶음밥"]]
}
```

위 코드는 구조 설명용 부분 예시입니다. 실제 파일에는 모든 메뉴·부가 항목이 들어 있습니다.
- `transcription`: 추출 방식, 정규화, `uncertain_items` 등.
- `table_labels`, `footer_lines`: 원문 라벨과 안내문.
- `provenance`: `image_sha256`, `extracted_at`, `extractor_version`, `identity_verified`, `identity_basis`, `validation_status`, `automation_note`.

확인하지 못한 공지 제목·파일명 등은 `null`일 수 있습니다. 이미지 파일명을 공지의 원본 파일명으로 가정하지 않습니다.

`meal` 값과 기존 앱 키의 대응:

| JSON | 앱 |
|---|---|
| 조식 | breakfast |
| 중식 | lunch |
| 석식 | dinner |
| 야식 | night |

`menus`는 문자열 배열의 배열입니다. 각 내부 배열이 원본의 한 메뉴 구획입니다. 제목은 각 내부 배열의 첫 항목이며 나머지는 원문 순서대로 표시합니다. 별도 쌀밥 줄이 없는 식단도 있으므로 항목 수를 고정하지 않습니다.

- 2156: 화요일 중식 두 구획, 주간 총 29구획.
- 2160: 화요일·목요일 중식 두 구획, 주간 총 30구획.
- `plus`와 `self_ramen`은 위치가 다른 원문 항목이므로 임의로 합치거나 중복 제거하지 않습니다.
- 텍스트는 `textContent`로 표시하고 메뉴 문자열을 HTML로 실행하지 않습니다.

## 날짜·갱신·실패 정책

- 새 공지 확인: 매주 수요일·금요일 오전 9시, `Asia/Seoul` 기준.
- 보관: 식단 종료일에 14일을 더한 한국 날짜부터 공개 목록·주차 JSON에서 제외하고 운영 승인 사본·불필요한 원본 캐시를 정리합니다. 삭제는 정기 작업 실행 때 적용하며 GitHub 커밋 이력을 강제로 지우지 않습니다.

- 한국 기준 달력 날짜로 `start <= 날짜 <= end`인 주차를 선택합니다. 공지 번호가 가장 큰 주차를 현재 주차로 선택하지 않습니다.
- `weeks`는 시작일 오름차순입니다. 다음 주 선게시가 가능하며 누락 주차도 가능하므로 배열 인접 항목을 이용해 이전·다음 주를 표시합니다.
- 자정 야식의 전날 처리 등 앱 동작은 Aside가 기존 의미를 유지해 구현합니다.
- 목록은 재검증하고, 주차 파일은 `sha256` 쿼리로 수정본 캐시를 구분합니다. 기간·버전 검증 실패 시 재조회 또는 오류·이전 정상본 상태를 표시합니다.
- 식단이 없는 날짜에 과거 식단을 이번 주 식단처럼 표시하지 않습니다. JSON 장애 시 `source.image_url`로 원본 확인 수단을 제공할 수 있습니다.
- 웹페이지에서 공지 번호 탐색이나 Tesseract OCR을 실행할 필요가 없습니다.
- 구조 검증은 문자 정확도 보증과 다릅니다. 오류율은 별도 정답 대조가 있을 때만 수치로 제시합니다.

## 현재 데이터의 출처와 제한

초기 두 주차는 제공된 실제 이미지에서 준비했습니다. 2156은 사용자 제공 정답 사본에 사용자가 확인한 `메콤 → 매콤` 네 곳의 정정을 반영했습니다. 2160은 Hermes에서 원본을 확대 판독했으며, 별도 정답지가 없어 오류율은 미측정입니다.

공지 상세 조회는 이 Hermes 실행 환경에서 HTTP 403이 확인되었습니다. Aside에서는 같은 URL을 별도 헤더·프록시 지정 없이 `fetch()`로 HTTP 200 확인했다는 사용자 제공 사실이 있습니다. 외부 접속 IP와 원인은 미확인입니다. 원본 이미지 URL은 정상 조회할 수 있습니다.

따라서 초기 JSON 공개와 신규 공지 무인 수집 완료를 구분합니다. 403이 유지되면 정상 JSON을 보존하며 자동으로 새 메뉴를 만들어 채우지 않습니다.
