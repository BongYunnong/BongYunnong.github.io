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


# 온라인 세션
## 세션 생성하기
- PuzzlePlatformsGameInstance.h
    ``` C++
    #include "OnlineSubsystem.h"
    ...
    private:
        IOnlineSessionPtr SessionInterface;
        void OnCreateSessionComplete(FName SessionName, bool Success);
    ```
- PuzzlePlatformsGameInstance.cpp
    ```C++
    #include "Interfaces/OnlineSessionInterface.h"
    #include "OnlineSessionSettings.h"
    ...
    void UPuzzlePlatformsGameInstance::Init()
    {
        IOnlineSubSystem* Subsystem = IOnlineSubsystem::Get();
        if(Subsystem != nullptr){
            UE_LOG(LogTemp, Warning, Text("Found subsystem %s"), *Subsystem->GetSubsystemName().ToString());
            SessionInterface = Subsystem->GetSessionInterface();
            if(SessionInterface.IsValid())
                SessionInterface->OnCreateSessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnCreateSessionComplete);
        }
    }

    void UPuzzlePlatformsGameInstance::Host()
    {
        if(SessionInterface.IsValid()){
            FOnlineSessionSettings SessionSettings;
            SessionSettings.bIsLANMatch = true;
            SessionSettings.NumPublicConnections = 2;
            SessionSettings.bShouldAdvertise = true;

            SessionInterface->CreateSession(0,TEXT("My Session Game"),SessionSettings);
        }
    }
    void UPuzzlePlatformsGameInstance::OnCreateSessionComplete(FName SessionName, bool Success)
    {
        if(!Success)
        {
            UE_LOG(LogTemp, Warning, Text("Could not create Session"));
            return;
        }
        ...
    }
    ``` 
## 세션 제거하기
- PuzzlePlatformsGameInstance.h
    ``` C++
    private:
        void OnCreateSessionComplete(FName SessionName, bool Success);
        void OnDestroySessionComplete(FName SessionName, bool Success);
        void CreateSession();
    ```
- PuzzlePlatformsGameInstance.cpp
    ``` C++
    const static FName SESSION_NAME = TEXT("My Session Game");
    void UPuzzlePlatformsGameInstance::Init()
    {
        ...
        SessionInterface->OnCreateSessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnCreateSessionComplete);
        SessionInterface->OnDestroySessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnDestroySessionComplete);
        ...
    }
    void UPuzzlePlatformsGameInstance::Host()
    {
        if(SessionInterface.IsValid()){
            auto ExistingSession = SessionInterface->GetNamedSession(SESSION_NAME);
            if (ExistingSession == nullptr)
            {
                CreateSession();
            }
            else
            {
                SessinoInterface->DestroySession(SESSION_NAME);
            }
        }
    }
    void UPuzzlePlatformsGameInstance::CreateSession()
    {
        if(SessionInterface.IsValid())
        {
            FOnlineSessionSettings SessionSettings;
            SessionSettings.bIsLANMatch = true;
            SessionSettings.NumPublicConnections = 2;
            SessionSettings.bShouldAdvertise = true;
            SessionInterface->CreateSession(0,TEXT("My Session Game"),SessionSettings);
        }
    }
    void UPuzzlePlatformsGameInstance::OnDestroySessionComplete(FName SessionName, bool Success)
    {
        if(Success)
        {
            CreateSession();
        }
    }
    ```
## 온라인 세션 찾기
- PuzzlePlatformsGameInstance.h
    ``` C++
    ...
    private:
        TSharedPtr<class FOnlineSessionSearch> SessionSearch;
        void OnFindSessionComplete(bool Success);
    ```
- PuzzlePlatformsGameInstance.cpp
    ``` C++
    void UPuzzlePlatformsGameInstance::Init()
    {
        ...
        SessionInterface->OnFindSessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnFindSessionComplete);
        SessionSearch = MakeSharable(new FOnlineSessionSearch());
        if(SessionSearch.IsValid())
        {
            SessionSearch->bIsLanQuery = true;
            UE_LOG(LogTemp, Warning, Text("Starting Find Session"));
            SessionInterface->FindSession(0,SessionSearch.ToSharedRef());
        }
        ...
    }
    
   void UPuzzlePlatformsGameInstance::OnFindSessionComplete(bool Success)
    {
        if(Success && SessionSearch.IsValid())
        {
            UE_LOG(LogTemp, Warning, Text("Finished Find Session"));
            for(const FOnlineSessionSearchResult& SearchResult : SessionSearch->SearchResult)
            {
                UE_LOG(LogTemp, Warning, Text("Found Session names: %s"),*SearchResult.GetSessionIdStr());
            }
        }
        
    }
    ```

