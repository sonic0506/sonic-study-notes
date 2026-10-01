---
name: add-note
description: inbox/에 있는 학습 노트 초안을 지식베이스(notes/)에 추가한다. AI 티 제거(humanizer), 프론트매터 정규화, 기존 문서와의 선행/관련 링크 연결을 자동 처리하고 커밋은 하지 않는다.
argument-hint: <inbox/파일.md>
disable-model-invocation: true
---

# /add-note

입력: `$ARGUMENTS` (inbox/ 안의 초안 하나). 설계: `docs/superpowers/specs/2026-10-01-knowledge-graph-notes-design.md`

아래 순서를 지킨다. 각 단계의 결과는 마지막 보고에 모은다.

## 0. 사전 확인 — 하나라도 어긋나면 아무것도 고치지 말고 중단, 이유 보고
- 저장소 루트에서 실행 중이고 `git rev-parse` 가 성공한다.
- 입력 파일이 존재하고 경로가 `inbox/` 아래다.

## 1. 원본 보관
```bash
git add "<입력 파일>"
```
원본이 index에 남아 이후 모든 변경이 `git diff` 로 보인다. 이 뒤로 `git add` 하지 않는다.

## 2. 정규화
**slug(파일명) 결정**
1. 미작성 링크 목록을 구한다:
   ```bash
   grep -ohE '\[\[[^]|#]+' notes/*.md | sed 's/\[\[//' | sort -u   # 링크 대상
   ls notes | sed 's/\.md$//'                                     # 존재하는 문서
   ```
   앞 목록에만 있는 이름이 미작성 링크다. 그중 초안이 다루는 개념과 같은 것이 있으면 **그 이름을 slug로 쓴다.**
2. 없으면 초안의 핵심 개념을 영문 소문자 kebab-case로 짓는다 (예: `docker-compose`, `tcp-handshake`).
3. `notes/<slug>.md` 가 이미 있으면 **중단**하고 보고한다. 합칠지는 사용자가 정한다.

**프론트매터** — 초안에 이미 있는 값은 유지하고 빈 항목만 채운다. 키 순서는 아래와 같다.
```yaml
---
aliases: [영문 제목, 한글 제목]
tags: [소문자, 분야]
prerequisites: []
related: []
status: draft
created: YYYY-MM-DD   # 오늘 날짜
---
```
tags는 기존 notes/에서 쓰는 태그를 우선 재사용한다.

## 3. 휴머나이즈
`humanizer:humanizer` 스킬로 본문 산문을 다듬어 파일에 덮어쓴다.
- 건드리지 않는 것: 프론트매터, 코드 블록과 인라인 코드, 명령어, 기술 용어, 기존 `[[링크]]`, 헤딩 구조
- 사실과 의미를 바꾸지 않는다. 내용을 추가하지 않는다.
- 초안의 문체(해요체/합니다체/한다체)를 유지한다.
- 수정한 문장 수를 센다.

## 4. 연결 분석
`notes/*.md` 각각의 프론트매터와 `#` 헤딩을 읽는다. 관계가 애매한 문서만 본문까지 읽는다.

판단 기준:
- **prerequisites**: 이 글을 이해하려면 그 문서 내용을 먼저 알아야 한다.
- **related**: 대등하게 함께 보면 좋다 (비교 대상, 같은 문제를 다른 방식으로 푸는 것 등).
- 단어가 지나가듯 한 번 나온 것은 연결하지 않는다. 확신이 없으면 넣지 않는다.
- 새 글이 **기존 문서의 선행 지식**인지도 반대 방향으로 확인한다.
- 다뤄야 하는데 문서가 없는 핵심 개념은 미작성 링크 후보로 둔다 (한 글에 3개 이하).

## 5. 연결 적용
링크는 반드시 따옴표 붙은 YAML 리스트로 쓴다:
```yaml
prerequisites:
  - "[[docker]]"
```
빈 리스트는 `[]` 로 둔다.

- **새 글**: `prerequisites`, `related` 를 채운다. related는 새 글에만 적는다 (반대쪽은 백링크로 보인다).
- **기존 문서**: 새 글이 그 문서의 선행 지식이면 그 문서의 `prerequisites` 에 `"[[<slug>]]"` 를 추가한다. 프론트매터의 그 항목만 고치고 본문과 다른 키는 건드리지 않는다.
- **새 글 본문**: 연결한 기존 문서 개념이 처음 나오는 곳 한 번만 `[[slug|원래 표현]]` 으로 바꾼다. 코드 블록·인라인 코드·헤딩 안은 바꾸지 않는다. 미작성 링크 후보도 같은 방식으로 본문에 건다.
- **순환 검사**: 선행 관계 A→B(A의 prerequisites에 B)를 추가하기 전에, B에서 시작해 prerequisites를 따라갔을 때 A에 닿으면 추가하지 않고 보고에 적는다.

## 6. 이동
```bash
git mv "<입력 파일>" "notes/<slug>.md"
```

## 7. 확인 후 보고
```bash
grep -rnE '^\s*-\s*\[\[' notes/   # 출력이 없어야 한다 (따옴표 없는 링크 검사)
git status --short
```
보고 형식:
```
✔ inbox/<원래 이름> → notes/<slug>.md
  휴머나이즈: N문장 수정
  prerequisites: [[a]], [[b]]
  related: [[c]]
  기존 문서 수정: notes/x.md (prerequisites += [[<slug>]])
  새 미작성 링크: [[d]]
  건너뛴 순환: 없음
검토: git diff   /   되돌리기: git restore --staged --worktree . && git mv notes/<slug>.md "<입력 파일>"
```

## 하지 말 것
- 커밋하지 않는다.
- 기존 문서의 본문을 고치지 않는다.
- 따옴표 없는 `[[링크]]` 를 YAML에 쓰지 않는다.
- inbox/ 밖의 파일을 입력으로 받지 않는다.
