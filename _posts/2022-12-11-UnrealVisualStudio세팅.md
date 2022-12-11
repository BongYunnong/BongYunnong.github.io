---
title: "언리얼 Visual Studio 세팅"
permalink: /posts/unreal/unrealvisualstudiosetting
excerpt: "Unreal Visual Studio Setting"
last_modified_at: 2022-12-11T19:46:00-04:00
toc: true
---

# 언리얼 Visual Studio Setting
- 참고 : https://docs.unrealengine.com/4.27/ko/ProductionPipelines/DevelopmentSetup/VisualStudioSetup/
## Visual Studio Installer
- C++를 사용한 게임 개발
    - ![image](https://user-images.githubusercontent.com/11372675/206899433-e787efce-51d9-4535-9843-e75f0ffbd1ef.png)
- .Net 데스크톱 개발
    - ![image](https://user-images.githubusercontent.com/11372675/206901299-0e5a5bff-7b7b-4b87-a845-89bbc7c9ef17.png)
- Desktop development with C++
- Universal Windows Platform devvelopment
- 언어팩
    - 되도록이면 영어 환경에서 개발하는 것이 좋음
## Visual Studio
- solution 구성의 dropdown 메뉴 폭 수정
    - 처음에는 Deug까지밖에 안 보여서 이게 Debug인지, Debug Editor인지 모름
    - ![image](https://user-images.githubusercontent.com/11372675/206900512-a5a0f8e8-a31d-4138-bf0b-f90da32f4df9.png)
    1. 메뉴 우클릭 > Customize...
        - ![image](https://user-images.githubusercontent.com/11372675/206900544-67cbc119-c725-4b56-b2a9-6d7e99537ccf.png)
    2. Commands 탭 > Toolbar을 'Standard'로 설정 
    3. Preview 목록에서 Solution Configurations 선택 > Modify Selection 선택
        - ![image](https://user-images.githubusercontent.com/11372675/206900564-df1d1592-717d-421e-a088-35757827bf8d.png)
    4. Width를 200으로 설정 > OK
        - ![image](https://user-images.githubusercontent.com/11372675/206900576-6e5b8732-33de-490a-b0c6-bae50590ec9a.png)
- Solution Platform Dropdown 추가
    1. Standard 툴바 맨 오른쪽에 확장 버튼 선택 > Add or Remove Buttons 옵션 선택
        - ![image](https://user-images.githubusercontent.com/11372675/206900592-b1caccc0-9d4a-491e-b2cd-6146f2a78999.png)
    2. Solution Platforms 옵션 활성화
        - ![image](https://user-images.githubusercontent.com/11372675/206900618-e9aa9bd1-1a0c-4280-9135-da550cc4ef44.png)
- 오류 목록 창 끄기
    - 왜 오류목록을 끄냐 싶을텐데, 언리얼 에디터에서 나오는 실제 코드 오류를 확인하는게 더 정확함
    1. Tools > Options...
    2. Projects and Solutions 선택 > Always show Error List if build finishes with error 체크해제 > OK
    - ![image](https://user-images.githubusercontent.com/11372675/206900641-8a3c264a-3ce4-40d0-8c16-9b342c5e300d.png)
- 기타
    - Show Inactive Blocks 끄기
        - 이유 : 텍스트 에디터에 너무 많은 코드 chunk가 회색으로 표시됨..
        - Tools > Options > Text Editor > C/C++ > View에서 해제
        - ![image](https://user-images.githubusercontent.com/11372675/206900669-2fb1eea7-d387-47ac-a7c3-aa7ed85dd9b3.png)
    - Disable External Dependencies Folders를 true로 설정
        - 이유 : Solution Explorer에서 불필요한 폴더를 숨길 수 있음
        - Tools > Options > Text Editor > C/C++ > Advanced
        - ![image](https://user-images.githubusercontent.com/11372675/206900696-1e692ac0-f20a-41b6-9cd3-0dde27fb3d02.png)
    - Edit & Continue 기능이 필요없으면 끄기
        - Edit & Continue? : 원래는 응용프로그램에 버그가 있으면 응용 프로그램을 중지하고 코드를 변경한 후 컴파일해야한다. Edit & Continue는 실행 중인 응용 프로그램이 메모리에서 수정
        - 필자는 딱히 필요없을 것 같아서 끔
        - Tools > Options > Debugging > General > Edit and Continue
        - ![image](https://user-images.githubusercontent.com/11372675/206900742-1c57b184-ddfd-403b-98fe-8feb82bd127e.png)
    - IntelliSense 켜기
        - C++ 컴파일러로 모든 코드 행을 컴파일함
        - 주의! 가끔 오작동함
        - 주의! 어떤 C++ 파일에 대해서는 호환되지 않을 수 있음
        1. Options > C/C++ > Advanced
        2. Auto QuickInfo : True
        3. Disable IntelliSense : False
        4. Disable Auto Updating : False
        5. Disable Error Reporting : False
        6. Disable Squiggles : False
        - ![image](https://user-images.githubusercontent.com/11372675/206900781-ee40f60c-84af-49b8-9b2c-393e0ad2fc0b.png)

## UnrealVS Extension
- 참고 : https://docs.unrealengine.com/5.1/en-US/using-the-unrealvs-extension-for-unreal-engine-cplusplus-projects/
- Unreal Engine으로 개발할 때 자주 사용하는 동작을 쉽게 사용할 수 있게 됨
- 주의! Visual Studio Community, Professional에서만 호환(Express버전은 호환 안 됨)
- 포함된 기능
    - 시작 프로젝스 설정
    - 시작 프로젝트 빌드를 위한 바인딩가능 명령
    - 명령줄 인수 설정
    - 프로젝트 일괄 빌드
    - 프로젝트 빠른 빌드 메뉴
- 설치하기
    1. UnrealVS.vsix 파일 찾고 설치
        - C:\Users\tiger\Documents\UnrealEngine\Engine\Extras\UnrealVS\VS2022
    2. Instsaller가 올바른 VS 버전을 대상으로 하는지 확인 후 설치
        - ![image](https://user-images.githubusercontent.com/11372675/206908447-3907506d-a063-468a-9648-d87dd5e03547.png)
    3. VS 실행시키고 Tools > Extensions and Updates...에서 Extension 확인 가능
    4. View > Toolbars > Unreal VS 활성화
        - ![image](https://user-images.githubusercontent.com/11372675/206908557-fff62664-695c-4435-8f36-8392c275c84b.png)
    5. Toolbar 우클릭 > Customize... > Commands 탭 > Toolbar에서 UnrealVS 선택 > Add Commands... 선택
    6. Category에서 Extension 선택 > Build startup Project 선택 > 확인
        - ![image](https://user-images.githubusercontent.com/11372675/206908606-9ceb311c-4f36-47aa-948e-31c8e7324425.png)
    7. 6번 방법으로 Compile Sningle File추가
    8. 설정 완료
        - ![image](https://user-images.githubusercontent.com/11372675/206908639-e1d82915-a448-4d04-bb13-d38312413aff.png)



# 기타
## visual assist X
- IntelliSense가 너무 느리고 불편해서 Visual Assist라는 확장프로그램을 사용하는 사람이 꽤 있음
- VAssistX > Visual Assist X Options > Advanced > Corrections에서 Format After Paste를 꺼야한다고 함
    - 안 그러면 VS X가 소스 코드 형식을 자동으로 결정해서 잘못된 형식이 될 수 있다고 함

## VCommander
- UPROPERTY같은 매크로 후에 들여쓰기가 자동으로 되는 문제를 해결할 수 있음
1. VCommander 설치
    - https://vlasovstudio.com/visual-commander/
2. UE4 vs extension 다운로드
    - https://github.com/zenoengine/ue4-vs-extensions
3. VCmd > Importt > UE4 vs extension(.vcmd확장자) 임포트
4. Extensions를 열어 설치된 커맨드 확인
