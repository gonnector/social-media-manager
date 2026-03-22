# Browser Automation for SNS Posting — Feasibility Research

**Date**: 2026-03-22
**Author**: JARVIS
**Session**: social-media-manager initial feasibility test

## 1. Background

Claude Code의 claude-in-chrome MCP를 사용하여 Threads, X, LinkedIn에 자동 포스팅하는 것이 가능한지 검증. 특히 텍스트 입력, 이미지 첨부, 멀티 포스트 스레드 구성의 자동화 가능 여부를 확인.

## 2. Text Input Methods

### 2.1 `document.execCommand('insertText')`

```javascript
const editor = document.querySelector('[contenteditable="true"][role="textbox"]');
editor.focus();
document.execCommand('insertText', false, text);
```

**결과**: 텍스트는 삽입되나 `\n` 줄바꿈이 무시됨. 모든 단락이 한 덩어리로 합쳐짐.
**원인**: contenteditable div에서 `insertText`는 `\n`을 무시하고 텍스트만 삽입.
**판정**: ❌ 사용 불가 (가독성 치명적)

### 2.2 Clipboard Text Paste (PowerShell → Ctrl+V)

```powershell
powershell -Command "
  Add-Type -AssemblyName System.Windows.Forms
  [System.Windows.Forms.Clipboard]::SetText('텍스트 내용')
"
# 이후 브라우저에서 Ctrl+V
```

**결과**: 줄바꿈 완벽 보존. 단락 구분 정상.
**원인**: 브라우저의 paste 이벤트 핸들러가 클립보드의 plain text를 그대로 contenteditable에 삽입하면서 `\n`을 `<br>` 또는 `<div>`로 변환.
**판정**: ✅ **채택** — 안정적이고 빠름

### 2.3 비교 요약

| 방식 | 줄바꿈 | 속도 | 안정성 | 판정 |
|------|--------|------|--------|------|
| `execCommand('insertText')` | ❌ | 빠름 | 높음 | 불가 |
| Clipboard `SetText` → Ctrl+V | ✅ | 빠름 | 높음 | **채택** |
| `type` action (char by char) | ✅ | 매우 느림 | 보통 | 비효율 |

## 3. Image Attachment — The Core Bottleneck

### 3.1 문제 구조

SNS 사이트의 이미지 업로드 흐름:
1. 숨겨진 `<input type="file" multiple>` 존재
2. UI의 이미지 아이콘 클릭 → 내부적으로 이 input을 click()
3. 브라우저가 **네이티브 OS 파일 선택기** 다이얼로그를 열음
4. 사용자가 파일 선택 → input의 files 속성에 세팅 → change 이벤트 발생

**병목**: 3단계의 네이티브 OS 파일 선택기는 브라우저 자동화(Chrome DevTools Protocol, Extension API) 범위 밖.

### 3.2 시도한 우회법

#### 3.2.1 Clipboard Image Paste

```powershell
powershell -Command "
  Add-Type -AssemblyName System.Windows.Forms
  Add-Type -AssemblyName System.Drawing
  [System.Windows.Forms.Clipboard]::SetImage(
    [System.Drawing.Image]::FromFile('path.png')
  )
"
# 이후 Ctrl+V
```

**결과**:
- Threads: 1장 성공, 이후 붙여넣기 시 기존 이미지 교체 (누적 불가)
- X: 이미지 clipboard paste 자체 미지원

**판정**: 단일 이미지만 가능. 다중 이미지 불가.

#### 3.2.2 DataTransfer + File Input (JavaScript)

```javascript
const b64 = 'iVBORw0KGgo...'; // base64 encoded image
const binary = atob(b64);
const bytes = new Uint8Array(binary.length);
for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
const file = new File([bytes], 'image.png', { type: 'image/png' });

const input = document.querySelector('input[type="file"]');
const dt = new DataTransfer();
dt.items.add(file);
input.files = dt.files;
input.dispatchEvent(new Event('change', { bubbles: true }));
```

**결과**: 1x1 테스트 이미지로 메커니즘 검증 성공 (Threads가 이미지 인식).
**제약**: 실제 이미지의 base64 데이터(23K~330K chars)를 JS 컨텍스트에 전달하는 것이 병목.
- javascript_tool 파라미터 크기 제한
- 10개 이미지 총 ~1.3MB base64

**판정**: 메커니즘은 유효하나 데이터 전달 문제 미해결.

#### 3.2.3 CORS Local HTTP Server

```python
# Python HTTP 서버로 이미지를 localhost:8765에서 서빙
class CORSHandler(SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header('Access-Control-Allow-Origin', '*')
        super().end_headers()
```

