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
    - stage에 올리는 것의 의미는, 현재 상태를 기억하고싶은 파일을 선택하는 것.
	- 이것은 GitBash의 아래 명령어와 같은 역할을 함
        > $ git add .
- 스테이지에 올려진 파일을 커밋으로 만들기(commit)
    - ![image](https://user-images.githubusercontent.com/11372675/205447832-0a807a07-8047-42a8-ac83-613b8e4f0d34.png)
    - 커밋 메시지 입력하고 "커밋" 버튼 클릭하기
		- 커밋 메시지는 해당 커밋의 간단한 설명
		- tip) 첫 줄은 제목, 한 줄 띄고 세번째 줄부터 본문의 역할을 함
    - 이것은 GitBash의 아래 명령어와 같은 역할을 함
        > $ git commit -m "원하는 메시지"
    - 커밋을 하면 history에서 다양한 정보를 확인 가능
        - ![image](https://user-images.githubusercontent.com/11372675/205447951-7d74e795-990a-4638-b673-6301fc03c9d7.png)
        - 방금 커밋한 내역의 날짜, 작성자, 커밋id, 변경 내용 등 확인 가능
	- 즉, 일종의 체크포인트의 역할을 한다.
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
- 변경사항 Add, Commit
    - ![image](https://user-images.githubusercontent.com/11372675/205450275-2f1d76d2-94b3-413c-b977-0f8f2f9f844a.png)
- posts/sourcetree로 commit이 올라간 것 확인
    - ![image](https://user-images.githubusercontent.com/11372675/205450304-28cbcb5d-05e0-4231-bd2c-fb37b7bfb0d3.png)
- push하기
    - push하려는 brancch 선택
        - ![image](https://user-images.githubusercontent.com/11372675/205450368-0eb7182e-343e-4b2a-9d08-6f59c4395150.png)
    - push된 것 확인
        - ![image](https://user-images.githubusercontent.com/11372675/205450395-f9ba9eb5-6237-4fbe-a608-a524d6572de4.png)
        - ![image](https://user-images.githubusercontent.com/11372675/205450418-5383b45b-bd91-48e1-967b-53b401533cef.png)

## 다른 사용자가 또 다른 branch를 사용한다면?
- 일단 master로 HEAD 이동
    - ![image](https://user-images.githubusercontent.com/11372675/205450512-46e35bc2-6205-4ee8-8628-ec3ef6f9ad4d.png)
- 새로운 branch 생성(posts/otherbranch)
		- ![image](https://user-images.githubusercontent.com/11372675/205450800-a4616832-8b05-4493-9f43-ad797e38ec12.png)
- posts/sourcetree에서 작업하던 내역이 사라졌다!
	- ![image](https://user-images.githubusercontent.com/11372675/205450698-4a2fd0c3-47f2-4f9c-bf9d-c5136b231bb1.png)
- otherbranch 사용자는 sourcetree사용자가 어떤 작업을 했는지도 모르고 수정을 가했다고 생각해보자.
	- ![image](https://user-images.githubusercontent.com/11372675/205450961-8885d596-34c2-4bcd-934b-71eda10360e7.png)
	- 새로운 파일도 만들고, 기존 파일도 수정해보았다.
- add, commit
	- ![image](https://user-images.githubusercontent.com/11372675/205451014-42adc62c-2db3-49db-8612-2f3a5a14d253.png)
- history 확인
	- ![image](https://user-images.githubusercontent.com/11372675/205451039-b4066793-85a9-4256-a198-4c4dd48ba0b9.png)
	- 오 뭔가 가지가 생겼다!

## 각자의 작업물 합치기(Merge)
- Merge : 두 Commit의 합집합 구하기
	- merge commit : 두 branch의 Commmit이 적절하게 적용되어 합쳐지는 경우
	- fast-forward : 한 Commit이 다른 Commit을 포함하여 큰 Commit으로 합쳐지는 경우
	- conflict : 두 Commit 사이에 어떤 Commit을 따라야 할 지 모호한 경우 

- fast-forward
	- master branch를 기준으로 posts/sourcetree branch 합치기
		- master branch로 checkout
		- merge하려는 commit 우클릭 -> '병합' 선택
		- ![image](https://user-images.githubusercontent.com/11372675/205451593-fb117829-c9d9-4aa5-86ca-84de3c3c6dd8.png)
		- ![image](https://user-images.githubusercontent.com/11372675/205451632-7784677f-eff6-4378-bade-bd3a198d3056.png)
		- 병합 결과 확인
			- ![image](https://user-images.githubusercontent.com/11372675/205451690-a9f2f1de-d05d-4bb2-b267-4ae3801f4cb7.png)
			- commit 된 내역이 한 개 있구나 확인
		- push하기
			-  ![image](https://user-images.githubusercontent.com/11372675/205451721-22f37563-65f6-4dc0-871c-82a55b7d42f8.png)
			- ![image](https://user-images.githubusercontent.com/11372675/205451750-4c9716df-378e-4436-8f5c-d35fa92ba9b7.png)
			- master와 origin/master모두 posts/sourcetree branch의 최신 commit을 가리킴

- conflict해결하기
	- posts/otherbranch branch를 기준으로 병합을 해보자
	- posts/sourcetree와 posts/otherbranch 둘 다 똑같은 파일을 수정했기에 분명 conflict가 생길 것임
		- 그렇기에 다른 사람도 사용하는 master branch에 바로 병합하는 것 보다는 otherbranch에서 일단 병합을 해보고 문제가 있는지 확인하는 것이 안전함
	- posts/otherbranch로 checkout
		- ![image](https://user-images.githubusercontent.com/11372675/205451987-2fda26bb-0b5d-43ef-bb04-18e29d1c7ada.png)
	- 병합 확정
		- ![image](https://user-images.githubusercontent.com/11372675/205452002-3b724620-d44f-442e-99cd-fb88c934ba9c.png)
	- 헉! 충돌이다!
		-  ![image](https://user-images.githubusercontent.com/11372675/205452012-60f533e3-5955-44f5-990a-f07d7164ca17.png)
	- 커밋하지 않은 변경사항 확인
		- ![image](https://user-images.githubusercontent.com/11372675/205452087-0e9aeef6-28e5-4acf-8e2f-3ea002168646.png)
	- VisualStudio Code로 확인해보면?
		- ![image](https://user-images.githubusercontent.com/11372675/205452142-6012e872-fcfc-4abe-b58b-c70ef955c842.png)
		- 충돌이 난 부분을 강조해줌
		- 해석 
			- <<<HEAD 부터  === 까지 : posts/otherbranch의 작업내역
			- ====부터 >>>> posts/sourcetree 까지 : master, posts/sourcetree의 작업내역
		- 우리가 해야 할 것은?
			- <<<HEAD, ====, >>>> posts/sourcetree 부분을 지우고,
			- 수동으로 적절하게 배치하기
			- ![image](https://user-images.githubusercontent.com/11372675/205452401-cf2612c2-6ec7-44fb-9a9a-fd3626da19d8.png)
	- SourceTree에서 다시 확인해보기
		- ![image](https://user-images.githubusercontent.com/11372675/205452486-0bb47aed-d7b7-4eb1-9768-a849ed9138b8.png)
		- 수동으로 배치한 것이 잘 적용된 것 같으면 add, commit
		- ![image](https://user-images.githubusercontent.com/11372675/205452541-d78c8304-03f9-49d2-8299-7ca62dc432ae.png)
	- history에서 결과 확인
		- ![image](https://user-images.githubusercontent.com/11372675/205452577-d58cceba-7e86-4292-a694-cba118393f9e.png)
		- 오, 뭔가 posts/otherbranch쪽으로 가지가 연결되었다.	
	- push해서 원격 저장소에 올리기
		- ![image](https://user-images.githubusercontent.com/11372675/205452641-06c14db0-7014-4319-bfdd-dabc8093a226.png)

- 문제를 해결했으니 master에도 적용하기
	- master branch로 체크아웃하기
		- ![image](https://user-images.githubusercontent.com/11372675/205452727-84db1af1-3f04-4227-93c9-046a6485aabe.png)
	- 충돌을 해결하고 push한 origin/posts/otherbranch의 최신 commit을 선택하고 병합
		- ![image](https://user-images.githubusercontent.com/11372675/205452771-b0049291-da73-43bf-97a0-1b1bbafd5e5a.png)
	- Push를 할 Commit 내역 2개 확인
		- ![image](https://user-images.githubusercontent.com/11372675/205452812-c99707ac-d23e-44d2-8505-b1b5509c1741.png)
	- master branch에 push
		- ![image](https://user-images.githubusercontent.com/11372675/205452854-7db15f29-0d51-402e-99e1-add617bebc22.png)
	- 병합 끝!
		- ![image](https://user-images.githubusercontent.com/11372675/205452887-a930a312-c12c-48e8-a8de-0f0271e26964.png)
- 원격 저장소에 잘 올라갔는지 확인
	- ![image](https://user-images.githubusercontent.com/11372675/205452933-52800e64-026a-4d4f-a8ac-a5918640bf9c.png)

## SourceTree에서 pull request하기
- 내 쪽에서 충돌을 해결했다고 해서 바로 master에 병합하는게 맞을까?
- master에는 완벽한 코드만 있어야 하므로 다른 사람의 동의를 구해야한다.

- PullRequest를 해보자
    - master branch에서부터 새로운 branch 생성(posts/comment)
        - ![image](https://user-images.githubusercontent.com/11372675/205467302-b56f202d-4f98-4506-a154-c32fc0cd2310.png)
	- Comment.md 파일 생성
	    - ![image](https://user-images.githubusercontent.com/11372675/205467340-e0c6326d-9a67-466e-b851-97058969afa3.png)
	- Comment.md 파일 add, commit, push
		- ![image](https://user-images.githubusercontent.com/11372675/205467426-20f43ab2-514f-4b6d-8348-b08b95d869f3.png)
	- Github 저장소에서 방금 push한 것을 확인할 수 있음
		- ![image](https://user-images.githubusercontent.com/11372675/205467457-976f3b96-d6e1-444a-931d-f74e2771c904.png)
	- Compare & Pull Request 버튼을 클릭해보자
		- ![image](https://user-images.githubusercontent.com/11372675/205467615-7c232e97-7624-4572-bdd8-b853b0007812.png)
		- base : 병합한 결과가 올라갈 branch
		- compare : 비교 대상 branch
		- able to merge : base와 compare branch 사이에 충돌이 없다는 뜻
		- reviewers : 누군가를 지정해서 pull request를 확인해달라 요청할 수 있음
		- assignees : 이 pull request를 담당하는 사람 지정.(보통 자신)
		- label : 이 pull request의 특징을 label로 간략하게 표현
	- pull request 내역 확인
		- ![image](https://user-images.githubusercontent.com/11372675/205467637-9a3ec6f1-7e6a-4380-8f4e-1610b3be35e1.png)
		- 여기에 comment를 달거나, review하거나, merge하는 등 가능
	- 일단 merge해보자
		- ![image](https://user-images.githubusercontent.com/11372675/205467673-26415dd3-0545-428a-94ba-192abce30e3d.png)
	- merge 성공!
		- ![image](https://user-images.githubusercontent.com/11372675/205467691-742c0a44-e68e-4b53-809c-3c766acab3b0.png)
- SourceTree에서 확인하기
	- SourceTree에는 아직 반영이 되지 않았다.
		- ![image](https://user-images.githubusercontent.com/11372675/205467716-1e0a2fe8-b22e-4ad7-b4a2-d2b681447c49.png)
	- "패치"를 통해 새로고침 가능
		- ![image](https://user-images.githubusercontent.com/11372675/205467752-80f03188-0499-4037-991c-faea657f53e6.png)
	- 야호! pull request해서 merge된 것을 확인할 수 있다.
		- ![image](https://user-images.githubusercontent.com/11372675/205467765-a2a0a411-f394-4e9a-8ba2-3d05b1169acd.png)
- 내 저장소에 pull rerquest된 것 반영하기
	- master branch로 체크아웃
	- pull받기
		- ![image](https://user-images.githubusercontent.com/11372675/205467826-e4c71b03-2685-44bb-90aa-8034969ebd4d.png)
	- 야호! master(로컬)도 origin/master(원격)와 똑같이 pull request를 통해 merge된 커밋을 가리킨다!
		- ![image](https://user-images.githubusercontent.com/11372675/205467843-6c2e4293-f1be-4750-a6f7-8a008a76a486.png)
