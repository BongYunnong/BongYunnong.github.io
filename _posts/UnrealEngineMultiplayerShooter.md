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

