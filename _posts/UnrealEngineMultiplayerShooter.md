# Creating a Multiplayer Plugin
## Multiplayer Concept
- Play Standalone : 서버 사용 X
- Play As Listen Server : 클라이언트 하나가 Server가 된다.
- Play As Client : 뒤에 Dedicated Server를 두고 클라이언트는 각각 인스턴스로 실행

- LAN(Local Area Network)
    - local network에서(같은 라우터를 사용하는 네트워크) 네트워크 연결이 됨
- Open Level 노드 
    - Level Name에는 "Lobby"
    - Options에는 "listen"
    - 이러면 listen서버를 열고 Lobby로 이동
- Execute ConsoleCommand
    - Open 127.0.0.1

- 이제 packaging 한 후, 한 클라이언트에서 OpenLevel, 한 클라이언트에서 Execute를 실행하면 LAN 연결이 됨

## Lan Connection
- MPTestingCharacter.h
    ``` C++
    public:
        UFUNCTION(BlueprintCallable)
        void OpenLobby();

        UFUNCTION(BlueprintCallable)
        void CallOpenLevel(const FString& Address);

        UFUNCTION(BlueprintCallable)
        void CallClientTravel(const FString& Address);
    ```
- MPTestingCharacter.cpp
    ``` C++
    #include "Kismet/GameplayStatics.h"
    ...
    void AMPTestingCharacter::OpenLobby()
    {
        UWorld* World = GetWorld();
        if(World)
        {
            World->ServerTravel("/Game/ThridPerson/Maps/Lobby?listen");
        }
    }
    void AMPTestingCharacter::CallOpenLevel(const FSstring& Address)
    {
        UGameplayStatics::OpenLevel(this, *Address);
    }
    void AMPTestingCharacter::CallClientTravel(const FSstring& Address)
    {
        APlayerController* PlayerController = GetGameInstance()->GetFirstLocalPlayerController();
        if(PlayerController)
        {
            PlayerController->ClientTravel(Address,ETravelType::TRAVEL_Absolute);
        }
    }
    ```
    - CallOpenLevel이나ㅣ CallClientTravel이나 둘 다 Listen서버의 Lobby레벨로 들어갈 수 있음

## Online Subsystem
- 로컬 네트워크는 라우터로 연결되어있음
- 라우터는 ISP(Internet Service Provider)와 연결되어있음
    - ISP가 라우터에 public IP를 제공함

- 게임은 어떻게 나와 다른 네트워크에 있는 플레이어를 연결시킬까?
    - 방법은 다양함
        1. 서버가 온라인 상태인 플레이어들의 IP 리스트를 가지고있어서 연결해줌
            - 플레이어 수가 많아질수록 서버 부담이 커짐
        2. ListenServer : 플레이어 한 명이 서버가 됨
            - 서버는 어떤 Service를 제공하는 역할만 함
- Online Subsystem
    - 다른 플레이어의 IP를 몰라도 네트워크 연결을 할 수 있도록 해줌
    - 게임 세션을 열고 다른 플레이어를 연결해주는 어떤 서버의 서비스를 사용할거임
        - 이것을 Steam, Xbox, PS 등에서 제공을 함
        - 그러면 이거를 각각의 플랫폼에 맞춰서 다 공부하고, 변형해야하나? -> 언리얼에서 online Subsystem이라는 통합 서비스 제공
    - 언리얼엔진이 다양한 platform을 처리해서 하나의 code base로 작업할 수 있게 해줌(추상화 됨)

- Platform Services
    - Friends / Achievements / Sessions / ETC
    - 어떤 플랫폼을 타겟팅하고있는지를 Engine.ini파일에 저장함
        ```
        [OnlineSubsystsem]
        DefaultPlatformService = <Platform>
        ```

- Session Interface
    - Creating, Managing, and destroying online sessions
    - Searching for sessions
    - matchmaking
- Session 
    - An instance of the game running on the server
    - Advertised(players can join) or private(invite only)

- Session의 생애주기
    1. Session 생성
    2. player가 join하기를 기다림
    3. Player register
    4. Session 시작
    5. Play
    6. Session 종료
    7. Player unregister
    8. Session Update or Session 제거
- SessionInterface의 함수
    - CreateSession()
    - FindSessions()
    - JoinSession()
    - StartSession()
    - DestroySession()