## Scroll box같은 Widget에 자식 넣는 방법
``` c++
부모Widget->AddChild(자식Widget);
```
- 부모 Widget은 UUserWidget을 상속받음
- 자식 Widget은 UWidget을 상속받음

## TOPtional
- nullptr이 가능한데, 포인터는 아닌 것을 표현할 때 사용
- 값이 설정되지 않으면 0, -1같은 값이 되는 것이 아니라 nullptr이 나옴(Nullable같은데?)
- 아래 예시는 서버 목록에서 어떤 방을 선택했을 때 그 index를 받으려하는 예시
- UMainMenu.h
    ``` C++
    public:
        void SelectIndex(uint32 Index);
    private:
        TOptional<uint32> SelectedIndex;
    ```
- UMainMenu.cpp
    ``` C++
    void UMainMenu::SelectIndex(uint32 Index)
    {
        SelectedIndex = Index;
    }
    void UMainMenu::JoinServer()
    {
        if(SelectedIndex.IsSet())
        {
            UE_LOG(LogTemp, Warning, TEXT("Selected Index %d"), SelectdIndex.GetValue());
        }else
        {
            UE_LOG(LogTemp, Warning, TEXT("Selected Index not set %d"), SelectdIndex.GetValue());
        }
    }
    void UMainMenu::SetServerList(TArray<FString> ServerNames)
    {
        UWorld* World = this->GetWorld();
        if(!ensure(World!=nullptr)) return;

        ServerList->ClearChildren();

        uint32 i = 0;
        for(const FString& ServerName : ServerNames)
        {
            UServerRow* Row = CreateWidget<UServerRow>(World, ServerRowClass);
            if(!ensure(Row!=nullptr)) return;
            Row->ServerName->SetText(FText::FromString(ServerName));
            Row->Setup(this,i);
            ++i;
            ServerList->AddChild(Row);
        }
    }
    ```
- ServerRow.h
    ``` C++
    public:
        UPROPERTY(meta=(BindWidget))
        void UTextBlock* ServerName;

        void setup(class UMainMenu* Parent, uint32 index);
    private:
        UPROPERTY(meta=(BindWidget))
        class UButton* RowButton;
    
        UPROPERTY()
        class UMainMenu* Parent;
        uint32 Index;

        UFUNCTION()
        void OnClicked();
    ```
- ServerRow.cpp
    ``` C++
    void UServerRow::setup(class UMainMenu* Parent, uint32 index)
    {
        Parent = InParent;
        Index = InIndex;
        RowButton->OnClicked.AddDynamic(this, &UServerRow::OnClicked);
    }
    void UServerRow::OnClicked()
    {
        Parent->SelectIndex(Index);
    }
    ```

## Sessiono에 Join하기
- 원래는 join에 ip address를 넣었는데, 이제는 index를 넣어서 들어갈거임
- UMainMenu.cpp
    ``` C++
    void UMainMenu::JoinServer()
    {
        if(SelectedIndex.IsSet() && MenuInterface !=nullptr)
        {
            UE_LOG(LogTemp, Warning, TEXT("Selected Index %d"), SelectdIndex.GetValue());
            MenuInterface->Join(SelectedIndex.GetValue());
        }else
        {
            UE_LOG(LogTemp, Warning, TEXT("Selected Index not set"));
        }
    }
    ```
- PuzzlePlattformsGameInstance.h
    ```C++
    void OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result);
    ```
