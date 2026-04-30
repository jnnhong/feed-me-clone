# url-to-md 구현 계획

## 아키텍처 결정

| 결정 | 선택 | 이유 |
|---|---|---|
| 본문 추출 실행 위치 | Server (API Route) | defuddle은 Node.js 환경 전제; 임의 URL 호출 시 CORS 우회 필요 |
| Markdown 렌더러 | `react-markdown` | JSX 기반 렌더링; defuddle이 Markdown 반환하므로 표시 용도로만 필요 |
| 상태 관리 | `useUrlConversion` hook | 단일 변환 플로우; 외부 상태 라이브러리 불필요 |
| 페이지 경로 | `app/page.tsx` (루트) | 단일 페이지 도구; 별도 서브 라우트 없음 |
| 다크모드 전략 | `next-themes` `defaultTheme="system" enableSystem` | OS 설정 자동 추종; 인앱 토글 없음 |
| LLM 전달 방식 | 클립보드 복사 후 새 탭 열기 | 외부 서비스 API 의존 없음; 콘텐츠 크기 제한 없음 |

## 인프라 리소스

None

## 데이터 모델

### ConversionResult
- title (string, required)
- author? (string)
- date? (string)
- domain (string, required)
- markdown (string, required)

### ConversionError
- type: `'invalid-url' | 'unreachable' | 'extraction-failed'`

### PromptOption
- id: `'none' | 'summarize' | 'translate-ko' | 'explain-simple' | 'custom'`
- label: string
- text?: string (프리셋만; 'none'·'custom'은 없음)

## 필요 스킬

| 스킬 | 적용 Task | 용도 |
|---|---|---|
| next-best-practices — route-handlers | Task 1 | API Route 구조 (POST, Response.json, 에러 상태 코드) |
| shadcn — styling | Task 2–4 | 시맨틱 토큰, cn(), className은 레이아웃만 |
| shadcn — composition | Task 2–4 | Skeleton, Badge, Separator, DropdownMenu 그룹 규칙 |
| vercel-react-best-practices — async-defer-await | Task 1 | API Route 조기 반환 최적화 |

## 영향 받는 파일

| 파일 경로 | 변경 유형 | 관련 Task |
|---|---|---|
| `types/url-to-md.ts` | New | Task 1 |
| `lib/url-validation.ts` | New | Task 1 |
| `lib/url-validation.test.ts` | New | Task 1 |
| `app/api/convert/route.ts` | New | Task 1 |
| `app/api/convert/__tests__/route.test.ts` | New | Task 1 |
| `hooks/useUrlConversion.ts` | New | Task 2 |
| `hooks/useUrlConversion.test.ts` | New | Task 2 |
| `components/url-to-md/URLInput.tsx` | New | Task 2 |
| `components/url-to-md/URLInput.test.tsx` | New | Task 2 |
| `components/url-to-md/MarkdownResult.tsx` | New | Task 2 |
| `components/url-to-md/MarkdownResult.test.tsx` | New | Task 2 |
| `app/page.tsx` | Modify | Task 2 |
| `components/url-to-md/ExportButtons.tsx` | New | Task 3 |
| `components/url-to-md/ExportButtons.test.tsx` | New | Task 3 |
| `lib/llm-url.ts` | New | Task 4 |
| `lib/llm-url.test.ts` | New | Task 4 |
| `components/url-to-md/PromptSelector.tsx` | New | Task 4 |
| `components/url-to-md/PromptSelector.test.tsx` | New | Task 4 |
| `components/url-to-md/LLMOpenDropdown.tsx` | New | Task 4 |
| `components/url-to-md/LLMOpenDropdown.test.tsx` | New | Task 4 |
| `app/layout.tsx` | Modify | Task 5 |

## Tasks

### Task 1: 변환 API + 서버 핵심 로직

> ⚠️ HIGH RISK — defuddle API가 예상과 다를 경우 전체 플로우에 영향. 먼저 실행해 실패를 조기 확인한다.

- **담당 시나리오**: Scenario 1 (서버 사이드 happy path), Scenario 10 (URL 접근 불가), Scenario 11 (본문 추출 실패)
- **크기**: M (3 구현 파일)
- **의존성**: None
- **참조**:
  - next-best-practices — route-handlers
  - vercel-react-best-practices — async-defer-await
  - defuddle 설치: `bun add defuddle`
  - react-markdown 설치: `bun add react-markdown`
  - defuddle 문서: https://defuddle.md/docs