- Game Plan
    - Click host -SessionSettings-> create session -> Open Lobby
    - Click Join -SearchSettigns-> Find Session -Pick a valid Session-> Join Session -Get the Address-> Client Travel

## Configure For Steam
- https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Online/Steam/
- MenuSystem이라는 C++ Project 생성
- Plugins에서 Online SubSystem Steam 활성화
- MenuSystem.Build.cs에 "OnlineSubsystemSteam", "OnlineSubsystem" 추가
- Config > DeaufaultEngine.ini
    ```
    [/Script/Engine.GameEngine]
    +NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemSteam.SteamNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")

    [OnlineSubsystem]
    DefaultPlatformService=Steam

    [OnlineSubsystemSteam]
    bEnabled=true
    SteamDevAppId=480

    ; If using Sessions
    ; bInitServerOnClient=true

    [/Script/OnlineSubsystemSteam.SteamNetDriver]
    NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
    ```
    - tip) 만약 세션 만드는 것에 실패하면 위의 주석(;)을 제거해보자
- 에디터와 VS를 닫고, Saved, Intermediate, Binaries 제거
    - .uproject 우클릭 후 Generate visual studio project files

## Online Subsystem에 접근하기
- MenuSystemCharacter.h
    ``` C++
    public:
        // Pointer to the online session interface
        //IOnlineSessionPtr OnlineSessionInterface; IOnlineSessionPtr은 TSharedPtr<class IOnlineSession, ESPMode::ThreadSafe>의 typedef
        // IOnlineSessionPtr을 사용하려면 header에서 include하던가 전방선언해야하는데, typedef형태라서 전방선언 못하니 그냥 typedef를 쓰지 않는 방식으로 처리
	    TSharedPtr<class IOnlineSession,ESPMode::ThreadSafe> OnlineSessionInterface;
    ```
- MenuSystemCharacter.cpp
    ``` C++
    #include "OnlineSubsystem.h"
    #include "Interfaces/OnlineSessionInterface.h"
    ...
    AMenuSubsystemCharacter::AMenuSystemCharacter()
    {
        ...
        IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
        if (OnlineSubsystem)
        {
            OnlineSessionInterface = OnlineSubsystem->GetSessionInterface();
            if(GEngine)
            {
                GEngine->AddOnScreenDebugMessage(-1, 15.f, FColor::Blue,
                     FString::Printf(TEXT("Found subsystem %s"),*OnlineSubsystem->GetSubsystemName().ToString())
                );
            }
        }
    }
    ```
- 실행을 해보면 Found Subsystem NULL이라는 메시지가 출력됨(Steam을 사용 못 하고있네)
- 언리얼은 NULL이라는 online subsystem을 가짐
    - LAN 연결을 위해 고안됨
- packaging을 끝낸 exe파일을 실행하면 Steam Subsystem을 찾음


## Creating Session
- Delegate
    - Stores Functions
    - Broadcast
- Session Interface는 Delegate를 활용함

- MenuSystemCharacter.h
    ``` C++
    #include "Interfaces/OnlineSessionInterface.h"
    ...
    public:
        IOnlineSessionPtr OnlineSessionInterface;
    protected:
        UFUNCTION(BlueprintCallable)
        void CreateGameSession();

        void OnCreateSessionComplete(FName SessionName, bool bWasSuccessful);
    private:
        FOnCreateSessionCompleteDelegate CreateSessionCompleteDelegate;
    ```
