---
layout: single
title:  "GitHub Issues 관리 및 자동화 가이드"
date: 2025-04-03
tags: git
categories: 
   - Git
---

# GitHub Issues 관리 및 자동화 가이드

이 문서는 프로젝트에서 GitHub Issues를 체계적으로 활용하고, 로컬 Markdown 파일과 자동화 도구를 통해 이슈를 효율적으로 관리하는 방법을 안내합니다.

---

## 1. GitHub Issues 기본 객념

- GitHub Issue는 작업 항목, 버그, 기능 제안, 문서 요청 등을 기록하는 **할 일 카드**입니다.
- 프로젝트 진행 상황을 추적하고, 팀원 간 협업을 원화하게 합니다.

---

## 2. 이슈 분류 체계 및 라벨 예시

| 유형   | 라벨                     | 설명                             |
| ------ | ------------------------ | -------------------------------- |
| 버그 | `bug`                    | 기능 오류 또는 예제 생환         |
| 기능 | `feature`, `enhancement` | 새로운 기능 또는 개선 제안       |
| 작업 | `task`, `infra`          | 설정, 리파틱터링, 일반 작업      |
| 문서 | `documentation`          | README, 가이드, ICD 등 문서 정리 |
| 질문 | `question`               | 기술적 문의 또는 논의            |

**우선순위 라벨**: `priority: high`, `priority: medium`, `priority: low`
**상태 라벨**: `status: todo`, `status: in-progress`, `status: done`

> 이 라벨들은 GitHub 매 저장소에서 **무일정 생성**해야 사용할 수 있습니다.

---

## 3. 템플릿을 활용한 이슈 작성

`.github/ISSUE_TEMPLATE/` 폴더에 Markdown 파일로 템플릿을 정의하면, 이슈 생성 시 자동 양식이 제공됩니다.

### 예시: 버그 리포트 템플릿 (`bug_report.md`)
```markdown
---
name: 버그 리포트
about: 동작 오류나 시스템 문제를 보고합니다.
title: "[Bug] "
labels: bug
assignees: ''
---

### 요약
버그가 무엇인지 간단하게 설명해주세요.

### 재현 방법
1. 어느 동작을 했는지
2. 어느 현상이 발생했는지
3. 기대한 동작은 무엇인지

### 테스트 환경
- OS / 보드:
- 소프트웨어 버전 / 브랜치:

### 로그 / 스크립샵
(여부선)
```

> 이외에도 `feature_request.md`, `task.md` 등을 구성하여 역할별 이슈를 관리하세요.

---

## 4. 버그가 여러 개인 경우 처리 방법

### 원칙: 하나의 버그 = 하나의 이슈
- 동부적 추적, 다단자 지정, PR 연결이 쉽다

### 예외: 동일한 모듈에서 발생한 다수의 버그는 하나로 보고 가능
```markdown
### [다수 버그]
- [ ] RTSP 스트리밍 뫤충
- [ ] NFC 재불팅 후 인식 안 되는 현상
- [ ] GStreamer pipeline 종료 안 되는 문제
```

---

## 5. 커미트/PR과 이슈 자동 연결

### `Closes #이슈번호` 사용
- 커미트 메시지 또는 PR 본문에 다음과 같이 작성하면, PR 머지 시 이슈가 자동으로 닫힌다

```bash
git commit -m "Fix RTSP freeze (Closes #101)"
```
가령 PR 본문에:
```markdown
This PR resolves the issue where the RTSP stream freezes after 10 minutes.
Closes #101
```

---

## 6. 로컬 기록 방식 (선택)

### `issues/task/2025-03-28_phase_mag_store_ctrl.md`
```markdown
# [Task] BRAM 저장 모듈 설계

## 작업 내용
FFT의 `phase`, `magnitude`, `체조 주화수` 데이터를 외부 BRAM에 저장하기 위한 RTL 모듈 설계

## 모기
후의 PS에서 데이터를 쉽게 접근 가능하도록 AXI4 인터피스 또는 직접 접근용 BRAM 구조 사용

## 참고
- 입력 데이터: FFT 결과 1024개
- 저장 대상: external BRAM
- 설계 방식: Verilog RTL 기반
```

---

## 7. 자동화 방식 1: GitHub CLI 기본 이슈 등록

### 사전 준비
1. [GitHub CLI 설치](https://cli.github.com/)
2. 인증: `gh auth login`

### 스크립트 예시: `scripts/create_issue_from_md.sh`
```bash
#!/bin/bash

MD_FILE=$1

# 파일 여부 확인
if [ ! -f "$MD_FILE" ]; then
  echo "파일이 존재하지 않습니다: $MD_FILE"
  exit 1
fi

# 라벨 발체: 폴더 이름 = 라벨
LABEL=$(basename $(dirname "$MD_FILE") | tr '[:upper:]' '[:lower:]')
TITLE=$(head -n 1 "$MD_FILE" | sed 's/^#* *//')

# 게시판 생성
gh issue create \
  --title "$TITLE" \
  --body-file "$MD_FILE" \
  --label "$LABEL" \
  --assignee "pcw1029"
```

### 사용 예시
```bash
./scripts/create_issue_from_md.sh issues/task/2025-03-28_phase_mag_store_ctrl.md
```

> 라벨이 없을 경우 GitHub 저장소에서 무정 생성 필요
> 가령:
> ```bash
> gh label create "task" --color "0E8A16" --description "일반 작업(Task) 이슈"
> ```

---

## 사용 예시 요약

1. `issues/task/2025-03-28_phase_mag_store_ctrl.md` 로컬 파일 작성
2. `scripts/create_issue_from_md.sh` 실행 → GitHub 이슈 자동 등록
3. GitHub Issue #이 만들어진 것을 확인
4. 커미트: `git commit -m "Fix BRAM logic (Closes #104)"`
5. PR merge 시 이슈 자동 닫기

---

## 참고
- GitHub CLI 문서: https://cli.github.com/manual
- 이슈 템플릿 문서: https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/about-issue-and-pull-request-templates

---

의견이나 자동화 스크립트 개정 요청은 여기에 남겨주세요!
