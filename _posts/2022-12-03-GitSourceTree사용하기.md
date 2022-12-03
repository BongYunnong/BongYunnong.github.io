---
title: "소스트리 사용하기"
permalink: /posts/git/usingsourcetree
excerpt: "How to use SourceTree"
last_modified_at: 2022-12-03T23:41:00-04:00
toc: true
---

# Git
- https://bongyunnong.github.io/posts/git/installgit

# SourceTree
- SourceTree는 Git의 사용을 관리할 수 있도록 도와주는 GUI(Graphic User Interface) 프로그램이다.

## SourceTree 설치
- https://www.sourcetreeapp.com/
- 설치 파일 다운로드 및 실행
    - ![image](https://user-images.githubusercontent.com/11372675/205446577-687212d5-13d0-4b38-b253-f6d0f5505a0f.png)
- Registration
    - BitBucket Server나 BitBucket 계정 선택
    - BitBucket은 GitHub와 비슷한 서비스 제공
    - BitBucket은 구글 계정 등으로 소셜 로그인이 가능하므로, BitBucket 선택
    - 로그인 완료!
        - ![image](https://user-images.githubusercontent.com/11372675/205446699-fef54722-4e43-40d6-bb35-f32899b781e2.png)
- 도구 설치
    - ![image](https://user-images.githubusercontent.com/11372675/205446754-0a60707a-6a58-42b2-89da-9e0a1ca9dfb2.png)
    - Git을 이미 설치하였다면 위의 이미지처럼 Git이 저장된 경로를 확인 가능
    - Mercurial은 Git과 비슷한 버전관리툴
    - 우리는 Git만 사용할 것이기에 Mercurial은 체크 해제
- Preferences
    - ![image](https://user-images.githubusercontent.com/11372675/205446835-7fe27e61-0c1f-468e-bc13-42f51adb32c7.png)
    - user name과 user email 입력
- SSH 키 불러오기?
    - ![image](https://user-images.githubusercontent.com/11372675/205446897-d82057d1-4e11-4637-aca0-4b855311e475.png)
    - SSH 키는 Secure Shell의 약자로, 안전하게 로그인하기 위한 키의 역할을 함.
    - 현재는 필요 없기에 "아니오" 선택

- 계정 추가 선택
    - ![image](https://user-images.githubusercontent.com/11372675/205446967-a243c1a8-4b4c-4c06-ac73-6f8eebd879c2.png)
- 호스팅 계정 편집
    - ![image](https://user-images.githubusercontent.com/11372675/205446996-accb3c3e-469e-4d8c-8a9a-1d4481f6dab7.png)
    - 호스팅 서비스를 Bitbucket에서 Github로 변경
    - OAuth 토큰 새로고침 버튼 클릭
        - ![image](https://user-images.githubusercontent.com/11372675/205446949-d1bd585f-8c56-49c7-877a-401b662bbb01.png)
- 원격 저장소 확인하기
    - ![image](https://user-images.githubusercontent.com/11372675/205447114-e28c39da-8435-4d95-a9f8-486dcd20f41f.png)

## SourceTree 사용하기
- 소스트리 탭 목록
    - Local : 로컬 저장소
    - Remote : 원격 저장소
    - Clone : 원격 저장소를 내 컴퓨터에 받아오기
    - Add : 내 컴퓨터의 로컬저장소를 소스트리에 추가하기
    - Create : 내 컴퓨터의 폴더에 새로운 로컬저장소 생성하기

- 이미 Git을 사용하고있었다면 Add를 클릭
    - 원하는 로컬 저장소 경로 선택
        - ![image](https://user-images.githubusercontent.com/11372675/205447393-006d9ba8-9a6c-4f21-9464-521d340b8890.png)
    - Git 저장소가 연결된 것 확인
        - ![image](https://user-images.githubusercontent.com/11372675/205447420-ea128861-4cf6-40cc-b19a-bf9f82796cff.png)
    - 좌측 History 탭을 선택하면 지금까지의 Commit 내역도 확인할 수 있다.
        - ![image](https://user-images.githubusercontent.com/11372675/205447491-3eccc555-81f7-4b5b-be5d-887526a924ea.png)

## SourceTree로 변경내역 add, commit, push해보기
- 좌측 상단 "커밋" 버튼을 클릭하면 변경내역을 확인할 수 있다.
    - ![image](https://user-images.githubusercontent.com/11372675/205447601-079e7f3d-81d9-4a9d-be0c-8f9312fb54be.png)
- 원하는 파일을 스테이지에 올리기(Add)'
    - ![image](https://user-images.githubusercontent.com/11372675/205447645-f01a87ee-3bbc-413b-9061-3a5a3d212ac6.png)
    - 이것은 GitBash의 아래 명령어와 같은 역할을 함
        > $ git add .
- 스테이지에 올려진 파일을 커밋으로 만들기(commit)
    - ![image](https://user-images.githubusercontent.com/11372675/205447832-0a807a07-8047-42a8-ac83-613b8e4f0d34.png)
    - 커밋 메시지 입력하고 "커밋" 버튼 클릭하기
    - 이것은 GitBash의 아래 명령어와 같은 역할을 함
        > $ git commit -m "원하는 메시지"
    - 커밋을 하면 history에서 다양한 정보를 확인 가능
        - ![image](https://user-images.githubusercontent.com/11372675/205447951-7d74e795-990a-4638-b673-6301fc03c9d7.png)
        - 방금 커밋한 내역의 날짜, 작성자, 커밋id, 변경 내용 등 확인 가능
- 로컬 변경내역 원격 저장소에 업로드하기(push)
    - 누가 봐도 "Push"버튼을 눌러야 할 것 같음
    - ![image](https://user-images.githubusercontent.com/11372675/205447994-6d47b5d9-e618-45cc-9eac-863453eb05cc.png)
    - 이것은 GitBash의 아래 명령어와 같은 역할을 함
        > $ git push

## 에러! 만약 다음과 같은 에러가 발생하면?
- 문제 : scope가 잘못되었다고 한다.
    - ![image](https://user-images.githubusercontent.com/11372675/205448237-dde5b9aa-930f-4f7a-985d-41d54567e36c.png)
    - 필자의 경우 'workflow' scope가 없다고 함.
- 원인 : 원하는 작업을 하기 위한 key 값이 제대로 들어가있지 않기 때문
- 해결해보자!
    - Github의 Settings 선택
        - ![image](https://user-images.githubusercontent.com/11372675/205448348-72d06e8e-5e72-4bd6-884a-6e239f510f74.png)
    - 왼쪽 하단에 Developer Settings 탭 선택
        - ![image](https://user-images.githubusercontent.com/11372675/205448372-fd41f464-d89b-48d6-8f4a-1e79f82a2208.png)
    - Personal Access Token 생성하기
        - ![image](https://user-images.githubusercontent.com/11372675/205448424-ba9fe92d-3365-4886-bf2c-62f1eaf4cdaa.png)
    - 필요한 옵션 선택
        - 필자는 'workflow' scope가 문제였다.
        - ![image](https://user-images.githubusercontent.com/11372675/205448537-31c69880-feb5-4202-be04-7d2687ad9226.png)
        - 생성된 key를 꼭 복사해놓자(나중에 못 볼 수 있음)
    - 제어판 -> 사용자 계정 -> 자격 증명 관리자
        - ![image](https://user-images.githubusercontent.com/11372675/205448318-b6f21cf7-15a2-4cf9-9e79-119888753cc8.png)
        - Windows 자격 증명 관리 버튼을 클릭해보자.
    - github.com 자격 증명 선택
        - ![image](https://user-images.githubusercontent.com/11372675/205448953-06977a0c-4fbf-4abf-a436-e95891d7a65f.png)
    - 복사한 personal token 붙여넣기
        - ![image](https://user-images.githubusercontent.com/11372675/205448926-54595532-0859-411a-90b1-c046d30ce81a.png)
    - Push 성공!
        - ![image](https://user-images.githubusercontent.com/11372675/205448904-a9821088-d125-4c71-bf72-ce3daf59829f.png)

## SourceTree에서 branch 사용하기
- branch 생성하기
    - ![image](https://user-images.githubusercontent.com/11372675/205450185-306c181e-4d1f-4251-b930-4cb30ae4665e.png)
    - 현재 branch 확인
    - 새 branch 이름 설정
    - 새 branch로 체크아웃(새 branch로 이동)
    - 브랜치 생성 버튼 클릭
- 굵은 글씨로 현재 있는 branch 확인
    - ![image](https://user-images.githubusercontent.com/11372675/205450231-09794ddf-d805-46ff-a643-44516fb15180.png)