- **구현 대상**:
  - `types/url-to-md.ts`
  - `lib/url-validation.ts` + `lib/url-validation.test.ts`
  - `app/api/convert/route.ts` + `app/api/convert/__tests__/route.test.ts`
- **수용 기준**:
  - [ ] 유효한 URL로 `POST /api/convert` 호출 시 `{ title, markdown, domain }` JSON과 HTTP 200을 반환한다
  - [ ] 존재하지 않거나 접근이 차단된 URL로 POST 시 HTTP 502와 `{ error: 'unreachable' }` 를 반환한다
  - [ ] 접근은 되지만 추출 가능한 본문이 없는 URL로 POST 시 HTTP 422와 `{ error: 'extraction-failed' }` 를 반환한다
- **검증**:
  - `bun run test -- url-validation`
  - `bun run test -- convert`

---

### Task 2: URL 입력 UI + 변환 결과 + 에러 표시

- **담당 시나리오**: Scenario 1 (전체), Scenario 2, 3, 9, 10, 11 (클라이언트 UI)
- **크기**: M (4 구현 파일 + page 수정)
- **의존성**: Task 1 (API 응답 계약)
- **참조**:
  - shadcn — styling, composition (Skeleton, Separator 사용; semantic tokens)
  - vercel-react-best-practices — async-defer-await
  - wireframe: 초기 화면, 로딩 중, 변환 결과, 에러 상태 스크린
- **구현 대상**:
  - `hooks/useUrlConversion.ts` + `hooks/useUrlConversion.test.ts`
  - `components/url-to-md/URLInput.tsx` + `URLInput.test.tsx`
  - `components/url-to-md/MarkdownResult.tsx` + `MarkdownResult.test.tsx`
  - `app/page.tsx` (modify: FeedMe 앱으로 교체)
- **수용 기준**:
  - [ ] 변환 버튼 클릭 직후 로딩 인디케이터(spinner)가 보인다
  - [ ] 변환 완료 후 페이지 제목 텍스트가 결과 영역 상단에 표시된다
  - [ ] 변환 완료 후 Markdown 본문이 렌더링된 형태로 표시된다
  - [ ] 지우기(X) 버튼 클릭 후 URL 입력창이 빈 상태가 된다
  - [ ] 지우기 전에 결과가 표시돼 있었다면, 결과 영역은 변하지 않는다
  - [ ] 재변환 완료 후 결과 영역에 새 페이지 제목이 표시된다
  - [ ] 이전 페이지 제목은 더 이상 보이지 않는다
  - [ ] 유효하지 않은 URL로 변환 시도 시 "유효하지 않은 URL입니다" 에러 메시지가 URL 입력창 근처에 표시된다
  - [ ] 유효하지 않은 URL 에러 시 로딩 인디케이터가 나타나지 않는다
  - [ ] 유효하지 않은 URL 에러 시 이전 변환 결과가 있다면 결과 영역이 변하지 않는다
  - [ ] 지우기(X) 버튼 클릭 후 에러 메시지가 사라진다
  - [ ] API에서 unreachable 에러 수신 시 로딩 인디케이터가 사라진 뒤 "페이지에 접근할 수 없습니다" 메시지가 표시된다
  - [ ] API에서 unreachable 에러 수신 시 결과 영역에 빈 Markdown이 표시되지 않는다
  - [ ] API에서 extraction-failed 에러 수신 시 "본문을 추출할 수 없습니다" 에러 메시지가 표시된다
  - [ ] API에서 extraction-failed 에러 수신 시 결과 영역에 빈 내용이 렌더링되지 않는다
- **검증**:
  - `bun run test -- useUrlConversion`
  - `bun run test -- URLInput`
  - `bun run test -- MarkdownResult`

---

### Checkpoint: Tasks 1–2 이후
- [ ] 모든 테스트 통과: `bun run test`
- [ ] 빌드 성공: `bun run build`
- [ ] Browser MCP — `/` 접속, 실제 URL 입력 후 변환, 제목·본문 렌더링 확인; 잘못된 URL로 에러 메시지 확인; 증거 `artifacts/url-to-md/evidence/checkpoint-1.png` 저장

---

### Task 3: 복사하기 + .md 다운로드

- **담당 시나리오**: Scenario 4, 5
- **크기**: S (1 구현 파일)
- **의존성**: Task 2 (결과 Markdown 접근)
- **참조**:
  - shadcn — styling
  - wireframe: 변환 결과 화면 Row 1 (복사하기, .md 다운로드 버튼)