- PuzzlePlattformsGameInstance.cpp
    ``` C++
    void UPuzzlePlatformsGameInstance::Init()
    {
        ...
        SessionInteface->OnJoinSessinoCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnJoinSessionComplete)
    }
    void UPuzzlePlatformsGameInstance::Join(uint32 Index)
    {
        if(!SessionInterface.IsValid()) return;
        if(!SessionSearch.IsValid()) return;

        if(Menu != nullptr)
            Menu -> Teardown();

        SessionInterface->JoinSession(0,SESSION_NAME, SessionSearch->SearchResults[Index]);
    }
    void UPuzzlePlatformsGameInstance::OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
    {
        if(!SessionInteface.IsValid()) return;
        FString Address;
        if(SessionInterface->GetResolvedConnectString(SessionName,Address))
        {
            UE_LOG(LogTemp, Warning, TEXT("Could not get connect string."));
        }

        UEgine* Engine = GetEngine();
        if(!ensure(Engine!=nullptr)) return;

        Engine->AddOnScreenDebugMessage(0,5,FColor::Green, FString::Printf(TEXT("Joining %s"),*Address));

        APlayerController* PlayerController  GetFirstLocalPlayerController();
        if(!ensure(PlayerController != nullptr)) return;

        PlayerController->ClientTravel(Address, ETravelType::Travel_Absolute);
    }
    ```

## Steam OSS 활성화하기
- Online Subsystem Steam 플러그인 활성화
- 프로젝트이름.Build.cs에 "OnelineSubsystemSteam" 추가
    - DefaultEngine.ini에 추가됨 -> DefaultPlatformService를 NULL에서 Steam으로 변경
        ```
        [OnlineSubsystem]
        DefaultPlatformService=Steam
        ```
    - 추가적으로, OnelineSubSystem 문서에 가서 EndResult를 복붙
        - 무료 사용자 설정임
- Steam 서비스를 이용하려면 cmd에서 "엔진위치" "uproject위치" -game -log 로 실행해야함
    - 여러개의 인스턴스를 실행할수는 없어서, Steam 서비스를 이용하려면 -game -log -nosteam 옵션을 두자
    - 컴퓨터당 하나의 Steam 인스턴스만 가질 수 있기 때문..->테스트할때는 nosteam으로

## Steam로비의 Presence
- Steam OSS를 활성화해도 다른 네트워크의 컴퓨터와 연결은 안 됨
- Presence는 Steam의 lobby를 활성화해주는 역할을
- PuzzlePlatformsGameInstance.cpp
    ``` C++
    void UPuzzlePlatformsGameInstance::CreateSession()
    {
        if(SessionInteface.IsValid())
        {
            FOnlineSessionSettings SesseionSettings;
            if(IOnlineSubsystem::Get()->GetSubsystemName() == "NULL")
            {
                SessionSettings.bIsLANMatch = true;
            }else{
                SessionSettings.bIsLANMatch = false;
            }
            SessionSettings.NumPublicConnections = 2;
            SessionSettings.bShouldAdvertise = true;
            SessionSettings.bUsesPresence = true;   // 요거 true

            SessionSettings.bUseLobbiesIfAvailable=true;    // 4.27버전부터는 이 코드를 추가

            SessionInterface->CreateSession(0,SESSION_NAME, SessionSettings);
        }
    }
    void UPuzzlePlatformsGameInstance::RefreshServerList()
    {
        SessionSearch = MakeShareable(new FOnlineSessionSearch());
        if(SessionSearch.IsValid())
        {
            //SessionSearch->bIsLanQuery = true;
            SessionSearch->MaxSearchResults = 10*0;
            SessionSearch->QuerySettings.Set(SEARCH_PRESENCE,true, EOnlineComparisonOp::Equals);
            UE_LOG(LogTemp, Warning, TEXT("Starting Find Session."));
            SessionInterface->FindSession(0, SessionSearch.ToSharedRef());
        }
    }
    ```

- HostSession->FindSession->JoinSession->GetResolvedConnectString->ClientTravel

## ServerList 강조표시
- 버튼 이벤트
    - OnHovered,OnUnhovered를 써도 되지만 Tick에서 ishovered를 써도 됨
    - OnClicked
- ServerRow.h
    ``` C++
    public:
        UPROPERTY(BlueprintReadOnly)
        bool Selected =false;
    ```
- MainMenu.h
    ``` C++
    void UpdateChildren();
    ```
