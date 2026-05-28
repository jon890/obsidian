# Obsidian Vault — Claude Code 작업 규칙

이 vault 는 **간단한 메모 작성 공간**이다.
일지·아이디어·업무 기록을 자유롭게 남긴다.

컴파일된 지식 그래프(brain)는 별도 저장소(`~/personal/fos-brain`)에서 관리한다.
이 vault 의 메모 중 brain 에 통합할 가치가 있는 것은 `brain-add` skill 로 fos-brain 에 가져간다.

## 디렉터리 역할

- `YYYY년 메모/` — 연도별 메모·일지. 사용자가 자유롭게 작성. LLM 은 사용자 요청 없이 수정하지 않는다.
- `Hello Obsidian.md` — vault 안내 노트.

## 작업 원칙

1. **메모는 사용자 소유**: LLM 은 사용자가 명시 요청할 때만 메모를 편집한다.
2. **brain 통합은 brain-add 로**: 메모를 지식 그래프에 넣으려면 `brain-add` skill 을 호출한다.
   - brain-add 가 메모를 `fos-brain/raw/` 로 가져가 `wiki/` 로 컴파일한다.
   - 이 vault 의 메모 원본은 그대로 둔다(brain-add 는 복사만 한다).

## brain 관련 skill

전역 skill(`~/.claude/skills/`). 대상은 모두 `~/personal/fos-brain`:

- `brain-add` — 소스(메모·URL·PDF 등)를 brain 으로 가져와 컴파일
- `brain-search` — brain 지식 기반에 질문
- `brain-lint` — brain 무결성 점검

## 금지 사항

- `YYYY년 메모/` 메모를 사용자 요청 없이 수정·삭제
- 이 vault 에 wiki 디렉터리를 다시 만들기(지식 그래프는 fos-brain 소관)
