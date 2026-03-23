# 한영 토글 기능 구현 계획

## 개요
단일 `index.html` 파일(2,593줄)에 한영 토글 기능을 추가합니다.
현재 모든 텍스트가 한국어로 하드코딩되어 있으며, i18n 인프라가 없습니다.

---

## 구현 전략

**핵심 원칙**: 사용자가 직접 편집하는 콘텐츠(이름, 회사명, 경력 내용 등)는 번역 대상이 아닙니다.
번역 대상은 **UI 요소**(버튼, 툴팁, 힌트, 토스트 메시지, placeholder 등)와 **기본 템플릿 텍스트**입니다.

### 번역 범위 분류

#### A. UI 문자열 (JS에서 동적 전환)
- 편집 모드 버튼: "✏ 편집 모드" / "✓ 편집 완료"
- 툴바 라벨, 힌트, 버튼 텍스트
- 토스트 메시지 (삭제됨, 추가됨, 저장 완료 등)
- confirm/alert 다이얼로그 메시지
- 색상 팔레트 이름
- 태그 색상 라벨
- 드래그 핸들 title 속성
- `data-hint` 속성 (편집 모드 툴팁)

#### B. HTML 정적 콘텐츠 (기본 템플릿 텍스트)
- 섹션 제목: "경력", "보유 역량 & 도구", "업무 스타일"
- 기본 placeholder 텍스트: "이 름", "핵심 역할 · 전문 분야 · 키워드" 등
- 인쇄/PDF 버튼 텍스트

---

## 구현 단계

### Step 1: i18n 사전(Dictionary) 작성
`<script>` 내에 `I18N` 객체를 추가합니다.

```javascript
const I18N = {
  ko: {
    // UI strings
    editMode: '✏ 편집 모드',
    editDone: '✓ 편집 완료',
    editEnd: '편집 종료',
    editHint: '텍스트 클릭 후 수정 · 항목 드래그로 순서 변경',
    dirty: '● 변경됨',
    // ... 180+ strings
  },
  en: {
    editMode: '✏ Edit Mode',
    editDone: '✓ Done Editing',
    editEnd: 'End Edit',
    editHint: 'Click text to edit · Drag items to reorder',
    dirty: '● Modified',
    // ... 영문 번역
  }
};
```

### Step 2: 언어 상태 관리
```javascript
let currentLang = localStorage.getItem('resume_lang') || 'ko';
```

### Step 3: 토글 버튼 UI 추가
- 우측 하단 버튼 그룹에 한/EN 토글 버튼 추가
- 현재 언어 표시, 클릭 시 전환
- 인쇄 시 숨김 (`.no-print`)

### Step 4: `applyLanguage()` 함수 구현
언어 전환 시 실행되는 핵심 함수:

1. **`data-i18n` 속성 기반 DOM 텍스트 교체**
   - 번역 대상 HTML 요소에 `data-i18n="key"` 속성 추가
   - `applyLanguage()` 호출 시 해당 요소의 `textContent`를 사전에서 조회하여 교체

2. **`data-hint` 속성 교체**
   - 각 hint에 대응하는 i18n 키를 `data-hint-i18n="key"` 속성으로 지정
   - 전환 시 `data-hint` 값을 사전에서 조회하여 교체

3. **JS 동적 문자열 교체**
   - `t('key')` 헬퍼 함수: `I18N[currentLang][key]`를 반환
   - 토스트, confirm, 버튼 생성 등에서 하드코딩 문자열을 `t()` 호출로 교체

### Step 5: 기존 코드 수정 (핵심 주의 사항)

