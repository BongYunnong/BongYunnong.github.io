---
title: "Unreal Source Engine 내려받기"
permalink: /posts/Unreal/
excerpt: "Unreal Source Engine 내려받기"
last_modified_at: 2022-11-30T00:31:00-04:00
toc: true
---

# 시작하기
- 언리얼 문서에서 친절하게 사용법을 알려주고있으므로 참고
    - [한국어 문서][한국어문서]
    - [영어 문서][영어문서]

- 꼭 언리얼 소스 코드를 다운로드 할 필요는 없다.
    - 그냥 Epic GameLauncher를 통해서 다운로드하면 매우 쉽게 언리얼 엔진을 다운로드 받을 수 있다.
    - 하지만 소스 코드로 다운받으면
        - 에픽 기술자가 제공하는 모든 최신 기능과 버그 픽스에 액세스 가능
        - 프로젝트에 버그가 있을 경우 직접 fix 및 rebuild 가능
        - 버그를 제대로 수정하면 엔진을 개선하고 unreal engine community에 공헌하게 됨
        - 무려 Dedicated Server를 사용할 수 있음!(즉, Epic GemaLauncher로 엔진을 다운받으면 DedicatedServer를 사용할 수 없다.)

# 언리얼 엔진 Github 저장소
- [언리얼엔진 저장소][언리얼엔진저장소] 저장소에 액세스
    - 언리얼 엔진 회원이 되어야한다.
    - Github 계정이 있어야 한다.
    - Github 계정과 언리얼 엔진 계정을 연결해야한다.
        - https://www.unrealengine.com/ko/ue-on-github
        1. github 계정 생성
        2. 언리얼 엔진 계정 대시보드 열기
        3. github 연결하기
        4. 계정 연결하기
        5. OAuth 앱 인증하기
        6. 이메일 초대 수락하기


# 소스 코드 다운로드
- Github 저장소에 접근하면 여러가지 branch를 확인 가능
- dev, staging, test라는 이름이 붙은 branch는 사용자를 위한 것은 아니고 Epic 개발자들을 위한 것임
- release branch는 최신 공식 버전을 반영
    - Epic QA팀의 검수를 마친 branch
    - Unreal Engine 프로젝트를 시작하기에 적함함
- main branch
    - 대부분의 언리얼 엔진 개발사항은 ue5-main branch에서 확인 가능
    - 그렇기 때문에 버그가 있을 수 있고, 어쩌면 컴파일도 안 될 가능성도 있음

- 다운로드 과정
    1. Github 저장소를 fork, clone하기
        - 만약 Git을 사용하지 않겠다면 "download zip"버튼을 통해 로컬 환경에 언리얼 엔진 소스 다운받기
    2. Visual Studio 설치하기
        - 문서에서는 2019 버전을 추천함
        - 필자는 visual studio 2022로 진행했는데 문제 없었음
        - Visual Studio를 설치할 때 "C++를 사용한 게임 개발" 옵션을 선택하도록 하자
            - 세부 정보 : C++ 프로파일링 도구, C++ AddressSanitizer, Windows10SDK, IntelliCode, UnrealEngine 설치 관리자, Nuget 패키지 관리자 등..
    3. 파일 탐색기를 통해 언리얼 소스를 다운받은 폴더로 접근하여 "Setup.bat" 파일 실행
        - 이 작업을 통해 엔진에 필요한 바이너리 콘텐츠를 다운받고 언리얼을 위한 요구사항을 맞출 수 있게 된다.
        - 만약 경고창이 뜨면 More Info -> Run Anyway 클릭
    4. "GenerateProjectFiles.bat" 파일 실행
        - 엔진을 위한 프로젝트 파일을 생성
        - 만약 에러가 발생하면 VisualStudio Installer의 개별 구성요소에서 .Net Framework 4.6.2도 설치해야 나중에 에러가 발생하지 않음
    5. "UE5.sln" 파일을 실행
        - Visual Studio가 실행
            - solution configuration을 "Development Editor"로 설정
            - solution platform을 Win64로 설정
            - F5키를 누르면 전체 빌드
            - 매우매우매우매우 오래걸림
    6. Visual Studio로 solution을 실행하고, UE5를 startup Project로 설정한 후(우클릭하면 메뉴에 존재) Debug -> Start new instance를 클릭하면 언리얼 엔진 실행
        - F5를 눌러도 실행 가능
    7. 야호! 드디어 끝!