- MainMenu.cpp
    ``` C++
    void UMainMenu::SelectIndex(uint32 Index)
    {
        SelectedIndex = Index;
        UpdateChildren();
    }
    void UMainMenu::UpdateChildren()
    {
        for(int32 i = 0; i < ServerList->GetChildrenCount();i++)
        {
            auto Row = Cast<UServerRow>(ServerList->GetChildAt(i));
            if(Row != nullptr)
            {
                Row->Selected = ( SelectedIndex.IsSet() && SelectedIndex.GetValue() == i);
            }
        }
    }
    ```
- Blueprint에서 Selected와 IsHovered를 사용해서 Tick에서 강조 효과 넣기


## 더 많은 서버 정보
- setServerList에서 Name으로 받던 것을 FServerData라는 구조체를 받도록 변경
``` C++
USTRUCT()
struct FServerData
{
    GENERATED_BODY()
    
    FString Name;
    uint16 CurrentPlayers;
    uint16 MaxPlayers;
    FString HostUsername;
};
```
- MainMenu.cpp
    ``` C++
    void UPuzzlePlatformsGameInstance::OnFindSessionsComplete(bool Success)
    {
        if(Success && SessionSearch.IsValid() && Menu != nullptr)
        {
            UE_LOG(LogTemp, Warning, TEXT("Finished Find Session"));

            TArray<FServerData> ServerNames;
            for(const FOnlineSessionSearchResult& SearchResult : SessionSearch->SearchResults)
            {
                UE_LOG(LogTemp, Warning, TEXT("Found Session names : %s"), *SearchResult.GetSessionIdStr());
                FServerData Data;
                Data.Name = SearchResult.GetSessionIdStr();
                Data.MaxPlayers = SearchResult.Session.SessionSettings.NumOpenPublicConnections;    // 세션에서 사용 가능한 자리의 수를 의미
                Data.CurrentPlayers = Data.MaxPlayers - SearchResult.Session.NumOpenPublicConnections;
                Data.HostUsername = SearchResult.Session.OwningUserName;
                ServerNames.Add(Data);
            }
            Menu->SetServerList(ServerNames);
        }
    }
    void UMainMenu::SetServerList(TArray<FServerData> ServerNames)
    {
        UWorld* World = this->GetWorld();
        if(!ensure(World != nullptr)) return;

        ServerList->ClearChildren();

        uint32 i = 0;
        for(const FServerData& ServerData : ServerNames)
        {
            UServerRow* Row = CreateWidget<UServerRow>(World, ServerRowClass);
            if(!ensure(Row != nullptr)) return;

            Row->ServerName->SetText(FText::FromString(ServerData.Name));
            Row->HostUser->SetText(FText::FromString(ServerData.HostUserName));
            Row->ConnectionFraction->SetText(FText::FromString(FString::Printf(TEXT("%d\
            %d",ServerData.CurrentPlayers,ServerData.MaxPlayers))));
            Row->Setup(this,i);
            ++i;
            ServerList->AddChild(Row);
        }
    }
    ```
- ServerRow.h
    ``` C++
    UPROPERTY(meta=(BindWidget))
    class UTextBlock* ServerName;
    UPROPERTY(meta=(BindWidget))
    class UTextBlock* HostUser;
    UPROPERTY(meta=(BindWidget))
    class UTextBlock* ConnectionFraction;
    ```

## 커스텀 세션 세팅
- custom session setting으로 원하는 값을 세션에 넘길 수 있다.
- MenuInterface.h
    ``` C++
    // virtual void Host() = 0;
    virtual void Host(FString ServerName) = 0;
    ```
- PuzlePlatformsGameInstance.h
    ``` C++
    FString DesiredServerName;
    ```