- **구현 대상**:
  - `components/url-to-md/ExportButtons.tsx` + `ExportButtons.test.tsx`
- **수용 기준**:
  - [ ] 복사하기 버튼 클릭 후 버튼 레이블이 "복사됨!"으로 변한다
  - [ ] 클립보드 내용이 결과 영역의 Markdown과 동일하다 (프롬프트 미포함)
  - [ ] 다운로드 시 생성되는 앵커의 href가 `.md` 확장자를 포함한다
  - [ ] 다운로드 파일 내용이 결과 영역의 Markdown과 동일하다 (프롬프트 미포함)
- **검증**:
  - `bun run test -- ExportButtons`

---

### Task 4: 프롬프트 선택 + LLM 열기

- **담당 시나리오**: Scenario 6, 7, 8
- **크기**: M (3 구현 파일)
- **의존성**: Task 2 (결과 Markdown), Task 3 (export 버튼과 동일 toolbar에 통합)
- **참조**:
  - shadcn — composition (DropdownMenu 그룹 규칙), styling
  - wireframe: 변환 결과 Row 2 (프롬프트 칩, 열기 드롭다운), 직접 입력 화면
- **구현 대상**:
  - `lib/llm-url.ts` + `lib/llm-url.test.ts`
  - `components/url-to-md/PromptSelector.tsx` + `PromptSelector.test.tsx`
  - `components/url-to-md/LLMOpenDropdown.tsx` + `LLMOpenDropdown.test.tsx`
- **수용 기준**:
  - [ ] "직접 입력" 칩 선택 시 텍스트 입력창이 나타난다
  - [ ] 프롬프트 없음 상태에서 ChatGPT로 열기 클릭 시 `window.open`이 `chatgpt.com` URL로 호출된다
  - [ ] 프롬프트 없음 상태에서 열기 클릭 시 클립보드에 Markdown 내용이 복사된다
  - [ ] 프리셋 프롬프트 선택 후 열기 시 클립보드 내용이 `[프롬프트 텍스트]\n\n[Markdown]` 형식이다
  - [ ] 커스텀 프롬프트 입력 후 열기 시 클립보드 내용이 `[커스텀 텍스트]\n\n[Markdown]` 형식이다
  - [ ] 다음 변환 완료 후 직접 입력창이 비어 있다
  - [ ] 현재 탭 상태는 열기 버튼 클릭 후 변하지 않는다 (Scenario 6, 7, 8 공통)
- **검증**:
  - `bun run test -- llm-url`
  - `bun run test -- PromptSelector`
  - `bun run test -- LLMOpenDropdown`
  - Browser MCP — 열기 클릭 후 새 탭 URL이 `chatgpt.com` / `claude.ai`인지 확인 + 클립보드 내용 검증; 증거 `artifacts/url-to-md/evidence/task-4-llm-open.png` 저장

---

### Task 5: 다크모드 자동 적용

- **담당 시나리오**: 불변 규칙 (다크모드)
- **크기**: S (1 파일 수정)
- **의존성**: None (독립적; 모든 컴포넌트가 semantic token 사용하므로 수정 없이 동작)
- **참조**:
  - shadcn — styling (no manual `dark:` color overrides)
- **구현 대상**:
  - `app/layout.tsx` (modify: ThemeProvider에 `defaultTheme="system"` + `enableSystem` 추가)
- **수용 기준**:
  - [ ] OS 다크모드 활성화 시 앱 배경·텍스트가 다크 테마로 자동 전환된다
  - [ ] 앱 내 별도 다크/라이트 토글이 존재하지 않는다
- **검증**:
  - Human review — OS 다크모드 ON/OFF 전환 후 브라우저 새로고침으로 테마 변경 확인; 증거 `artifacts/url-to-md/evidence/task-5-darkmode.png` 저장

---

### Checkpoint: Tasks 3–5 이후
- [ ] 모든 테스트 통과: `bun run test`
- [ ] 빌드 성공: `bun run build`
- [ ] Browser MCP — 복사하기 클릭 후 버튼 레이블 확인; 프롬프트 칩 선택 후 열기 드롭다운 확인; 다크모드 시각 확인; 증거 `artifacts/url-to-md/evidence/checkpoint-2.png` 저장

---

## 미결정 항목

없음