- MenuSystemCharacter.cpp
    ``` C++
    #include "OnlineSubsystem.h"
    #include "OnlineSessionSettings.h"
    ...
    AMenuSystemCharacter::AMenuSystemCharacter() :
        CreateSessionCompleteDelegate(FOnCreateSessionCompleteDelegate::CreateUObject(this,&ThisClass::OnCreateSessionComplete))// MenuSystemCharacter를 쓰는 대신 ThisClass라고 써도 됨
    {
        ...
    }
    void AMenuSystemCharacter::CreateGameSession()
    {
        if(!OnlineSessionInterface.IsValid())
        {
            return;
        }

        auto ExistingSession = OnlineSessionInterface->GetNamedSession(NAME_GameSession);
        if(ExistingSession != nullptr)
        {
            OnlineSessionInterface->DestroySession(NAME_GameSession);
        }

        OnlineSessionInterface->AddOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegate);

        TSharedPtr<FOnlineSessionSettings> SessionSettings = MakeShareable(new FOnlineSessionSettings());
        SessionSettings->bIsLANMatch=false;
        SessionSettings->NumPublicConnections = 4;
        SessionSettings->bAllowJoinInProgress = true;
        SessionSettings->bAllowJoinViaPresence = true; // Steam서비스는 지역에 맞는 사용자와 연결을 해줌
        SessionSettings->bShouldAdvertise = true;       // 다른 플레이어들이 이 Session을 찾을 수 있도록 함
        SessionSettings->bUsesPresence = true;
        SessionSettings->bUseLobbiesIfAvailable = true;	// 최근 언리얼 버전에서는 이거 없으면 세션을 못 찾는다 함

        const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
        OnlineSessionInterface->CreateSession(*LocalPlayer->GetPreferredUniqueNetId(),NAME_GameSession, *SessionSettings);
    }
    void AMenuSystemCharacter::OnCreateSessionComplete(FName SessionName, bool bWasSuccessful)
    {
        if(bWasSuccessful)
        {
            if(GEngine)
            {
                GEngine->AddOnScreenDebugMessage(
                    -1,
                    15.f,
                    FColor::Red,
                    FString::Printf(TEXT("Created Session : %s"),*SessionName.ToString())
                );
            }
        }
        else
        {
            if(GEngine)
            {
                GEngine->AddOnScreenDebugMessage(
                    -1,
                    15.f,
                    FColor::Red,
                    FString(TEXT("Failed to create Session!"))
                );
            }
        }
    }
    ```
- 만약 .Net관련 문제가 있으면 VS Installer > Individual Components >  .NET Core 3.1 Runtime(LTS)를 설치하자

## Find Session
- 알맞은 Query(질의)를 통해 원하는 Session 찾기
- MenuSystemCharacter.h
    ``` C++
    public:
        UFUNCTION(BlueprintCallable)
        void JoinGameSession();
        void OnFindSessionsComplete(bool bWasSuccessful);
    private:
	    FOnFindSessionsCompleteDelegate FindSessionsCompleteDelegate;
	    TSharedPtr<FOnlineSessionSearch> SessionSearch;
    ```
- MenuSystemCharacter.h
    ``` C++
    AMenuSystemCharacter::AMenuSystemCharacter() :
        CreateSessionCompleteDelegate(FOnCreateSessionCompleteDelegate::CreateUObject(this, &ThisClass::OnCreateSessionComplete)),
        FindSessionsCompleteDelegate(FOnFindSessionsCompleteDelegate::CreateUObject(this, &ThisClass::OnFindSessionsComplete))
    {
        ...
    }
    void AMenuSystemCharacter::JoinGameSession()
    {
        if (!OnlineSessionInterface.IsValid())
            return;

        OnlineSessionInterface->AddOnFindSessionsCompleteDelegate_Handle(FindSessionsCompleteDelegate);

        SessionSearch = MakeShareable(new FOnlineSessionSearch());
        SessionSearch->MaxSearchResults = 10000;
        SessionSearch->bIsLanQuery = false;
	    SessionSearch->QuerySettings.Set(SEARCH_PRESENCE, true, EOnlineComparisonOp::Equals);

        const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
        OnlineSessionInterface->FindSessions(*LocalPlayer->GetPreferredUniqueNetId(), SessionSearch.ToSharedRef());
    }

    void AMenuSystemCharacter::OnFindSessionsComplete(bool bWasSuccessful)
    {
        for (auto Result : SessionSearch->SearchResults)
        {
            FString Id = Result.GetSessionIdStr();
            FString User = Result.Session.OwningUserName;

            if (GEngine)
            {
                GEngine->AddOnScreenDebugMessage(
                    -1,
                    15.f,
                    FColor::Cyan,
                    FString::Printf(TEXT("Id : %s, User : %s"), *Id, *User)
                );
            }
        }
    }
    ```

## Lobby
- 새로운 Level 만들기 "Lobby"
- 이제는 원하는 Session으로 이동할것임
- MenuSystemCharacter.h
    ``` C++
    protected:
    	void OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result);
    private:
        FOnJoinSessionCompleteDelegate JoinSessionCompleteDelegate;
    ```