```javascript
// 브라우저에서 fetch
const resp = await fetch('http://127.0.0.1:8765/image.png');
// → Failed to fetch
```

**결과**: 실패
**원인**: SNS 사이트의 CSP(Content Security Policy)가 HTTPS → HTTP localhost fetch를 차단.
**판정**: ❌

#### 3.2.4 Base64 Direct Injection

base64 문자열을 javascript_tool의 text 파라미터에 직접 포함시키는 방식.

**결과**: 가장 작은 이미지도 23K chars → 도구 파라미터 크기 한계로 실패.
**판정**: ❌

### 3.3 해결 가능한 경로

| 경로 | 원리 | 실현 가능성 | 비고 |
|------|------|-----------|------|
| Playwright MCP `browser_file_upload` | `page.setInputFiles()` — 브라우저 레벨에서 파일 직접 세팅 | 높음 | Playwright 브라우저에서 로그인 필요 |
| Desktop Automation MCP | PyAutoGUI/AHK로 파일 선택기 직접 제어 | 높음 | MCP 서버 개발 필요 |
| Computer Use | Anthropic의 OS 레벨 데스크톱 제어 | 중간 | Claude Code에서 사용 불가 (API 전용) |
| chrome extension 개선 | upload_image 도구가 파일 경로를 직접 받도록 | 낮음 | Anthropic feature request |

## 4. Platform-Specific Findings

### 4.1 Threads

- **글자 제한**: 500자/포스트 (한글 원본 ~410자 → 영문 번역 ~700자 → 압축 필요)
- **스레드 체인**: "Add to thread"로 다음 포스트 추가. 작성 모달에서 최대 ~10개까지 한 번에 구성 가능
- **토픽**: "Add a topic" 드롭다운으로 설정 (예: "AI Threads")
- **이미지**: 최대 10장, 캐러셀로 표시. Attach Text 기능으로 긴 글 첨부 가능
- **스크래핑**: Firecrawl 차단, SPA라서 `get_page_text` 실패 → `read_page` (접근성 트리) 또는 `javascript_tool`로 DOM 직접 접근 필요
- **file input**: `accept="image/avif,image/jpeg,image/png,image/webp,video/mp4,video/quicktime,video/webm" multiple=true`

### 4.2 X (Twitter)

- **글자 제한**: 25,000자 (Premium), 280자 (무료)
- **이미지**: 최대 4장
- **텍스트 입력**: 클립보드 paste 정상 동작 (줄바꿈 보존)
- **이미지 paste**: 미지원 (텍스트 paste만 가능)
- **스크래핑**: Firecrawl 차단 → `[data-testid="tweetText"]` 셀렉터로 DOM 접근
- **file input**: `accept="image/jpeg,image/png,image/webp,image/gif,video/mp4,video/quicktime" multiple=true`

### 4.3 LinkedIn

- **글자 제한**: Post는 사실상 무제한, Article은 별도 에디터
- **구조**: Post (짧은 소개) + Article (긴 본문)을 분리 가능
- **텍스트 입력**: 클립보드 paste 정상 동작
- **기존 Article 활용**: 한글 Article을 그대로 두고 영문 Post에서 "Google Translate 사용하라"고 안내하는 전략 유효
- **스크래핑**: `.feed-shared-update-v2__description` 셀렉터로 포스트 텍스트, `/pulse/` URL로 Article 접근

## 5. Architecture Implications

### 5.1 확정된 파이프라인

```
콘텐츠 수집 (브라우저 자동화)
    ↓
번역 (Claude, 플랫폼별 글자 제한 고려)
    ↓
텍스트 포스팅 (PowerShell clipboard → Ctrl+V)
    ↓
이미지 첨부 (수동 하이브리드 or Playwright)
    ↓
게시 (Post 버튼 클릭)
```

### 5.2 해결해야 할 것

1. **이미지 자동 첨부**: Playwright MCP `browser_file_upload` 검증
2. **포스팅 반복 자동화**: 현재 포스트당 3회 도구 호출 (복사→포커스→붙여넣기) → 스크립트로 통합
3. **브라우저 확장 연결 안정성**: 장시간 세션에서 claude-in-chrome 연결 끊김 빈번
4. **Threads 체인 대안**: 12개 분리 포스트 대신 Post 1(소개+이미지) + Post 2(Attach Text로 풀 본문) 구조가 더 효율적

## 6. Conclusion

텍스트 자동화는 완전히 검증됨. 이미지 자동화가 핵심 미해결 과제이며, Playwright MCP가 가장 유력한 해결책. 단기적으로는 하이브리드(텍스트 자동 + 이미지 수동) 전략이 현실적.
