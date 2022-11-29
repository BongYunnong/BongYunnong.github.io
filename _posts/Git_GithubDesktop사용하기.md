---
title: "Github Desktop 사용하기"
permalink: /posts/git/
excerpt: "소프트스킬 독후감"
last_modified_at: 2022-11-29T23:33:00-04:00
toc: true
---

# Git
## Git이란?
- 컴퓨터 파일의 변경사항을 추적하는 버전 관리 시스템(VCS:Version Control System)이다.
    - 간단하게 이야기하면 파일, 폴더를 동기화시켜주는 소프트웨어이다.

- 이게 왜 필요할까?
    - 프로그램을 만들다보면 데이터가 날라가는 경우도 있고, 중간에 잘못 판단해서 이전 버전으로 롤백해야하는 상황이 생긴다. 이때 Git을 사용하면 이전에 백업을 해둔 버전으로 돌아갈 수 있기에 안전하게 프로그램을 개발할 수 있다.

- Repository(저장소) : Git으로 버전 관리하는 디렉토리
- local repository(로컬 저장소) : 로컬 환경(개인 PC)에서 관리되는 Git 저장소
- remote repository(원격 저장소) : 외부 환경(Github 등)에서 관리되는 Git 저장소

# Github
- Github는 Git저장소를 관리할 수 있도록 클라우드 서버 기반으로 구현된 호스팅 서비스이다. 
- Github같은 서비스를 통해 여러명의 사람들이 각자의 PC로 작업한 결과물을 공유-동기화하여 협업이 가능하다.


## Github 저장소 만들기
- Github 사이트 접속
    - [https://github.com][github]
- 회원가입 및 로그인
- Github 저장소를 만들기
    1. profile 클릭
    2. your repositories 클릭
    3. New 버튼 클릭
    4. Repository name 설정
    5. 공개(Public)할 것인지, 비공개(Private)할 것인지 선택
    6. Add a README file 옵션은 활성화하면 해당 저장소를 간단하게 설명하는 markdown파일을 생성

# Github Desktop
- Git 프로그램 자체는 직관적이지 않아서 초보가 사용하기 불편하다.
    - Github Desktop은 branch나 commit 내용 등을 직관적인 UI로 보여주기에 보다 쉽게 접근, 이용할 수 있다.

## Github Desktop 설치하기
- Github Desktop 사이트 접속
    - [https://desktop.github.com/][githubDesktop]
- 다운로드 버튼 클릭
- 실행하면 Github 계정으로 로그인 가능
## Github의 원격 저장소를 로컬 저장소로 가져오기
- 첫 번째 방법
    - File 탭 -> Clone Repository -> 원하는 저장소 선택 -> 적절한 로컬 path 설정 -> Clone
- 두 번째 방법
    - github 사이트에서 원하는 저장소로 이동
    - code 버튼 클릭 -> url을 copy -> github desktop에서 File 탭 -> Clone Repository -> URL 탭 선택 -> 복사한 url paste -> 적절한 로컬 path 설정 -> Clone

## 로컬 저장소의 변경사항을 Github의 원격 저장소에 적용하기
- 이제 원격 저장소를 clone하여 로컬 PC에 저장소가 복사가 되었으면, 복사된 파일들을 변경하면서 작업이 이루어질 것이다. 그런데 로컬 PC에서만 변경이 된 것이기에, 내가 작업한 것을 다른 사람이 확인을 하기 위해서는 작업물을 다시 원격 저장소에 올려야한다.
    - 이러한 작업은 "Add", "commit"과 "push"를 통해 이루어진다.

- Add
    - commit을 하기 전에 commit을 할 파일을 묶는 작업
    - 이때 commit을 할 파일을 묶는 것을 'stage에 올린다'라고 표현함
    - terminal 명령어
        > git add . 
        - 모든 변경사항을 stage에 올리겠다.
- Commit
    - 파일의 변경사항을 로컬 저장소에 기록하는 것
    - terminal 명령어
        > git commit -m "first commit test"
        - -m은 뒤에 문자열을 commit message로 설정하겠다는 옵션
- Push
    - commit된 내용을 원격 저장소에 기록하는 것
    - terminal 명령어
        > git push
        - 원격 저장소에 기록하겠다.
    
- tip) 위의 terminal 명령어들을 Git bash를 활용할 때 입력하는 명령어들이다. 하나의 예시이고, 여러가지 옵션을 줄 수 있다.


- GithubDesktop에서 변경사항 원격 저장소에 적용하기
    1. 로컬 저장소에서 파일을 변경한 후 선택된 저장소에서의 "Changes" 확인
    2. Changes 목록 중에서 원격 저장소에 업로드하고 싶은 파일을 선택
    3. 어떤 변경사항이 있는지 메시지를 기록한 후 "Commit to main" 버튼을 클릭
    4. "Push Origin" 버튼 클릭
    5. Github 웹사이트에 접속하여 적용된 변경사항 확인

## 원격 저장소의 변경사항을 로컬 저장소에 적용하기
- 이번에는 위와 반대로, 원격 저장소의 변경사항을 로컬 저장소에 적용하는 방법이다.
    - 보통 협업을 하면 다른 사람들이 원격 저장소에 자신의 작업물을 올릴테니, 그 작업물을 내 로컬 환경에서 보고싶을 때 필요하다.
- GithubDesktop에서 원격 저장소의 변경사항 가져오기
    1. "Fetch Origin" 버튼 클릭
    2. 변경사항이 있으면 "Pull Origin" 버튼이 활성화
    3. Pull Origin 버튼 클릭하면 원격 저장소의 작업물을 Pull(가져옴)

## 주의!
- 만약 원격 저장소에서 가져오려하는 파일과 현재 내가 작업중인 파일이 겹친다면, 둘 중 한 쪽의 작업내용은 소실될 것이다. 이때 "Merge"하는 과정이 필요하다.
    - 이것은 다음 포스트에서 다루도록 하자.

## ETC
- Github 대신 Gitlab을 사용하는 경우도 있다.
- GithubDesktop을 아예 사용하지 않는 경우도 있고(Git bash로 관리) SourceTree나 SVN을 활용하는 협업 방식도 있다.

[github]: https://github.com
[githubDesktop]: https://desktop.github.com/