- MenuSystemCharacter.cpp
    ``` C++
    AMenuSystemCharacter::AMenuSystemCharacter() :
        CreateSessionCompleteDelegate(FOnCreateSessionCompleteDelegate::CreateUObject(this, &ThisClass::OnCreateSessionComplete)),
        FindSessionsCompleteDelegate(FOnFindSessionsCompleteDelegate::CreateUObject(this, &ThisClass::OnFindSessionsComplete)),
        JoinSessionCompleteDelegate(FOnJoinSessionCompleteDelegate::CreateUObject(this, &ThisClass::OnJoinSessionComplete))
    {
        ...
    }
    void AMenuSystemCharacter::CreateGameSession()
    {
        ...
	    SessionSettings->Set(FName("MatchType"), FString("FreeForAll"), EOnlineDataAdvertisementType::ViaOnlineServiceAndPing);
    }

    void AMenuSystemCharacter::OnCreateSessionComplete(FName SessionName, bool bWasSuccessful)
    {    
    	if (bWasSuccessful)
        {
            ...
            UWorld* World = GetWorld();
            if (World)
            {
                World->ServerTravel(FString("/Game/ThirdPerson/Maps/Lobby?listen"));
            }
        }
        else
            ...
    }
    void AMenuSystemCharacter::OnFindSessionsComplete(bool bWasSuccessful)
    {
        if(!OnlineSessionInterface.IsValid())
            return;

        for (auto Result : SessionSearch->SearchResults)
        {
            ...
            FString MatchType;
            Result.Session.SessionSettings.Get(FName("MatchType"), MatchType);
            ...
            if (MatchType == FString("FreeForAll"))
            {
                if (GEngine)
                {
                    GEngine->AddOnScreenDebugMessage(
                        -1,
                        15.f,
                        FColor::Cyan,
                        FString::Printf(TEXT("Joining Match Type: %s"), *MatchType)
                    );
                }
                OnlineSessionInterface->AddOnJoinSessionCompleteDelegate_Handle(JoinSessionCompleteDelegate);
                const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
                OnlineSessionInterface->JoinSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, Result);
            }
        }
    }
    void AMenuSystemCharacter::OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
    {
        if (!OnlineSessionInterface.IsValid())
            return;

        FString Address;
        if (OnlineSessionInterface->GetResolvedConnectString(NAME_GameSession, Address))
        {
            if (GEngine)
            {
                GEngine->AddOnScreenDebugMessage(
                    -1,
                    15.f,
                    FColor::Yellow,
                    FString::Printf(TEXT("Connect String : %s"), *Address)
                );
            }

            APlayerController* PlayerController = GetGameInstance()->GetFirstLocalPlayerController();
            if (PlayerController)
            {
                PlayerController->ClientTravel(Address, ETravelType::TRAVEL_Absolute);
            }
        }
    }
    ```

## Plugin
- Plugin
    - Collection of code and data
    -  Easily enable and disable
    - Per-Project basis
    - Runtime gameplay functionality
    - Editor functionality
    - Made up of one or more moudules
- Modules
    - A distinct unit of C++ code
    - Contains a build file(.Build.cs)
    - Code only(no uassets)
    - Encapsulation(each module has a distinct purpose)
    - Organization
    - Our project itself is a module
- .uproject파일을 열어보면 사용중인 Plugin을 확인할 수 있음
- plugin 만들기
    - Edit > Plugin > Add > Blank 선택
    - Author와 Description 설정
    - Create Plugin
    - Content Browser에서 Show Plugin Content 옵션을 활성화하면 볼 수 있음
    - Visual Studio에도 프로젝트 아래의 Plugins 폴더에 해당 plugin 디렉토리가 생성된 것 확인 가능
- MultiplayerSessions.uplugin
    ``` 
    ...
    "Modules": [
		{
			"Name": "MultiplayerSessions",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		}
	],
	"Plugins": [
		{
			"Name": "OnlineSubsystem",
			"Enabled": true
		},
		{
			"Name": "OnlineSubsystemSteam",
			"Enabled": true
		}
	]
    ```
    - OnlineSubsystem와 OnlineSubsystemSteam 플러그인 사용하겠다고 명시
- MultiplayerSessions.Build.cs
    - OnlineSubSystem과 OnlineSubsystsemSteam 빌드하겠다고 명시
    ``` C#
    PublicDependencyModuleNames.AddRange(
    new string[]
    {
        "Core",
        "OnlineSubsystem",
        "OnlineSubsystemSteam",
        // ... add other public dependencies that you statically link with here ...
    }
    );
    ```