- PuzlePlatformsGameInstance.cpp    
    ``` C++
    const static FName SERVER_NAME_SETTINGS_KEY = TEXT("ServerName");
    void UPuzzlePlatformsGameInstance::Host(FString ServerName)
    {
        DesiredServerName = ServerName;
        if(SessionInterface.IsValid())
        {
            ...
        }
    }
    void UPuzzlePlatformsGameInstance::CreateSession()
    {
        if(SessionInteface.IsValid())
        {
            ...
            SessionSettings.Set(SERVER_NAME_SETTINGS_KEY,DesiredServerName,EOnlineDataAdvertisementType::ViaOnlineServiceAndPing);

            SessionInterface->CreateSession(0,SESSION_NAME, SessionSettings);
        }
    }

    void UPuzzlePlatformsGameInstance::OnFindSessionsComplete(bool Success)
    {
        if(Success && SessionSearch.IsValid() && Menu != nullptr)
        {
            ...
            for(const FOnlineSessionSearchResult& SearchResult : SessionSearch->SearchResults)
            {
                ...
                Data.HostUsername = SearchResult.Session.OwningUserName;

                FString ServerName;
                if(SearchResult.Session.SessionSettings.Get(SERVER_NAME_SETTINGS_KEY,ServerName))
                {
                    Data.Nmae = ServerName;
                }else{
                    Data.Nmae = "Could not find name";
                }
                ServerNames.Add(Data);
            }
            Menu->SetServerList(ServerNames);
        }
    }
    ```
- Host 이름을 지정할 수 있는 Menu를 만들도록 하자
    - EditableTextBox 타입의 ServerHostName사용
    ``` C++
    void MeinMenu::HostServer(){
        if(MenuInterface!=nullptr){
            FString ServerName = ServerHostName->Text.ToString();
            MenuInterface->Host(ServerName);   
        }
    }
    ```

## 게임모드와 멀티플레이
- Host, Join을 하면 바로 본 게임으로 넘어가는데.. 어떤 Lobby에서 모인 후 다 같이 가면 좋지 않을까? -> GameMode를 사용하자
    - PreLogin, PostLogin, RestartPlayer 등 다양한 기능이 존재함
- GameModeBase는 GameMode의 축소판

- 로비를 위한 게임모드 만들기
    - PuzzlePlatformsGameMode를 상속받는 "LobbyGameMode" 클래스 생성
    - Lobby 맵에서 GameMode override하도록 설정
- LobbyGameMode.h
    ``` C++
    GENERATED_BODY()
    public:
        virtual void PostLogin(APlayerController* NewPlayer) override;
        virtual void Logout(AController* Exiting) override;
    private:
        uint32 NumberOfPlayers = 0;
    ```

- LobbyGameMode.cpp
    ``` C++
    void ALobbyGameMode::PostLogin(APlayerController* NewPlayer)
    {
        Super::PostLogin(NewPlayer);
        ++NumOfPlayers;
        if(NumOfPlayers>=3)
        {
            //UE_LOG(LogTemp, Warning, TEXT("Reached 3 players!"));
            UWorld* World = GetWorld();
            if(!ensure(World!=nullptr)) return;

            bUseSeamlessTravel=true;
            World->ServerTravel("/Game/PuzzlePlatforms/Maps/Game?listen");
        }
    }
    void ALobbyGameMode::Logout(AController* Exiting)
    {
        Super::Logout(Exiting);
        --NumOfPlayers;
    }
    ```
- Non-seamless travel
    - 서버가 모든 클라이언트의 연결을 끊고 새 맵에 다시 연결하도록 지시한 후 자체적으로 새로운 맵을 로드
- Transition map
    - 맵을 로드할 수 없을 때 언제든지 로드되는 맵
    - 큰 두개의 맵을 동시에 로드하는 것은 무리가 있으니 아주 작은 맵으로 전환하는 과정이 필요
    - ProjectSetting에서 Default Transition Map을 지정할 수 있음

## 언리얼에서 디버깅하기
- Editor Symbols for debugging 설치 옵션 활성화
- 중단점 설정 후 Debug->Attach to process -> UE4Editor

- 지금 프로젝트에는 NumOpenPublicConnections가 제대로 동작하지 않는 문제점이 있었음..
    - 디버깅을 해보면 Session이 null이라는 것을 알 수 있고, 서버를 열 때 사용한 SessionName과 우리가 정한 SESSION_NAME 상수가 다른 값이라서 그랬다는 것을 알 수 있음
    - SessionName은 palyerState에서 가져온 값이라는 것이 확인되었으니, 그거에 맞게 SESSION_NAME을 "Game"으로 수정해주면 됨
