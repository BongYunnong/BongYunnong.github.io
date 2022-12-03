---
title: "언리얼 CPP 파일 삭제"
permalink: /posts/unreal/removeunrealcppfile
excerpt: "How to remove Unreal CPP file"
last_modified_at: 2022-12-01T14:46:00-04:00
toc: true
---

# 언리얼의 CPP 파일 삭제
## CPP 파일을 삭제해볼까?
- 언리얼 CPP 프로젝트를 생성해서 작업을 하다보면 필요없는 CPP 파일을 지워야 할 때가 있다.
- 그냥 지우면 되는거 아닌가? 싶겠지만 놀랍게도 그냥 지울 수 없다.
    - ![image](https://user-images.githubusercontent.com/11372675/204975619-ccd1fd77-9c78-49b4-9bc0-f39c51c0536a.png)
    - delete 버튼이 비활성화되어있다.

## 해결방법 
1. 언리얼 엔진 종료
2. Source폴더 내부의 필요없는 cpp 파일, 헤더 파일을 삭제
3. .vs, Binaries, Intermmediate, 프로젝트명.sln 파일을 삭제
    - ![image](https://user-images.githubusercontent.com/11372675/204976099-25e51349-27e8-43a4-a999-10c0412045d3.png)
4. 프로젝트명.uproject 파일 우클릭 > Generate Visual Studio Project files 옵션을 선택
    - ![화면 캡처 2022-12-01 144428](https://user-images.githubusercontent.com/11372675/204976402-2b182e61-fdb8-4d04-8ff3-6a40e5af95c6.png)
    - 언리얼 엔진 Rebuild
    - 아까 지웠던 .vs, Binaries, Intermmediate, vs solution파일이 다시 생성된다.
5. 야호! 해결!

## 관련 파일, 폴더 설명
- .vs
    - Visual Studio에서 프로젝트를 실행할 때 초기화 및 데이터 구조를 저장하여 나중에 빠르게 불러오기 위한 폴더
- Binaries
    - C++ 코드가 컴파일된 결과물을 저장
    - 폴더를 삭제해도 빌드될 때마다 새로 생성된다.
- Intermediate
    -  엔진이나 게임 빌드 도중 생성된 임시 파일이 위치한 공간
- 프로젝트명.sln
    - C++ 프로젝틀르 관리하기 위한 Visual Studio Solution파일.
    - 이 solution이 관리하는 프로젝트 파일은 Intermediate > ProjectFiles폴더에 위치
    - 이 파일을 삭제해도 Generate Visual Studio project file 옵션을 선택하면 새로 생성

## 기타 폴더 설명
- Config
    - 엔진 행위를 제어하는 값 설정용 환경설정 파일
- Content
    - Engine이나 게임에 대한 콘텐츠, 애셋, 맵 등이 위치한 공간
- DerivedDataCache
    - 참조된 콘텐츠에 대해 로드시 생성된 파생 데이터 파일이 위치한 공간
    - 이 캐시 파일이 없으면 로드 시간이 엄청 길어짐
- Build
    - 엔진이나 게임을 빌드하는 데 필요한 파일, 플랫폼별 빌드를 만드는 데 필요한 파일 위치한 공간
- Saved 
    - 자동저장, 환경설정(.ini), 로그 파일이 위치한 공간
- Source
    - C++ 코드가 위치한 공간
    - 이 폴더를 삭제하면 프로젝트가 망가져버린다.

## ETC
- 언리얼 디렉토리 문서
    - https://docs.unrealengine.com/4.27/en-US/Basics/DirectoryStructure/
- 가끔씩 이전에 분명 생성한 파일이 Unreal Project에서 보이지 않을 때가 있다.
    - 파일 탐색기로 Source폴더에 들어가보면 제대로 존재함
    - 이것 역시 위의 방법을 통해 해결 가능하다.