## Creating Our own Online Subsystem
- GameInstance
    - Spawned at game creation
    - Not destroyed until the game is shut down
    - Persists between levels
- UGameInstanceSubsystem을 상속받아서 우리만의 subsystem을 만들 수 있다.
    - https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Subsystems/
    - gameInstanceSubsystem을 만들면 UGameInstance가 만들어질 때 자동으로 만들어지고, Initialize, DeInitialize될 것이다.
        - UGameInstance가 Shutdown되면 자동으로 garbageCollect처리 됨
- MultiplayerSessionsSubsystem클래스 만들기
    - ![image](https://user-images.githubusercontent.com/11372675/208901985-65193872-52e8-4202-919f-c48e5cbeced5.png)
    - ContentBrowser에 보면 MultiplayerSessions Content와 MultiplayerSessions C++ Classes 폴더가 만들어진 것 확인 가능
    - 빨간줄이 뜬다면? : Editor, VS 닫고 MultiplayerSessions 폴더의 Binaries, Intermediate, Saved를 지우고 Generate VisualStudioProjects    
        - .uproject를 다시 실행해보면 rebuild할거냐고 물음 -> 확인하면 빨간 줄 없어짐
- MultiplayerSessionsSubsystem.h
    ``` C++
    #include "Interfaces/OnlineSessionInterface.h"
    ...
    public:
        UMultiplayerSessionsSubsystem();
    protected:
    private:
        IOnlineSessionPtr SessionInterface;
    ```
- MultiplayerSessionsSubsystem.cpp
    ``` C++
    #include "MultiplayerSessionsSubsystem.h"
    #include "OnlineSubsystem.h"
    UMultiplayerSessionsSubsystem::UMultiplayerSessionsSubsystem()
    {
        IOnlineSubsystem* Subsystem = IOnlineSubsystem::Get();
        if (Subsystem)
        {
            SessionInterface = Subsystem->GetSessionInterface();
        }
    }
    ```

## Session interface functions
1. Delegate를 만들고, 적절한 콜백함수를 binding
2. Delegate를 list에 add (ex. AddOnCreateSessionCompleteDelegate_Handle())
3. 위의 예시같은 경우, FDelegateHandle을 return함
4. Delegate를 List에서 clear함 (ex. ClearOnCreateSessionCompleteDelegate_Handle())

- MultiplayerSessionsSubsystem.h
    ``` C++
    public:
        // To handle session functionality. The Menu class will call these
        void CreateSession(int32 NumPublicConnections, FString MatchType);
        void FindSessions(int32 MaxSearchResults);
        void JoinSession(const FOnlineSessionSearchResult& SessionResult);
        void DestroySession();
        void StartSession();
    protected:
        // Internal callbacks for the delegates we'll add to the Online Session Interface delegate list
        // These don't need to be called outside this class
        void OnCreateSessionComplete(FName SessionName, bool bWasSuccessful);
        void OnFindSessionsComplete(bool bWasSuccessful);
        void OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result);
        void OnDestroySessionComplete(FName SessionName, bool bWasSuccessful);
        void OnStartSessionComplete(FName SessionName, bool bWasSuccessful);
    private:
        // To add to the Online Sesison Interface delegate list
        // we'll bind our MultiplayerSessionsSubsystem internal callbacks to these
        FOnCreateSessionCompleteDelegate CreateSessionCompleteDelegate;
        FDelegateHandle CreateSessionCompleteDelegateHandle;
        FOnFindSessionsCompleteDelegate FindSessionsCompleteDelegate;
        FDelegateHandle FindSessionsCompleteDelegateHandle;
        FOnJoinSessionCompleteDelegate JoinSessionCompleteDelegate;
        FDelegateHandle JoinSessionCompleteDelegateHandle;
        FOnDestroySessionCompleteDelegate DestroySessionCompleteDelegate;
        FDelegateHandle DestroySessionCompleteDelegateHandle;
        FOnStartSessionCompleteDelegate StartSessionCompleteDelegate;
        FDelegateHandle StartSessionCompleteDelegateHandle;
    ```
- MultiplayerSessionsSubsystem.cpp
    ``` C++
    UMultiplayerSessionsSubsystem::UMultiplayerSessionsSubsystem() :
        CreateSessionCompleteDelegate(FOnCreateSessionCompleteDelegate::CreateUObject(this, &ThisClass::OnCreateSessionComplete)),
        FindSessionsCompleteDelegate(FOnFindSessionsCompleteDelegate::CreateUObject(this, &ThisClass::OnFindSessionsComplete)),
        JoinSessionCompleteDelegate(FOnJoinSessionCompleteDelegate::CreateUObject(this, &ThisClass::OnJoinSessionComplete)),
        DestroySessionCompleteDelegate(FOnDestroySessionCompleteDelegate::CreateUObject(this, &ThisClass::OnDestroySessionComplete)),
        StartSessionCompleteDelegate(FOnStartSessionCompleteDelegate::CreateUObject(this, &ThisClass::OnStartSessionComplete))

    {
        IOnlineSubsystem* Subsystem = IOnlineSubsystem::Get();
        if (Subsystem)
        {
            SessionInterface = Subsystem->GetSessionInterface();
        }
    }
    ```

- UserWidget을 상속받아 Menu라는 클래스 생성
- MultiplayerSessions.Build.cs
    ``` C#
    PublicDependencyModuleNames.AddRange(
    new string[]
    {
        "Core",
        "OnlineSubsystem",
        "OnlineSubsystemSteam",
        "UMG",
        "Slate",
        "SlateCore"
        // ... add other public dependencies that you statically link with here ...
    }
    );
    ```
- Menu.h
    ``` C++
    #pragma once
    #include "CoreMinimal.h"
    #include "Blueprint/UserWidget.h"
    #include "Menu.generated.h"
    UCLASS()
    class MULTIPLAYERSESSIONS_API UMenu : public UUserWidget
    {
        GENERATED_BODY()
    public:
        UFUNCTION(BlueprintCallable)
        void MenuSetup();
    };
    ```
- Menu.cpp
    ``` C++
    #include "Menu.h"
    void UMenu::MenuSetup()
    {
        AddToViewport();
        SetVisibility(ESlateVisibility::Visible);
        bIsFocusable = true;

        UWorld* World = GetWorld();
        if (World)
        {
            APlayerController* PlayerController = World->GetFirstPlayerController();
            if (PlayerController)
            {
                FInputModeUIOnly InputModeData;
                InputModeData.SetWidgetToFocus(TakeWidget());
                InputModeData.SetLockMouseToViewportBehavior(EMouseLockMode::DoNotLock);
                PlayerController->SetInputMode(InputModeData);
                PlayerController->SetShowMouseCursor(true);
            }
        }
    }
    ```
- MultiplayerSessions Content에 UserWidget 클래스의 WBP_Menu 블루프린트 생성
    - ![image](https://user-images.githubusercontent.com/11372675/208922276-81cb7b48-90b4-4619-9acb-52ce08baf448.png)
    - Class Setting에서 Parent Class를 Menu로 설정
    - test를 위해 Level Blueprint에서 BeginPlay > Create WBP Menu Widget > Menu Setup 연결 후, 플레이하면 보임

## Accessing our Subsystem
1. Menu 클래스가 SubsystemFunction을 호출
2. MultiplayerSessionsSubsystem이 SessionInterfaceFunction을 호출
3. SessionInterface와는 MultiplayerSessionsSubsystem의 Session Interface Delegates가 binding되어있음
4. Callback 함수 호출
5. Subsystem Delegate를 통해 Menu class에 결과를 알려줘야함
- 이렇게 하면 좋은 점은 MultipalyerSessionsSubsystem은 Menu를 몰라도 됨
- 비슷하게, SessionInterface는 MultiplayerSessionsSubsystem을 몰라도 됨
- Menu.h
    ``` C++
    public:
        UFUNCTION(BlueprintCallable)
	    void MenuSetup(int32 NumberOfPublicConnections = 4, FString TypeOfMatch = FString(TEXT("FreeForAll")));
    protected:
        virtual bool Initialize() override;
	    virtual void OnLevelRemovedFromWorld(ULevel* InLevel, UWorld* INWorld) override;
    private:
        UPROPERTY(meta=(BindWidget))
        class UButton* HostButton;
        UPROPERTY(meta = (BindWidget))
        UButton* JoinButton;

        UFUNCTION()
        void HostButtonClicked();
        UFUNCTION()
        void JoinButtonClicked();

	    void MenuTearDown();

        // the subsystem designed to handle all online session functionality
        class UMultiplayerSessionsSubsystem* MultiplayerSessionsSubsystem;

        int32 NumPublicConnections{ 4 };
        FString MatchType{ TEXT("FreeForAll") };
    ```
    - menu=(BindWidget) 옵션을 통해 C++과 BP를 연동할 수 있음
        - 변수 이름이랑 WBP의 위젯 이름이랑 똑같아야함
- Menu.cpp
    ``` C++
    #include "Menu.h"
    #include "Components/Button.h"
    #include "MultiplayerSessionsSubsystem.h"
    void UMenu::MenuSetup(int32 NumberOfPublicConnections, FString TypeOfMatch)
    {
        NumPublicConnections = NumberOfPublicConnections;
        MatchType = TypeOfMatch;

        AddToViewport();
        SetVisibility(ESlateVisibility::Visible);
        bIsFocusable = true;

        UWorld* World = GetWorld();
        if (World)
        {
            APlayerController* PlayerController = World->GetFirstPlayerController();
            if (PlayerController)
            {
                FInputModeUIOnly InputModeData;
                InputModeData.SetWidgetToFocus(TakeWidget());
                InputModeData.SetLockMouseToViewportBehavior(EMouseLockMode::DoNotLock);
                PlayerController->SetInputMode(InputModeData);
                PlayerController->SetShowMouseCursor(true);
            }
        }

        UGameInstance* GameInstance = GetGameInstance();
        if (GameInstance)
        {
            MultiplayerSessionsSubsystem = GameInstance->GetSubsystem<UMultiplayerSessionsSubsystem>();
        }
    }
    bool UMenu::Initialize()
    {
        if (!Super::Initialize())
        {
            return false;
        }
        if (HostButton)
        {
            HostButton->OnClicked.AddDynamic(this, &UMenu::HostButtonClicked);
        }
        if (JoinButton)
        {
            JoinButton->OnClicked.AddDynamic(this, &UMenu::JoinButtonClicked);
        }
        return true;
    }
    void UMenu::OnLevelRemovedFromWorld(ULevel* InLevel, UWorld* INWorld)
    {
        MenuTearDown();
        Super::OnLevelRemovedFromWorld(InLevel, INWorld);
    }
    void UMenu::HostButtonClicked()
    {
        if (MultiplayerSessionsSubsystem)
        {
		    MultiplayerSessionsSubsystem->CreateSession(NumPublicConnections, MatchType);
            UWorld* World = GetWorld();
            if (World)
            {
                World->ServerTravel("/Game/ThirdPerson/Maps/Lobby?listen");
            }
        }
    }
    void UMenu::JoinButtonClicked()
    {
        if (GEngine)
        {
            GEngine->AddOnScreenDebugMessage(
                -1,
                15.f,
                FColor::Yellow,
                FString(TEXT("Join Button Clicked"))
            );
        }
    }
    void UMenu::MenuTearDown()
    {
        RemoveFromParent();
        UWorld* World = GetWorld();
        if (World)
        {
            APlayerController* PlayerController = World->GetFirstPlayerController();
            if (PlayerController)
            {
                FInputModeGameOnly InputModeData;
                PlayerController->SetInputMode(InputModeData);
                PlayerController->SetShowMouseCursor(false);
            }
        }
    }
    ```
- MultiplayerSessionsSubsystem.h
    ``` C++
    private:
    	TSharedPtr<FOnlineSessionSettings> LastSessionSettings;
    ```
- MultiplayerSessionsSubsystem.cpp
    - MenuSystemCharacter에서 Test로 넣어놨던 Settings를 참고하자
    ``` C++
    #include "OnlineSessionSettings.h"
    ...
    void UMultiplayerSessionsSubsystem::CreateSession(int32 NumPublicConnections, FString MatchType)
    {
        if (!SessionInterface.IsValid())
        {
            return;
        }
        auto ExistingSession = SessionInterface->GetNamedSession(NAME_GameSession);
        if (ExistingSession != nullptr)
        {
            SessionInterface->DestroySession(NAME_GameSession);
        }
        // Store the delegate in a FDelegateHandle so we can later remove it from the delegate list
        CreateSessionCompleteDelegateHandle = SessionInterface->AddOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegate);

        LastSessionSettings = MakeShareable(new FOnlineSessionSettings());
        LastSessionSettings->bIsLANMatch = IOnlineSubsystem::Get()->GetSubsystemName() == "NULL" ? true : false;
        LastSessionSettings->NumPublicConnections = NumPublicConnections;
        LastSessionSettings->bAllowJoinInProgress = true;
        LastSessionSettings->bAllowJoinViaPresence = true;
        LastSessionSettings->bShouldAdvertise = true;
        LastSessionSettings->bUsesPresence = true;
        LastSessionSettings->bUseLobbiesIfAvailable = true;	// 최근 언리얼 버전에서는 이거 있어야함
        LastSessionSettings->Set(FName("MatchType"), MatchType, EOnlineDataAdvertisementType::ViaOnlineServiceAndPing);

        const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
        if (!SessionInterface->CreateSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, *LastSessionSettings))
        {
            // Create Session 실패하면 Delegate List Clear
            SessionInterface->ClearOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegateHandle);
        }
    }
    ```

## Callbacks to our subsystem functions
- MultilayerSessionsSubsystem.h
    ```C++
    // Declaring our own custom delegates for the menu class to bind callbacks to
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMultiplayerOnCreateSessionComplete, bool, bWasSuccessful);
    ...
    public:
        // our own custom delegates for the Menu class to bind callbacks to
        FMultiplayerOnCreateSessionComplete MultiplayerOnCreateSessionComplete;
    ``` 
    - MULTICAST : 다수의 클래스가 멤버 함수들을 delegate에 binding할 수 있다는 의미
    - DYNAMIC : delegate가 serialize될 수 있고, BP Graph로 읽거나 쓸 수 있다는 의미 (Event Dispatcher라고 부름)
- Menu.h
    ``` C++
    protected:
        // Callbacks for th custom delegates on the MultiplayerSessionsSubsystem
        UFUNCTION()
        void OnCreateSession(bool bWasSuccessful);
    ```
- Menu.cpp
    ``` C++
    void UMenu::MenuSetup(int32 NumberOfPublicConnections, FString TypeOfMatch)
    {
        ...
        if (MultiplayerSessionsSubsystem)
        {
            MultiplayerSessionsSubsystem->MultiplayerOnCreateSessionComplete.AddDynamic(this, &ThisClass::OnCreateSession);
        }
    }
    void UMenu::OnCreateSession(bool bWasSuccessful)
    {
        if (bWasSuccessful)
        {
            if (GEngine)
            {
                GEngine->AddOnScreenDebugMessage(
                    -1,
                    15.f,
                    FColor::Yellow,
                    FString(TEXT("Session Created Successfully"))
                );
                UWorld* World = GetWorld();
                if (World)
                {
                    World->ServerTravel("/Game/ThirdPerson/Maps/Lobby?listen");
                }
            }
        }
    }
    void UMenu::HostButtonClicked()
    {
        if (MultiplayerSessionsSubsystem)
        {
            MultiplayerSessionsSubsystem->CreateSession(NumPublicConnections, MatchType);
        }
    }
    ```
- MultiplaerSessionsSubsystem.cpp
    - Create Session이 성공했는지, 아닌지를 통해 Broadcast
    ``` C++
    void UMultiplayerSessionsSubsystem::CreateSession(int32 NumPublicConnections, FString MatchType)
    {
        ...
        if (!SessionInterface->CreateSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, *LastSessionSettings))
        {
            SessionInterface->ClearOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegateHandle);

            // Broadcast our own custom delegate
            MultiplayerOnCreateSessionComplete.Broadcast(false);
        }
    }
    ...
    void UMultiplayerSessionsSubsystem::OnCreateSessionComplete(FName SessionName, bool bWasSuccessful)
    {
        if (SessionInterface)
        {
            SessionInterface->ClearOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegateHandle);
        }
        // Broadcast our own custom delegate
        MultiplayerOnCreateSessionComplete.Broadcast(bWasSuccessful);
    }
    ```
- tip) Delegate를 만들었으면 Binaries, Intermediate 지우고 Generate Visual Studio Project를 하는 것 추천
- .uproject LaunchGame하면 STEAM에 접속한거 확인 + Host하면 callback이 되어서 servertravel과 함께 로그도 찍히는 것 확인
