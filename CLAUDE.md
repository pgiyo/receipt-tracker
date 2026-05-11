# CLAUDE.md

이 파일은 Claude Code(claude.ai/code)가 이 저장소에서 작업할 때 참고하는 가이드입니다.

## 프로젝트 개요

영수증 지출 관리 앱 — 사용자가 영수증 이미지/PDF를 업로드하면 Upstage Vision LLM이 자동으로 내용을 파싱하여 구조화된 지출 데이터로 변환하는 경량 웹 애플리케이션입니다. 데이터베이스 없이 JSON 파일로만 저장합니다.

## 실행 명령어

### 백엔드 (Python FastAPI)
```bash
# 프로젝트 루트에서 — 가상환경 활성화 후 실행
cd backend
uvicorn main:app --reload
# Swagger UI: http://localhost:8000/docs
```

### 프론트엔드 (React + Vite)
```bash
cd frontend
npm install
npm run dev
# 실행 주소: http://localhost:5173
```

### 백엔드 패키지 설치
```bash
cd backend
pip install -r requirements.txt
```

## 아키텍처

```
frontend/src/
  pages/          # Dashboard.jsx, UploadPage.jsx, ExpenseDetail.jsx
  components/     # DropZone, ParsePreview, ExpenseCard, SummaryCard, FilterBar, Badge, Modal, Toast
  api/axios.js    # Axios 인스턴스 — VITE_API_BASE_URL 환경변수 사용

backend/
  main.py              # FastAPI 앱, CORS 설정, 라우터 등록
  routers/
    upload.py          # POST /api/upload — 파일 검증 + OCR 호출
    expenses.py        # GET/DELETE/PUT /api/expenses[/{id}]
    summary.py         # GET /api/summary
  services/
    ocr_service.py     # LangChain ChatUpstage 체인, 이미지→Base64→JSON 변환
    storage_service.py # expenses.json 읽기/쓰기 헬퍼
  data/expenses.json   # 누적 저장 JSON 배열; UUID v4 ID, ISO 8601 타임스탬프
```

## 핵심 설계 결정사항

**OCR 파이프라인**: 2단계 구조.
1. `POST https://api.upstage.ai/v1/document-digitization` (model=`ocr`)에 이미지/PDF 파일을 multipart로 직접 전송 → raw OCR 텍스트 반환. PDF는 Upstage API가 직접 처리하므로 pdf2image/Poppler 불필요.
2. `ChatUpstage(model="solar-pro")`에 OCR 텍스트를 전달하여 구조화 JSON 추출. LLM 응답이 ` ```json ``` ` 코드블록으로 감싸지므로 `ocr_service.py`에서 strip 처리 필요.

**데이터 영속성**: Vercel 서버리스 컨테이너는 실행 사이에 파일 시스템이 유지되지 않습니다. `VERCEL=1`이 감지되면 `/tmp/expenses.json`을 사용하며, 프론트엔드는 `localStorage`에도 병행 저장합니다.

**파일 검증**: 최대 10MB, 허용 MIME 타입: `image/jpeg`, `image/png`, `application/pdf`. OCR 호출 전 `upload.py`에서 서버 측 재검증합니다.

**오류 처리**: Toast 알림(`fixed bottom-4 right-4`, 3초 자동 소멸). 로딩 중에는 버튼 비활성화(`disabled opacity-50 cursor-not-allowed`). OCR 실패 시 빨간 배너와 재시도 버튼을 표시합니다.

## 환경변수

| 변수명 | 사용 위치 | 비고 |
|---|---|---|
| `UPSTAGE_API_KEY` | 백엔드 | 필수; Vercel 대시보드 또는 `.env`에 설정 |
| `VITE_API_BASE_URL` | 프론트엔드 빌드 | 프로덕션에서는 빈 문자열 (동일 도메인 상대 경로 사용) |
| `DATA_FILE_PATH` | 백엔드 | `VERCEL=1` 감지 시 `/tmp/expenses.json`으로 자동 설정 |

## API 엔드포인트

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/api/upload` | 영수증 업로드, 파싱된 JSON 반환 |
| GET | `/api/expenses` | 지출 목록 조회; `from`/`to` (YYYY-MM-DD) 쿼리 파라미터 지원 |
| DELETE | `/api/expenses/{id}` | UUID로 항목 삭제 |
| PUT | `/api/expenses/{id}` | 항목 부분 수정 |
| GET | `/api/summary` | 지출 합계; `month` (YYYY-MM) 쿼리 파라미터 지원 |

## 지출 항목 JSON 스키마

```json
{
  "id": "uuid-v4",
  "created_at": "ISO8601",
  "store_name": "string",
  "receipt_date": "YYYY-MM-DD",
  "receipt_time": "HH:MM | null",
  "category": "식료품|외식|교통|쇼핑|의료|기타",
  "items": [{"name": "string", "quantity": 0, "unit_price": 0, "total_price": 0}],
  "subtotal": 0, "discount": 0, "tax": 0, "total_amount": 0,
  "payment_method": "string | null"
}
```

## 스타일링

TailwindCSS v3. 주색상: `indigo-600`. 폰트: Pretendard(CDN), Noto Sans KR 폴백. 반응형 그리드: `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3`. 커스텀 키프레임 애니메이션(`slide-up`, `scale-in`, `fade-in`)은 `tailwind.config.js`에 등록합니다.

## 바이브 코딩 3원칙 (PRD 기준)

1. **"완료 기준"을 먼저 정의하라** — 각 Phase 구현을 요청하기 전에 3~5개의 완료 체크리스트를 먼저 작성합니다.
2. **새로운 기술은 조사 먼저, 구현 나중** — `context7`을 활용하여 `langchain-upstage` / Vite 환경변수 / Vercel Python 서버리스 API의 최신 사용법을 코드 작성 전에 확인합니다.
3. **버그는 분석 먼저, 수정 나중** — 에러 메시지와 관련 코드를 공유하고 근본 원인 분석을 먼저 요청한 뒤, 수정 방향에 동의하고 나서 수정을 진행합니다.

### Source Code가 변경되거나 라이브러리 버전이 변경되면 반드시 @PRD_영수증_지출관리앱.md 같이 업데이트 하고, 완료 기준의 Check Box에 완료된 사항들도 모두 체크표시 하세요.