## 권장 세팅
- solution configuration 메뉴 폭 늘리기
    - solution configuration를 "Development Editor"로 설정하면 폭이 좁아서 잘린다.
    - 툴바 우클릭 -> 사용자 지정 클릭 -> 명령 탭 선택 -> 도구 모음 옵션(표준) -> 솔루션 구성 선택 -> 선택사항 수정 클릭 -> 너비를 200으로 설정한 후 확인
- solution platform dropdown 메뉴 추가
    - 툴바 오른쪽에 확장 버튼 클릭 -> 단추 추가/제거 클릭 -> 솔루션 구성 체크
- 오류 목록 창 끄기
    - 일반적으로 Error List(오류 목록) 창은 코드에 오류가 있으면 자동으로 뜨는데, 언리얼엔진에서 사용할 때는 오류 목록 창에 잘못된 오류 정보가 표시될 수 있음
        - 오류 목록 창을 비활성화하고 언레얼 엔진 출력 창의 실제 코드 오류를 확인하는게 제일 좋음
    1. 오류 목록 창이 있다면 닫기
    2. Tools 메뉴에서 Options 대화창 열기
    3. Projects and Solutions를 선택하고 Always Show Error List if build finishes with error 체크 해제
    4. OK

- 기타 유용한 구성 설정
    - Show Inactive Blocks 끄기.
        - 이거 안 끄면 텍스트 에디터에 너무 많은 코드 청크가 회색으로 표시될 수 있음
        - Tools -> Options -> Text Editor -> C/C++ -> View 에서 끔
    - Disable External Dependencies Folders를 true로 설정하면 Solution Explorer에서 불필요한 폴더 숨길 수 있음
        - Tools -> Optiosn -> Text Editor -> C/C++ -> Advanced에서 찾을 수 있음
    - Edit & Continue 기능이 필요없으면 끄기
        - Tools -> Options -> Debugging -> Edit and Continue
    - IntelliSense 켜기
        - C++ 컴파일러를 사용해서 모든 코드 행을 검증함 -> 워크플로우가 매우 빨라진다.
        - 오류 목록의 우클릭 메뉴에서 켜고 끌 수 있음
        - 구불구불한 선
            - Options -> C/C++ -> Advanced
            - Auto Quick Info : True
            - Disable IntelliSense : False
            - Disable Auto Updating : False
            - Disable Error Reporting : False
            - Disable Squiggles : False
            - Disable #include Auto Complete... : False
            - Use Forward Slash in #include ... : False
            - Max Cached Translation Units : 15
        - 코드 편집할 때 구불구불한 선이 나타나는 데 시간이 좀 걸릴수도 있음
            - 이건 include 파일이 많기 때문
        - 가끔 잘못 탐지된 IntelliSense오류가 보일 수 있음
            - 꼭 필요한 코드는 #ifdef **INTELLISENSE**로 감싸서 구불구불한 선을 없앨 수 있음
        - 반응 속도를 높이려면 Max Cached Translation Units 세팅을 높이면 되지만 메모리 사용량이 늘어남
        - 일부 C++ 파일은 아직 IntelliSense와 호환되지 않음
        - 언리얼 빌드 툴에 -IntelliSense 옵션이 생김
            - 모든 프로젝트 파일에 IntelliSense 프로퍼티 시트를 생성하는 옵션

- 문서 참고
    - https://docs.unrealengine.com/4.27/ko/ProductionPipelines/DevelopmentSetup/VisualStudioSetup/

## ETC
- 빌드를 완료하면 프로젝트의 'Engine\Binaries\' 경로에서 주요 툴을 확인 가능
    - UE5Editor.exe : 언리얼 에디터
    - UnrealInsights.exe : 프로파일러 툴
        - https://docs.unrealengine.com/4.27/ko/TestingAndOptimization/PerformanceAndProfiling/UnrealInsights/
    - NetworkProfiler.exe : 네트워크 프로파일러 툴
        - https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/Networking/NetworkProfiler/

- UnrealVS 익스텐션
    - Visual Studio 용 UnrealVS 익스텐션으로, 언리얼 엔진으로 개발할 때 자주 쓰는 기능을 쉽게 접할 수 있음
    - 문서 : https://docs.unrealengine.com/4.27/ko/ProductionPipelines/DevelopmentSetup/VisualStudioSetup/UnrealVS/

[한국어문서]: https://docs.unrealengine.com/5.1/ko/downloading-unreal-engine-source-code/
[영어문서]: https://docs.unrealengine.com/5.1/en-US/downloading-unreal-engine-source-code/
[언리얼엔진저장소] : https://github.com/EpicGames/UnrealEngine/

