---
layout: single
title:  "Git Hub의 프로젝트 폴더 구성 및 설정 방법"
date: 2024-11-13
tags: git
categories: 
   - Git
---

# Git Hub의 프로젝트 폴더 구성 및 설정 방법

## 로컬 폴더 구성

프로젝트 이름(AAA)/
├── 장치 이름1 or 모듈 이름1(AA1)/        ← submodule
│    ├── HW/
│    └── SW/
├── 장치 이름2 or 모듈 이름2(AA2)/         ← submodule
│    ├── HW/
│    └── SW/

├── README.md
└── .gitignore



## Git Hub Repository 구성

회사 프로젝트의 경우 회사 이름의 Organization을 생성하여 관리하는 것이 효율적이다. Organization이 생성되어 있다고 가정한다.

아래와 같이 총 3개의 Repository를 생성한다.

AAA Repository 생성

AA1 Repository 생성

AA2 Repository 생성



AAA Repository 아래 AA1, AA2를 포함 시킬것이다.

각 Repository를 생성할때 Readme를 반드시 포함해서 각 모듈 혹은 장치별 설며을 기입한다.



## Monorepo 방식의 로컬 폴더 구성

로컬 저장소에 AAA를 git clone 한다.

AAA폴더로 이동하여 아래 명령을 실행하게 되면 AAA폴더 아래 AA1, AA2의 Git Repository를 가져온다.

 

```bash
$ git submodule add https://github.com/your-org/Organizations/AA1.git
$ git submodule add https://github.com/your-org/Organizations/AA2.git
```



AAA폴더 아래 AA1과 AA2 폴더가 생성된 후 AAA에 현재 서브모듈을 추가했다는 메시지를 아래 명령을 사용하여 기록한다.

```bash
$ git add .gitmodules AA1 AA2
$ git commit -m "Add AA1, BB2 as submodules"
```



push하여 로컬 저장소와 원격저장소를 일치 시켜준다.

```bash
$ git branch -M main
$ git push -u origin main
```



목표로 했던 폴더의 구성을 AA1, AA2의 폴더 아래 구성하면 된다.

├── AAA
│   ├── AA1
│   │   ├── HW
│   │   ├── README.md
│   │   └── SW
│   ├── AA2
│   │   ├── HW
│   │   ├── README.md
│   │   └── SW
│   └── README.md



## 클론 시 주의사항

다른 개발자가 전체를 clone 할 땐 서브모듈까지 같이 받는 명령을 써야 합니다:
git clone --recurse-submodules https://github.com/your-org/Acewavetech-RCS.git

## 원격 저장소 등록
로컬 저장소 abc폴더에 아래 명령을 통해 Git 초기화를 진행한다.

```bash
$ git init
```



로컬 저장소 abc를 원격저장소 pcwTest와 연결하기 위해 아래 명령을 수행한다.

```bash
$ git remote add origin https://github.com/your-username/pcwTest.git
```



로컬 저장소와 원격저장소의 내용을 병합하려면 아래 명령을 통해 가능하다.

```bash
$ git pull origin main --allow-unrelated-histories
```



변경 사항 커밋 및 푸시 명령을 통해 원격 저장소로 업로드 한다.

```bash
$ git add .
$ git commit -m "Initial commit from abc folder"
$ git push -u origin main
```



## 원격 저장소와 로컬 저장소의 충돌
로컬 저장소와 원격저장소의 동일한 파일이 존재하며 파일의 내용이 서로 다른경우 아래 2가지 방법으로 충돌을 해결해줘야 한다.
### pull 전 임시로 로컬 파일 백업

```bash
$ mv README.md README_local_backup.md
$ git pull origin main
$ mv README_local_backup.md README.md
```



### stash를 이용한 아전한 pull

```bash
$ git stash             # 현재 변경사항 (README.md 포함) 저장
$ git pull origin main  # GitHub의 내용 가져오기
$ git stash pop         # 저장한 변경사항 다시 적용
```


만약 충돌이 발생하면, 아래처럼 표시됩니다:

```
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
```


이럴 경우, README.md 파일을 열면 충돌 부분이 이렇게 보입니다

```bash
<<<<<<< HEAD
(로컬의 README.md 내용)
=======
(원격 저장소의 README.md 내용)
>>>>>>> origin/main
```

이 부분을 직접 수정한 뒤에 아래 명령어로 충돌 해결을 마무리합니다

```bash
$ git add README.md
$ git commit -m "Resolve conflict in README.md"
```



