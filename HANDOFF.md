# HANDOFF — 작업 인수인계 문서

> 이 문서는 **VPP 세션**(이어받기 세션)의 단일 진실 공급원(Single Source of Truth)입니다.
> 새 세션을 시작할 때 가장 먼저 읽고, 작업을 끝낼 때 마지막으로 갱신하세요.
> "VPP 세션"의 정의와 규칙은 `CLAUDE.md` 를 참고하세요.

---

## 1. 프로젝트 개요

- **이름**: 빛 그림 프로젝트 — 더블앵커 (Double Anchors)
- **성격**: 청소년 진로체험 예술교육 프로그램 소개용 **단일 파일 정적 랜딩 페이지**
- **저장소**: `leeoutheater/Double-Anchors_Landing-Page`
- **유일한 산출물**: `index.html` (약 1,431줄, HTML + 인라인 `<style>` + 인라인 `<script>`)
- **빌드 단계 없음**: 번들러·패키지 매니저·서버 불필요. 브라우저로 `index.html`을 열면 그대로 동작.
- **외부 의존성**: Pretendard 웹폰트 CDN 한 개뿐 (`cdn.jsdelivr.net`). 그 외 모든 JS/CSS는 인라인.

## 2. 기술 구성 (index.html 내부 구조)

| 영역 | 위치(대략) | 설명 |
|---|---|---|
| `<head>` / CSS 변수 | 1–460 | 색상 변수(`--bg-color #1a1a24`, `--point-color #f3c623` 등), 전역 스타일 |
| 본문 섹션 | 526–911 | `#about` `#program` `#experts` `#archive` `#faq` `#performance` |
| `<script>` | 913–1431 | 단일 스크립트 블록. 아래 기능 전부 포함 |

### 주요 자바스크립트 기능
- **다국어(i18n)**: `i18n` 사전 객체(`ko`/`en`/`ru`/`zh`/`vi` 5개 언어, 약 916–1133줄) + `changeLang(lang)`(1137줄). DOM의 `data-i18n="키"` 요소(95곳)를 순회하며 `innerHTML` 교체. 기본값 `currentLang='ko'`.
- **아카이브 영상 멀티소스 플레이어**(1180–1283줄): YouTube ID별로 대체 소스를 선언해 우선순위대로 재생.
  1. `mp4Url` → HTML5 `<video>` 직접 스트리밍 (가장 우선)
  2. `gdriveId` → Google Drive `/preview` iframe (AVI 등 미지원 포맷 자동 트랜스코딩)
  3. 기본 → `youtube-nocookie.com` iframe fallback
  - 설정은 `window.ARCHIVE_VIDEOS` 객체에 등록. 현재 `OdYwTbwq1GM`(송송뽕뽕 공연)은 GitHub Raw의 MP4로 직접 재생.
  - `IntersectionObserver`로 아카이브 섹션이 뷰포트 300px 이내 접근 시 자동 로드(음소거+루프). 미지원 시 즉시 로드.
- **소개 영상 토글**: `toggleAboutVideo()` (1149줄), `#aboutVideo`.
- **모달**: `openModal()`/`closeModal()` (1162줄~), 배경 클릭 시 닫힘.
- **FAQ 아코디언**: `toggleFaq()` (1173줄).
- **카드뉴스 캔버스 드로잉**: `drawAllCanvas()` 외 (1285줄~, `W=1080 H=1350`), 색상 팔레트 `C` 객체. 언어 변경 시 캔버스도 다시 그림.

## 3. 현재 상태 (DONE)

최근 커밋 기준 완료된 작업:
- ✅ 다중 소스 영상 플레이어 + `OdYwTbwq1GM` AVI→MP4 변환본 적용
- ✅ 아카이브 영상 자동 로드·재생 (음소거 + 루프, IntersectionObserver 기반)
- ✅ 아카이브 영상 클릭 시 인라인 재생 (YouTube iframe 임베드)
- ✅ 아카이브 다국어 번역 + NADA 영상 미리보기 정상화 + 독립 실행 안정성 보강
- ✅ 접수 마감 수정(5.15 → 6.25), 일일 진로체험 방식 (5개 언어 + 캔버스 카드 동기화)
- ✅ 전작 아카이브 타임라인 재설계: 2024 나다 → 2025 숲숲학교 → 2025 아단 공존의 지혜 → 2026 더블앵커

## 4. 남은 작업 / 알려진 이슈 (TODO)

> 다음 세션이 이어받을 항목. 새 항목이 생기면 여기에 추가하고, 끝내면 §3으로 옮기세요.

- [ ] (현재 명시적으로 접수된 미해결 작업 없음 — 새 요청을 받으면 여기에 기록)
- [ ] 외부 미디어 링크(YouTube/Drive/GitHub Raw) 유효성 정기 점검 — 링크 깨짐 시 영상 미재생
- [ ] 5개 언어 번역 키 누락 여부 점검 (`data-i18n` 95곳 ↔ 각 언어 사전 키 일치 확인)

## 5. 실행 / 검증 방법

빌드·테스트 프레임워크가 없으므로 **수동 검증**합니다.

```bash
# 1) 로컬 미리보기 (둘 중 택1)
python3 -m http.server 8000      # → http://localhost:8000/index.html
#   또는 브라우저로 index.html 직접 열기

# 2) 기본 정합성 점검 (에러 없이 끝내기 위한 최소 확인)
grep -c '<script' index.html                      # 스크립트 블록 1개 유지 확인
grep -oE 'data-i18n="[^"]*"' index.html | sort -u  # 사용 중인 i18n 키 목록
```

검증 체크리스트(수정 후 반드시 확인):
1. 페이지가 콘솔 에러 없이 로드되는가 (브라우저 DevTools Console)
2. 언어 전환 버튼(🇰🇷/🇺🇸/Русский/🇨🇳/🇻🇳) 클릭 시 모든 텍스트가 바뀌는가
3. 아카이브 섹션 스크롤 시 영상이 자동 재생되는가, 클릭 시 소리와 함께 재생되는가
4. 모달 열기/닫기, FAQ 아코디언 정상 동작
5. 카드뉴스 캔버스가 언어별로 다시 그려지는가

## 6. 인수인계 로그 (Handoff Log)

> 세션을 끝낼 때마다 **맨 위에** 한 줄 추가. 다음 세션이 맥락을 즉시 파악할 수 있게.

- **2026-06-03** — HANDOFF.md / CLAUDE.md 신설. "VPP 세션"(HANDOFF 기반 이어받기) 워크플로 정립. 작업 산출물·구조·검증법 문서화. 기능 변경 없음(문서만 추가).