#### 수정이 필요한 함수들:
| 함수 | 수정 내용 |
|------|-----------|
| `toggleEdit()` | 버튼 텍스트를 `t()` 호출로 변경 |
| `initBulletControls()` | "+ 항목 추가", "새 항목을 입력하세요." → `t()` |
| `initChipControls()` | "+ 추가", "새 태그" → `t()` |
| `initCareerControls()` | "+ Phase 추가", "+ 경력 추가", 토스트 메시지 → `t()` |
| `initSkillControls()` | "+ 항목 추가", "+ 그룹 추가", 토스트 메시지 → `t()` |
| `initContactControls()` | "+ 항목 추가" → `t()` |
| `initNoteControls()` | "+ 문단 추가" → `t()` |
| `initTagAddBtn()` | "새 태그" → `t()` |
| `openColorPalette()` | 색상 이름, "텍스트 색상", "초기화", "기본값" → `t()` |
| `createNewPhase()` | "담당 업무", "업무 내용을 입력하세요." → `t()` |
| `createNewCompany()` | "회사명", "업종 / 산업" 등 → `t()` |
| `saveJSON()` / `loadJSON()` | 토스트/alert 메시지 → `t()` |
| `performUndo()` / `performRedo()` | 토스트 메시지 → `t()` |
| `showUndoToast()` | "실행취소" 버튼 → `t()` |
| `getCleanHTMLString()` | "인쇄 / PDF 저장", meta description → `t()` |
| `initTagColorChange()` | "태그 삭제" title → `t()` |
| `PALETTE_COLORS` | 색상 이름을 함수로 교체 |
| `TAG_COLORS` | 태그 라벨을 함수로 교체 |
| `initCareerControls()` 내 섹션 탐색 | `'경력'` 하드코딩 → 양쪽 언어 모두 매칭 |
| `initSkillControls()` 내 섹션 탐색 | `'보유 역량'` 하드코딩 → 양쪽 언어 모두 매칭 |

### Step 6: `<html lang>` 속성 동적 변경
```javascript
document.documentElement.lang = currentLang === 'ko' ? 'ko' : 'en';
```

### Step 7: `<title>` 동적 변경
```javascript
document.title = t('pageTitle');
```

---

## 기존 기능 보호 체크리스트

1. **편집 모드 진입/종료** — `toggleEdit()` 수정 후 정상 작동 확인
2. **Undo/Redo** — `getPageContentSnapshot()` 및 `restoreSnapshot()` 영향 없음 확인
3. **드래그 & 드롭** — 태그/칩/회사/Phase 드래그 정상 동작
4. **JSON 저장/불러오기** — 언어 설정도 JSON에 포함하여 저장
5. **HTML 저장** — 현재 언어 기준 clean HTML 생성
6. **인쇄/PDF** — 토글 버튼 숨김, 텍스트 정상 출력
7. **색상 팔레트** — 팔레트 이름 번역 반영
8. **태그 색상 변경** — 팝업 정상 동작
9. **contenteditable** — 편집 가능 요소 정상 동작
10. **URL 자동 링크** — 영향 없음
11. **키보드 단축키** — Ctrl+S, Ctrl+Z 정상
12. **initCareerControls 섹션 탐색** — 섹션 제목이 바뀌어도 정상 탐색 (`data-section` 속성 추가)
13. **initSkillControls 섹션 탐색** — 동일하게 `data-section` 속성 기반으로 변경

---

## 위험 요소 및 대응

| 위험 | 대응 |
|------|------|
| 섹션 탐색이 한국어 텍스트에 의존 | `data-section="career"` 등 속성 기반으로 변경 |
| 편집 중 언어 전환 시 데이터 손실 | 편집 모드 중 토글 비활성화 또는 경고 |
| Undo 스냅샷에 언어 전환이 기록됨 | 언어 전환 시 undo 히스토리 유지 (구조 변경 아님) |
| 사용자가 편집한 콘텐츠가 덮어써짐 | 기본 템플릿 텍스트만 `data-i18n`으로 표시, 사용자 편집 시 속성 제거 |

---

## 영문 번역 기준

- **UI 버튼/라벨**: 간결하고 관용적인 영문 표현 사용
- **data-hint (가이드 텍스트)**: 영문 이력서/CV 작성 맥락에 맞게 윤문
- **기본 템플릿 텍스트**: 영문 이력서 관행에 맞는 자연스러운 표현
- 예시:
  - "경력" → "Experience"
  - "보유 역량 & 도구" → "Skills & Tools"
  - "업무 스타일" → "Work Style"
  - "성함을 입력하세요" → "Enter your name (e.g., John Doe)"
  - "핵심 역량 태그 — 우클릭으로 색상 변경" → "Core competency tag — right-click to change color, drag to reorder"

---

## 파일 변경 범위

**변경 파일**: `index.html` (단일 파일)

1. HTML `<body>` 내: `data-i18n`, `data-hint-i18n`, `data-section` 속성 추가 + 토글 버튼 HTML 추가
2. CSS `<style>` 내: 토글 버튼 스타일 추가
3. JS `<script>` 내: i18n 사전, `t()` 함수, `applyLanguage()`, 기존 함수 문자열 교체
