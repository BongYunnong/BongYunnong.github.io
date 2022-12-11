# FVector에 3D Widget 속성 넣기
``` C++
UPROPERTY(EditAnywhere, Meta=(MakeEditWidget=true))
FVector targetLocation;
```
- ![image](https://user-images.githubusercontent.com/11372675/205005657-cca8479f-8d65-4ec5-ba0b-5fd4332cccf6.png)

# 언리얼의 좌표 단위
- cm를 사용함

# Normalize와 SafeNormal의 차이
- Normalize는 벡터를 변형함
- SafeNormal은 새로운 벡터를 만듦
- 벡터 자체를 변형하지 않는 것이 좋은 습관(버그찾기 힘듦)

# Local Position을 Global Position으로 변경하기
``` C++
FVector GlobalTargetLocation = GetTransform().TransformPosition(TargetLocation);
```

# NAT
- 처음에 IP를 설계할 때 이렇게 인터넷이 활성화 될 줄 몰랐음
    - IPv4를 사용하다가 IPv6를 새로 개발함
    - 하지만 IPv4를 사용하던 이전 버전과의 호환성 때문에 모든 곳에서 교체가 이루어지지는 않았음
- 위의 문제를 해결하기 위해서, 글로벌 IP와 로컬 IP를 나눔
    - 일반적으로 ISP(Internet Service Provider - ex. SK, KT, LG유플러스...)가 라우터에 글로벌 IP 주소를 제공
    - 한 네트워크 안의 여러 기기는 로컬 IP 주소를 가짐
- NAT(Network Address Translation)
    - 글로벌 IP 주소를 로컬 IP 주소로 변환함
    - 주의할 점은, 외부로 나가는 연결을 해줄 장치가 없으면 글로벌 주소에서 하나의 특정 장치로 이동하는게 불가능함
- 하마치 소프트웨어를 활용
    - 하마치를 통해 글로벌 네트워크에 걸친 LAN을 만들 수 있음
    - 즉, 가상 로컬 네트워크를 만듦

# Hamachi 사용
- 5명까지 무료

- www.vpn.net
    - ![image](https://user-images.githubusercontent.com/11372675/205015839-084d62ec-1ca8-4ef7-8dc9-7ec72cec1799.png)
- 다운로드
    - ![image](https://user-images.githubusercontent.com/11372675/205015746-b1a003cc-4e95-4034-8e8d-a6c441cb8283.png)
- Sign Up
- Hamachi 실행
- 전원 켜기
- Sign in
- Create New Network
    - ![image](https://user-images.githubusercontent.com/11372675/205016201-99484ea2-0cf2-4ae0-b4b0-c829b5fb8894.png)
    - network Id와 암호 설정
- 다른 컴퓨터에서 접속
    - Join an existing network
    - network Id와 암호 입력
- 접속 확인
    - ![image](https://user-images.githubusercontent.com/11372675/205017038-53bc82b2-2da8-4f68-93fd-174ffcb929ea.png)

## 언리얼에서 사용하기
- 한 컴퓨터에서 Server를 열기
    > "엔진경로" "uproject경로" -server -log
- 다른 컴퓨터에서는 Hamachi에서 서버의 IPv4 주소를 복사하여 cmd에서 아래 명령어 수행
    > "엔진경로" "uproject경로" IPv4주소 -game -log
- 하마치로 테스트를 하려면 다른 플레이어가 내 프로젝트 파일을 가지고있어야함 -> github을 사용할 수 있겠다.

# UPROPERTY와 가비지 컬렉터
- 어떤 변수를 UPROPERTY로 설정을 해주어야 메모리를 관리해줄 UObject로 생각하고 가비지 컬렉터가 실행된다.

# UFUNCTION
- 동적 이벤트는 UFUNCTION()을 사용해야함


# nullpointer 처리하기
- nullpointer는 에디터를 아예 종료시켜버리는 위험이 크다.
- 자동적으로 기본적으로 null검사하는 방법
- 텍스트 편집기 nullret 기능
    ``` C++
    if(!ensure(TriggerVolume!=nullptr)) return;
    ```
# NULL과 nullptr차이
- C++11이전 버전에서는 NULL을 포인터가 아닌 정수 0과 동일하게 여김
- int n = NULL은 되지만 int n = nullptr은 에러가 발생(포인터로 인식을하려하기 때문)
- 최근 실무에서는 NULL대신 nullptr로 초기화를 많이 하는 추세이다.

# Visual Studio 단축키
- Ctrl + K , O : 이전 파일로 돌아가기

# CPP에서 Blueprint 찾기
- FindClassFinder라는 함수를 사용하자
- GameInstance 클래스에서 BP_PlatformTrigger를 찾아보자
    - PuzzlePlatformsGameInstance.cpp
        ``` C++
        #include "UObject/ConstructorHelpers.h"
        #include "PlatformTrigger.h"
        UPuzzlePlatformsGameInstance::UPuzzlePlatformsGameInstance(const FObjectInitializer& ObjectInitializer)
        {
            ConstructorHelpers::FClassFinder<APlatformTrigger> PlatformTriggerBPClass(TEXT("/Game/PuzzlePlatforms/BP_PlatformTrigger"));
            if (PlatformTriggerBPClass.Class != nullptr) {
                UE_LOG(LogTemp, Warning, TEXT("FClassFinder founded %s"), *PlatformTriggerBPClass.Class->GetName());
            }
            UE_LOG(LogTemp, Warning, TEXT("GameInsatnce Constructor"));
        }
        ```
    - 결과
        - ![image](https://user-images.githubusercontent.com/11372675/205316554-1c4f2129-47df-4093-a352-aaa40e8f4441.png)
        - 잘 찾네!
- tip) 게임모드 생성자를 보면 이미 FClassFinder를  사용하고있었음을 알고있다.
    ``` C++
    APuzzlePlatformsGameMode::APuzzlePlatformsGameMode()
    {
        static ConstructorHelpers::FClassFinder<APawn> PlayerPawnBPClass(TEXT("/Game/ThirdPerson/Blueprints/BP_ThirdPersonCharacter"));
        if (PlayerPawnBPClass.Class != NULL)
        {
            DefaultPawnClass = PlayerPawnBPClass.Class;
        }
    }
    ```
    - BP_ThirdPersonCharacter를 찾아서 DefaultPawnClass로 지정하는 코드

# Steamworks SDK(software development kit)
- Steam은 Valve사의 게임 플랫폼
- 게임 유통 뿐 아니라 도전과제, 멀티플레이 등의 통합 기능도 제공한다.
- 링크 : https://partner.steamgames.com/
- Steam 계정으로 log-in
- ![image](https://user-images.githubusercontent.com/11372675/206842822-fd505530-096a-4afd-a603-047eb59748cf.png)
- 약관 확인하고 sdk 다운로드
- 원하는 프로젝트의 Encrypted폴더에 압축 풀기
- 원하는프로젝트폴더\sdk\steamworksexample\SteamworksExample.sln실행
    - ![image](https://user-images.githubusercontent.com/11372675/206843097-f437bf5f-3208-475b-ad99-9cd6e6e774f1.png)
- 솔루션 빌드