- 위와 같은 문제는 SESSION_NAME을 임의로 설정했기 때문임
    - 더 안전한 방법은, SESSION_NAME같은 상수를 만들지 말고 이미 제공되는 NAME_GameSession을 사용하는 것(엔진 버전이 달라질 때도 유효하도록)

- PuzzlePlatformsGameInstance.cpp
    ``` C++
    void UPuzzlePlatformsGameInstance::CreateSession()
    {
        if(SessionInteface.IsValid())
        {
            FOnlineSessionSettings SesseionSettings;
            if(IOnlineSubsystem::Get()->GetSubsystemName() == "NULL")
            {
                SessionSettings.bIsLANMatch = true;
            }else{
                SessionSettings.bIsLANMatch = false;
            }
            SessionSettings.NumPublicConnections = 2;
            SessionSettings.bShouldAdvertise = true;
            SessionSettings.bUsesPresence = true;   // 요거 true

            SessionSettings.bUseLobbiesIfAvailable=true;    // 4.27버전부터는 이 코드를 추가

            //SessionInterface->CreateSession(0,SESSION_NAME, SessionSettings);
            SessionInterface->CreateSession(0,NAME_GameSession, SessionSettings);
        }
    }
    ```

- 그래도 문제가 있다면
    - IOnelineSession::UpdateSession 정보를 수동으로 추적하여 세션 설정 업데이트 필요

## 인원수 제한, 몇 초 후 게임 시작 등..
- LobbyGameMode.h
    ``` C++
    private:
        void StartGame();

        FTimerHandle GameStartTimer;
    ```
- LobbyGameMode.cpp
    ``` C++
    #include "TimerManager.h"
    #include "PuzzlePlatformsGameInstance.h"

    void ALobbyGameMode::PostLogin(APlayerController* NewPlayer)
    {
        Super::PostLogin(NewPlayer);
        ++NumberOfPlayers;
        if(NumOfPlayers >= 2)
        {
            GetWorldTimerManager().SetTimer(GameStartTimer,this,&ALobbyGameMode::StartGame, 10);
        }
    }
    void ALobbyGameMode::StartGame()
    {
        auto GameInstance = Cast<PuzzlePlatfomsGameInstance>(GetGameInstance());
        if(GameInstance==nullptr) return;
        GameInstance->StartSession();

        UWorld* World = GetWorld();
        if(!ensure(World!=nullptr)) return;

        bUseSeamlessTravel=true;
        World->ServerTravel("/Game/PuzzlePlatforms/Maps/Game?listen");
    }
    ```
- 아래는 세션이 시작했다는 것을 알림으로 서버 목록을 보고있는 다른 플레이어들이 들어올 수 없도록 하는 기능

- PuzzlePlatformsGameInstance.h
    ``` C++
    void StartSession();
    ```
- PuzzlePlatformsGameInstance.cpp
    ``` C++
    void PuzzlePlatformsGameInstance::StartSession()
    {
        if(SessionInterface.IsValid())
        {
            SessionInterface->StartSession(NAME_GameSession);
        }
    }
    ```

## 서버 해제
- PuzzlePlatformsGameInstance.h
    ``` C++
    private:
        void OnNetworkFailure(UWorld* World, UNetDriver* Net, ENetWorkFailure::Type FailureType, const FString& ErrorString);
    ```
- PuzzlePlatformsGameInstance.cpp   
    ``` C++
    void UPuzzlePlatformsGameInstance::Init()
    {
        ...
        if(SubSystem!=nullptr)
        {
            ...
            if(SessionInterface.IsValid())
            {
                ...
                SessionInterface->OnDestroySessionCompleteDelegates.AddUObject(this, &UPuzzlePlatformsGameInstance::OnDestroySessionComplete);
            }
        }
        if(GEngine!=nullptr)
        {
            GEngine->OnNetworkFailure().AddUObject(this, &UPuzzlePlatformsGameInstance::OnNetworkFailure);
        }
    }
    void UPuzzlePlatformsGameInstance::OnDestroySessionComplete(FName SessionName, bool Success)
    {
        if(Success)
        {
            CreateSession();
        }
    }
    void UPuzzlePlatformsGameInstance::OnNetworkFailure(UWorld* World, UNetDriver* Net, ENetWorkFailure::Type FailureType, const FString& ErrorString){
        LoadMainMenu();
    }
    ```
