# Obsidian Vault — Claude Code 작업 규칙

이 vault 는 Karpathy 스타일 LLM 지식 기반이다. 원본 (`raw/`) 은 사용자가 수집하고, 위키 (`wiki/`) 는 Claude Code 가 컴파일·유지한다. 사용자는 Obsidian 을 뷰어로 사용하며 wiki 를 거의 직접 편집하지 않는다.

## 디렉터리 역할

- `raw/web/`, `raw/papers/`, `raw/repos/`, `raw/notes/` — **원본**. LLM 은 읽기 전용으로 취급한다. 수정·삭제 금지 (사용자 명시 지시 예외).
- `wiki/INDEX.md` — 전체 목차 + 한 줄 요약. 모든 ingest/lint 후 최신 상태로 유지.
- `wiki/concepts/` — 개념 단위 페이지. 백링크 의무.
- `wiki/topics/` — 여러 개념을 묶는 상위 페이지.
- `YYYY년 메모/` — 사용자의 기존 일지. 손대지 않는다. 일지에서 주제가 무르익으면 사용자 요청 시 `raw/notes/` 로 승격.

## 작업 원칙

1. **사용자 편집 최소화**: wiki 의 모든 변경은 LLM 책임. 사용자가 wiki 를 직접 고친 흔적이 보이면 덮어쓰기 전에 확인.
2. **백링크 의무**: 새 concept 페이지를 만들면 (a) `INDEX.md` 에 등록, (b) 관련 다른 concept 에 양방향 링크 추가, (c) 어떤 `raw/` 출처에서 왔는지 페이지 하단 "Sources" 섹션에 기록.
3. **raw 는 출처**: wiki 의 주장은 raw 로 추적 가능해야 한다. 출처 없는 주장 금지.
4. **점진적 컴파일**: 한 번에 raw 전체를 처리하지 않는다. 새 raw 파일 또는 사용자가 지정한 범위만 ingest.
5. **lint 는 별도 호출**: 깨진 링크·고아 노트·중복 개념 점검은 사용자가 lint 를 명시 요청할 때만 실행.

## 페이지 스키마

### `wiki/concepts/<개념명>.md`

```markdown
---
type: concept
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# <개념명>

한 줄 정의.

## 핵심 포인트

- ...

## 관련 개념

- [[다른-개념]] — 관계 설명

## Sources

- [[../raw/papers/원본파일.md]]
- [[../raw/web/기사.md]]
```

### `wiki/topics/<주제명>.md`

여러 concept 을 묶는 narrative. 같은 frontmatter + "Concepts" 섹션에 `[[concept]]` 나열.

## 워크플로우 진입점 (Skill)

본 vault 에서 사용하는 skill (`~/.claude/skills/`):

- `obsidian-wiki-ingest` — `raw/` 의 새 파일을 wiki 로 컴파일
- `obsidian-wiki-query` — INDEX → concepts → raw 순으로 답변, 결과는 wiki 로 환원
- `obsidian-wiki-lint` — 9개 항목 (백링크/고아/중복/Sources/frontmatter/INDEX 동기화/`~` 함정/모순/누락 교차 참조)

모든 skill 은 `wiki/log.md` 에 append-only 로 활동 기록을 남긴다.

## 검색 도구: qmd

규모가 커지면 `grep` 으로는 한계가 있다. Karpathy 가 권장한 qmd (BM25 + 벡터 + LLM rerank) 를 설치해 두었다.

- 등록 컬렉션: `obsidian-wiki` (vault/wiki), `obsidian-raw` (vault/raw)
- 1차 검색 (BM25): `qmd search "<keyword>" -c obsidian-wiki`
- 의미 검색 (벡터): `qmd vsearch "<text>" -c obsidian-wiki`
- 하이브리드 + rerank (권장): `qmd query "<question>"`
- 인덱스 갱신: `qmd update` (파일 변경 후), `qmd embed` (임베딩 재생성)
- 상태 점검: `qmd status`

`obsidian-wiki-query` skill 은 wiki 가 일정 규모 이상이면 grep 대신 `qmd query` 를 1차 검색으로 사용한다.

## Obsidian 운영 가이드

vault 를 더 잘 활용하려면 Obsidian 자체 기능·플러그인을 활용한다.

- **Graph View** (기본 기능, 좌측 사이드바 아이콘 또는 `Cmd+G`)
  - wiki 의 백링크 그래프를 시각적으로 점검. 고아 노트·약한 연결 즉시 발견.
  - `concepts/`, `topics/` 폴더별 필터링 가능.
- **Dataview 플러그인** (커뮤니티 플러그인)
  - frontmatter 기반 동적 쿼리. 예: 모든 concept 페이지 자동 목록:
    ```dataview
    LIST FROM "wiki/concepts" WHERE type = "concept" SORT updated DESC
    ```
  - INDEX.md 를 일부 자동 생성하는 데 활용 가능.
- **Marp 플러그인** (또는 Marp CLI)
  - wiki 페이지를 슬라이드로 출력. 발표용 정리에 유용.
- **Obsidian Web Clipper** (브라우저 확장)
  - 웹 기사를 `raw/web/` 으로 직접 클리핑.
  - Settings → Files and links 에서 첨부 폴더 지정 후 `Ctrl+Shift+D` (macOS: `Cmd+Shift+D`) 로 이미지를 로컬에 자동 다운로드.
  - 클리핑한 기사에는 ingest skill 이 메타데이터(URL, 클리핑 날짜) 를 읽고 Sources 에 자동 기록.

## Future Work (Karpathy gist 미적용 항목)

낮은 우선순위로 보류 중인 확장. wiki 가 충분히 커지면 검토.

- **합성 데이터 생성** — 기존 wiki 로부터 Q&A 페어 합성. 미세조정 (fine-tuning) 데이터셋 후보.
- **로컬 모델 파인튜닝** — vault 도메인 특화 모델. wiki 가 40만 단어 + 100 페이지 이상일 때 ROI 검토.
- **자체 web UI / CLI 검색 인터페이스** — qmd 위에 vault 전용 UI. 현재는 `qmd query` CLI 직접 사용.
- **Marp 자동화** — `obsidian-wiki-query` 결과를 슬라이드로 자동 변환하는 출력 모드.

## 금지 사항

- `raw/` 파일 자동 수정·삭제
- wiki 페이지 일괄 재작성 (사용자 요청 시 외)
- `YYYY년 메모/` 일지 수정
