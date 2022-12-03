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

## otherbranch가 작업한 내역
### 작업1
- 작업1-1
### 작업2
- 작업2-1
- 작업2-2