- SessionInterface가 아니라 GEngine의 NetworkFailure를 사용한 것은 실제로 모든 연결 로직이 엔진 수준에서 발생하기 때문

## Actor Role
- Server에서는 모두가 Authority를 가짐
- Client1의 플레이어1은 Autonomous Proxy(자율프록시), 플레이어2는 Simulated Proxy이다.
- 반대로 Client2의 플레이어 1은 Simulated Proxy, 플레이어2는 Autonomous Proxy이다.

## Lag 설정
- ` 명령어로 "NetPktLag=1000"을 입력하면 랙을 실험할 수 있음
    - 밀리초 단위 : 1000은 1초

- Lag Glitc
hing
    - Server와 Client가 모두 Simulate하는 환경에서, Client는 가속하는 것을 Server에게 보냄. 이때 Lag 때문에 Server에서는 Client가 가속한다는 것을 늦게 알아차리고, 값을 변경하고, Client에게 다시 보내게 되는데, 그 와중에 Client는 계속 가속을 하고있었으니, 다시 이전에 있던 자리로 돌아가게 되어서 Glitching이 생김

## 구조체 복제하기
- 수도코드
    - OnTick
        1. Create a new Move
        2. Save to a list of unacknowledged moves
        3. Send the move to the server
        4. Simulate the move locally
    - OnReceiveMove
        1. Check that the move is valid(no cheating!)
        2. Simulate the move
        3. Send the canonical State to the Clients
    - OnReceiveServerState
        1. Remove all moves included in state
        2. Reset to server state
        3. Replay/Simulate unacknowledged moves
- GoKart.h
    ``` C++
    USTRUCT()
    struct FGoKartMove
    {
        GENERATED_USTRUCT_BODY()
        UPROPERTY()
        float Thottle;
        UPORPERTY()
        float SteeringThrow;

        UPROPERTY()
        float DeltaTime;
        UPROPERTY()
        float Time;
    };
    USTRUCT()
    struct FGoKartState
    {
        GENERATED_USTRUCT_BODY()
        UPROPERTY()
        FTransform Transform;
        UPROPERTY()
        FVector Velocity;
        UPROPERTY()
        FGoKartMove LastMove;
    };
    ...
    void SimulateMove(FGoKartMove Move);

    void ApplyRotation(float DeltaTime, float SteeringThrow);

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_SendMove(FGoKarttMove Move);

    UPROPERTY(ReplicatedUsing = OnRep_ServerState)
    FGoKartState ServerState;

    UFUNCTION()
    void OnRep_ServerState();

    FVector Velocity;

    float Throttle;
    float SteeringThrow;
    
    ```
- GoKart.cpp
    ``` C++
    void AGoKart::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);

        DOREPLIFETIME(AGoKart, ServerSate);
    }
    void AGoKart::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);

        if(IsLocallyControlled())
        {
            FGoKartMove Move;
            Move.DeltaTime = DeltaTime;
            Move.SteeringThrow = SteeringThrow;
            Move.Throttle = Throttle;

            Server_SendMove(Move);
            SimulateMove(Move);
        }

        DrawDebugString(GetWorld(),FVector(0,0,100), GetEnumText(Role), this, FColor::White, DeltaTime);
    }
    void AGoKart::SimulateMove(FGoKartMove Move)
    {
        FVector Force = GetActorForwardVector() * MaxDrivingForce * Move.Throttle;

        Force += GetAirResistance();
        Force += GetRollingResistance();

        FVector Acceleration = Force / Mass;

        Velocity = Velocity + Acceleration * Move.DeltaTime;

        ApplyRotation(Move.DeltaTime, Move.SteeringThrow);
        UpdateLocationFromVelocity(Move.DelatTime);
    }
    void AGoKart::OnRep_ServerState()
    {
        SetActorTransform(ServerState.Transform);
        Velocity = ServerState.Velocity;
    }
    void AGoKart::MoveForward(float value)
    {
        Throttle = Value;
    }
    void AGoKart::MoveRight(float value)
    {
        SteeringThrow = Value;
    }
    void AGoKart::Server_SendMove_Implementation(FGoKartMove Move)
    {
        SimulateMove(Move);
        ServerState.LastMove = Move;
        ServerState.Transform = GetActorTransform();
        ServerSTate.Velocity = Velocity;
    }
    void AGoKart::Server_SendMove_Validate(FGoKartMove Move)
    {
        return true;
    }
    void AGoKart::ApplyRotation(float DeltaTime, float SteeringThrow)
    {
        float DeltaLocation = FVector::DotProduct(GetActorForwardVector(), Velocity) * DeltaTme;
        float RotationAngle = DeltaLocation / MinTurningRadius * SteeringThorw;
        EQuat RotationDelta(GetActorUpVector(), RotationAngle);

        Velocity = RotationDelta.RotateVector(Velocity);
        AddActorWorldRotation(RotationDelta);
    }
    void AGoKart::UpdateLocationFromVelocity(float DeltaTime)
    {
        FVector Translation = Velocity * 100 * DeltaTime;
        FHitResult Hit;
        AddVectorWorldOffset(Translation, true, &Hit);
        if(Hit.IsValidBlockingHit())
        {
            Velocity = FVector::ZeroVector;
        }
    }
    ```
## Queue
- 클라이언트가 서버보다 앞서나갈 수 있도록
- GoKart.h
    ``` C++
    void ClearAcknowledgedMoves(FGoKartMove LastMove);

    TArray<FGoKartMove> UnacknowledgedMoves;
    ```
- GoKart.cpp
    ``` C++
    FGoKartMove AGoKart::CreateMove(float DeltaTIme )
    {
        FGoKartMove = Move;
        Move.DeltaTime = DeltaTime;
        Move.SteeringThrow = SteeringThrow;
        Move.Throttle = Throttle;
        Move.Time = GetWorld()->TimeSeconds;
        //Move.Time = AGameStateBase::GetServerWorldTimeSeconds();

        return Move;
    }
    void AGoKart::OnRep_ServerState()
    {
        SetActorTransform(ServerState.Transform);
        Velocity = ServerState.Velocity;

        ClearAcknowledgeMoves(ServerState.LastMove);
        
        for(const FGoKartMove& Move : UnacknowledgedMoves)
        {
            SimulateMove(Move);
        }
    }
    void AGoKart::ClearAcknowledgedMoves(FGoKartMove LastMove)
    {
        TArray<FGoKartMove> NewMoves;
        for(const FGoKartMove& Move : UnacknowledgedMoves)
        {
            // LastMove보다 더 이전의 Move 정보는 필요가 없음음
            if(Move.Time > LastMove.Time)
            {
                NewMoves.Add(Move);
            }
        }
        UnacknowledgedMoves = NewMoves;
    }
    void AGoKart::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        /*
        if(IsLocallyControlled())
        {
            FGoKartMove Move = CreateMove(DelatTime);

            if(!HasAuthority())
            {
                UnacknowledgedMoves.Add(Move);
                UE_LOG(LogTemp, Warning, TEXT("Queue Length : %d"), UnacknowledgedMoves.Num());
                SimulateMove(Move);
            }
            Server_SendMove(Move);
        }
        DrawDebugString(GetWorld(), FVector(0,0,100), GetEnumText(Role), this, FColor::White, DeltaTime);
        */
        if(Role==ROLE_AutonomousProxy)
        {
            FGoKartMove Move = CreateMove(DeltaTime);
            SimulateMove(Move);
            UnacknowledgedMoves.Add(Move);
            Server_SendMove(Move);
        }
        // if we are server and in control of the pawn
        if(Role==ROLE_Authority && IsLocallyControlled())
        {
            FGoKartMove Move = CreateMove(DeltaTime);
            Server_SendMove(Move);
        }
        if(Role==ROLE_SimulatedProxy)
        {
            SimulateMove(ServerState.LastMove);
        }
    }
    ```

- ActorComponent에서 actor에 접근하고싶다면 GetOwner()를 사용
- Base 액터가 Replicate라고 해서 그 안의 Component들이 다 자동으로 Replicate되는 것 아님 -> 수동으로 replicate하겠다고 설정 필요
- ActorComponent에는 GetRole()대신 GetOwnerRole()이 있음
    - RemoteRole은 GetOwner()->GetRemoteRole()해야함..