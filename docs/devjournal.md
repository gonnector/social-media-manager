# Development Journal

## 2026-03-22 — Session 1: Feasibility Test

### 목표
Threads, X, LinkedIn에 동일 콘텐츠의 영문 번역본을 자동 포스팅할 수 있는지 검증

### 수행한 작업
1. **콘텐츠 수집**: Threads/X/LinkedIn에서 한글 원본 스크래핑 (Firecrawl 차단 → 브라우저 자동화로 전환)
2. **번역**: 12개 Threads 포스트 + X 단일 포스트 + LinkedIn Article/Post 영문 번역
3. **Threads 500자 제한 대응**: 영문 번역이 원문보다 길어 500자 초과 → 서브에이전트로 12개 포스트 모두 500자 이내 압축
4. **포스팅 자동화 테스트**: 텍스트 입력, 이미지 첨부, 스레드 체인 구성

### 핵심 발견

#### 텍스트 입력
- `execCommand('insertText')`: 줄바꿈(`\n`) 무시 → 한 덩어리로 들어감. 사용 불가
- **클립보드 텍스트 붙여넣기 (PowerShell `SetText` → Ctrl+V)**: 줄바꿈 완벽 보존. **채택**

#### 이미지 첨부 — 핵심 병목
- 네이티브 OS 파일 선택기가 브라우저 자동화 범위 밖
- 클립보드 이미지 붙여넣기: 1장 성공, 추가 시 교체됨 (Threads)
- DataTransfer + file input: 메커니즘은 작동하나 대량 바이너리 전달이 병목
- CORS 로컬 HTTP 서버: CSP 차단
- base64 JS 주입: 도구 파라미터 크기 한계

#### 플랫폼별 특성
| 플랫폼 | 글자 제한 | 이미지 | 스크래핑 | 포스팅 API |
|--------|---------|--------|---------|-----------|
| Threads | 500자/포스트 | 10장, 체인 가능 | Firecrawl 차단, 브라우저 필요 | 없음 |
| X | 25,000자 (Premium) | 4장 | Firecrawl 차단, 브라우저 필요 | 유료 |
| LinkedIn | 무제한 (Post) | 다수 | Firecrawl 차단, 브라우저 필요 | Article 작성 API 없음 |

### 결론
- 텍스트 자동화: **가능** (클립보드 방식)
- 이미지 자동화: **현재 불가** → 하이브리드(텍스트 자동 + 이미지 수동) 채택
- 향후 해결 경로: Playwright MCP `setInputFiles()`, 데스크톱 자동화 MCP, Computer Use

### 다음 단계
- [ ] Playwright MCP `browser_file_upload` 테스트 (이미지 자동 첨부)
- [ ] 포스팅 워크플로우 스크립트화 (현재 수동 반복 → 자동화)
- [ ] 콘텐츠 수집 → 번역 → 포맷팅 → 포스팅 파이프라인 설계
- [ ] 플랫폼별 어댑터 패턴 구현
