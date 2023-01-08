## GameTimer
- WBP_CharacterOverlay에 MatchCountdownText 추가
- BlasterPlayerController.h
    ``` C++
    public:
    	void SetHUDMatchCountdown(float CountdownTime);
	    virtual void Tick(float DeltaTime) override;
    protected:
    	void SetHUDTime();
    private:
    	float MatchTime = 120.f;
	    uint32 CountdownInt = 0;
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        SetHUDTime();
    }
    void ABlasterPlayerController::SetHUDMatchCountdown(float CountdownTime)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->MatchCountdownText;
        if (bHUDValid)
        {
            if (CountdownTime < 0.f)
            {
                BlasterHUD->CharacterOverlay->MatchCountdownText->SetText(FText());
                return;
            }
            int32 Minutes = FMath::FloorToInt(CountdownTime / 60.f);
            int32 Seconds = CountdownTime - Minutes * 60;
            FString CountdownText = FString::Printf(TEXT("%02d:%02d"),Minutes,Seconds);
            BlasterHUD->CharacterOverlay->MatchCountdownText->SetText(FText::FromString(CountdownText));
        }
    }
    void ABlasterPlayerController::SetHUDTime()
    {
        uint32 SecondsLeft = FMath::CeilToInt(MatchTime - GetWorld()->GetTimeSeconds());
        if (CountdownInt != SecondsLeft)
        {
            SetHUDMatchCountdown(MatchTime - GetWorld()->GetTimeSeconds());
        }
        CountdownInt = SecondsLeft;
    }
    ```
- 주의! worldTime은 각 client마다 다름

## Syncing Client and ServerTime
- GetServerTime()처럼 RPC를 사용한다고 해도 RPC를 보내고 받는 시간 때문에 정확하지는 않음
- RoundTripTime = Current Client Time - Client Request time
- CurrentServerTime = ServerTime of Receipt + 1/2 * RountTripTime
- Client-Server delta = Current Server Time - Current Client Time
- GetServerTime() = Current Client Time - Client-Server Delta

- BlasterPlayerController.h
    ``` C++
    public:
        // synced with server world clock
        virtual float GetServerTime();
        // Sync with server clock as soon as possible
        virtual void ReceivedPlayer() override;
    protected:
        // Sync time between client and server

        // Requests the current server time, passing in the client's time when the request was sent
        UFUNCTION(Server, Reliable)
        void ServerRequestServerTime(float TimeOfClientRequest);
        // Reports the current server time to the client in response to ServerRequestServerTime
        UFUNCTION(Client, Reliable)
        void ClientReportServerTime(float TimeOfClientRequest, float TimeServerReceivedClientRequest);
        // difference between client and server time
        float ClientServerDelta = 0.f;	
        UPROPERTY(EditAnywhere, Category = Time)
        float TimeSyncFrequency = 5.f;
	    float TimeSyncRunningTime = 0.f;
	    void CheckTimeSync(float DeltaTime);
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        SetHUDTime();
        CheckTimeSync(DeltaTime);
    }
    void ABlasterPlayerController::CheckTimeSync(float DeltaTime)
    {
        TimeSyncRunningTime += DeltaTime;
        if (IsLocalController() && TimeSyncRunningTime > TimeSyncFrequency)
        {
            ServerRequestServerTime(GetWorld()->GetTimeSeconds());
            TimeSyncRunningTime = 0;
        }
    }
    void ABlasterPlayerController::ServerRequestServerTime_Implementation(float TimeOfClientRequest)
    {
        float ServerTimeOfReceipt = GetWorld()->GetTimeSeconds();
        ClientReportServerTime(TimeOfClientRequest, ServerTimeOfReceipt);
    }
    void ABlasterPlayerController::ClientReportServerTime_Implementation(float TimeOfClientRequest, float TimeServerReceivedClientRequest)
    {
        float RoundTripTime = GetWorld()->GetTimeSeconds() - TimeOfClientRequest;
        float CurrentServerTime = TimeServerReceivedClientRequest + (0.5f * RoundTripTime);
        ClientServerDelta = CurrentServerTime - GetWorld()->GetTimeSeconds();
    }
    float ABlasterPlayerController::GetServerTime()
    {
        if (HasAuthority()) return GetWorld()->GetTimeSeconds();
        return GetWorld()->GetTimeSeconds() + ClientServerDelta;
    }
    ```
    - GetServerTime을 해줄때마다 delta를 계산하는 것이 아니라 Tick에서 threshold넘기면 다시 체크해주는 방식
- tip) ReceivedPlayer가 서버 시간을 가장 빨리 알 수 있는 시점임

- 적용(BlasterPlayerController.cpp)
    ``` C++
    void ABlasterPlayerController::SetHUDTime()
    {
        uint32 SecondsLeft = FMath::CeilToInt(MatchTime - GetServerTime());
        if (CountdownInt != SecondsLeft)
        {
            SetHUDMatchCountdown(MatchTime - GetServerTime());
        }
        CountdownInt = SecondsLeft;
    }
    ```

## Match State
- GameMode vs GameModeBase
    - AGameModeBase
        - Default Pawn Class
        - Spawns Player's Pawn
        - Restart Players
        - Restart Game
    - AGameMode
        - Match Game
        - Handling Match State
        - Custom Match State
- GameMode는 'MatchState'라는 namespace를 가짐
    1. Entering Map
    2. Waiting To Start
    3. InProgress
    4. Waiting Post Match
    5. Leaving Map
    6. Aborted
    - tip) Custom Match State를 만들 수 있는데, 2~3 사이, 3~4 사이에 넣을 수 있음
    - tip) 1~2 : WarmupTime, 3 : MatchTime, 4~6 : Cooldown Time
    - tip) GameMode에서 bDelayedStart=true로 설정해놓으면 실제로 게임이 시작하는 것이 우리에게 맡겨짐
- BlasterGameMode.h
    ``` C++
    public:
        ABlasterGameMode();
	    virtual void Tick(float DeltaTime) override;

        UPROPERTY(EditDefaultsOnly)
        float WarmupTime = 10.f;
        float LevelStartingTime = 0.f;
    protected:
        virtual void BeginPlay() override;
    private:
        float CountdownTime = 0.f;
    ```
- BlasterGameMode.cpp
    ``` C++
    ABlasterGameMode::ABlasterGameMode()
    {
        bDelayedStart = true;
    }
    void ABlasterGameMode::BeginPlay()
    {
        Super::BeginPlay();
        LevelStartingTime = GetWorld()->GetTimeSeconds();
    }
    void ABlasterGameMode::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        if (MatchState == MatchState::WaitingToStart)
        {
            CountdownTime = WarmupTime - GetWorld()->GetTimeSeconds() + LevelStartingTime;
            if (CountdownTime <= 0.0f)
            {
                StartMatch();
            }
        }
    }
    ```

## On Match State Set
- BlasterGameMode.h
    ``` C++
    protected:
    	virtual void OnMatchStateSet() override;
    ```
- BlasterGameMode.cpp
    ``` C++
    void ABlasterGameMode::OnMatchStateSet()
    {
        Super::OnMatchStateSet();
        for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; It++)
        {
            ABlasterPlayerController* BlasterPlayer = Cast<ABlasterPlayerController>(*It);
            if (BlasterPlayer)
            {
                BlasterPlayer->OnMatchStateSet(MatchState);
            }
        }
    }
    ```
- BlasterPlayerController.h
    ``` C++
    public:
	    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    	void OnMatchStateSet(FName State);
    private:
        UPROPERTY(ReplicatedUsing = OnRep_MatchState)
        FName MatchState;

        UFUNCTION()
        void OnRep_MatchState();
    ```
- BlasterPlayerController.cpp
    ``` C++
    #include "Net/UnrealNetwork.h"
    #include "Blaster/GameMode/BlasterGameMode.h"

    void ABlasterPlayerController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(ABlasterPlayerController, MatchState);
    }
    void ABlasterPlayerController::OnMatchStateSet(FName State)
    {
        MatchState = State;

        if (MatchState == MatchState::InProgress)
        {
            BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
            if (BlasterHUD)
            {
                BlasterHUD->AddCharacterOverlay();
            }
        }
    }
    void ABlasterPlayerController::OnRep_MatchState()
    {
        if (MatchState == MatchState::InProgress)
        {
            BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
            if (BlasterHUD)
            {
                BlasterHUD->AddCharacterOverlay();
            }
        }
    }
    ```
- BlasterHUD.h
    ``` C++
    public:
        void AddCharacterOverlay();
        UPROPERTY()
        class UCharacterOverlay* CharacterOverlay;
    ```
- BlasterHUD.cpp
    ``` C++
    void ABlasterHUD::BeginPlay()
    {
        Super::BeginPlay();
        //UE_LOG(LogTemp, Warning, TEXT("BeginPlay"));
        //AddCharacterOverlay();
    }
    ```
    - 이제 HUD는 게임이 진짜 시작할 때 보여줄거임

- 근데 이러면 HUD가 업데이트가 안 되어있음 -> HUD를 만들기 전에 값이 업데이트 되기 때문


- BlasterPlayerController.h
    ``` C++
    protected:
    	void PollInit();
    private:
        UPROPERTY()
        class UCharacterOverlay* CharacterOverlay;
        bool bInitializeCharacterOverlay = false;
        float HUDHealth;
        float HUDMaxHealth;
        float HUDScore;
        int32 HUDDefeats;
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::Tick(float DeltaTime)
    {
        ...
        PollInit();
    }
    void ABlasterPlayerController::PollInit()
    {
        if (CharacterOverlay == nullptr)
        {
            if (BlasterHUD && BlasterHUD->CharacterOverlay)
            {
                CharacterOverlay = BlasterHUD->CharacterOverlay;
                if (CharacterOverlay)
                {
                    SetHUDHealth(HUDHealth, HUDMaxHealth);
                    SetHUDScore(HUDScore);
                    SetHUDDefeats(HUDDefeats);
                }
            }
        }
    }
    void ABlasterPlayerController::SetHUDHealth(float Health, float MaxHealth)
    {
        ...
        if (bHUDValid)
        {
            ...
        }
        else
        {
            bInitializeCharacterOverlay = true;
            HUDHealth = Health;
            HUDMaxHealth = MaxHealth;
        }
    }
    void ABlasterPlayerController::SetHUDScore(float Score)
    {
        ...
        if (bHUDValid)
        {
            ...
        }
        else
        {
            bInitializeCharacterOverlay = true;
            HUDScore = Score;
        }
    }
    void ABlasterPlayerController::SetHUDDefeats(int32 Defeats)
    {
        ...
        if (bHUDValid)
        {
            ...
        }
        else
        {
            bInitializeCharacterOverlay = true;
            HUDDefeats = Defeats;
        }
    }
    ```

## WarmupTimer
- Source > Blaster > HUD에 UserWidget상속받은 Announcement 클래스 생성
- Announcement.h
    ``` C++
    public:
        UPROPERTY(meta = (BindWidget))
        class UTextBlock* WarmupTime;
        UPROPERTY(meta = (BindWidget))
        UTextBlock* AnnouncementText;
        UPROPERTY(meta = (BindWidget))
        UTextBlock* InfoText;
    ```
- Blueprints > HUD에 Announcement 상속받은 WBP_Announcement 생성
    - Textblock으로 AnnouncementText, WarmupTime, InfoText 추가

- BlasterPlayerController.h
    ``` C++
    public:
        void HandleMatchHasStarted();
    ```
- BlasterPlayerController.cpp
    ``` C++
    #include "Blaster/HUD/Announcement.h"
    void ABlasterPlayerController::BeginPlay()
    {
        Super::BeginPlay();
        BlasterHUD = Cast<ABlasterHUD>(GetHUD());
        if (BlasterHUD)
        {
            BlasterHUD->AddAnnouncement();
        }
    }
    void ABlasterPlayerController::OnMatchStateSet(FName State)
    {
        MatchState = State;
        if (MatchState == MatchState::InProgress)
        {
            HandleMatchHasStarted();
        }
    }
    void ABlasterPlayerController::OnRep_MatchState()
    {
        if (MatchState == MatchState::InProgress)
        {
            HandleMatchHasStarted();
        }
    }
    void ABlasterPlayerController::HandleMatchHasStarted()
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        if (BlasterHUD)
        {
            BlasterHUD->AddCharacterOverlay();
            if (BlasterHUD->Announcement)
            {
                BlasterHUD->Announcement->SetVisibility(ESlateVisibility::Hidden);
            }
        }
    }
    ```
- BlasterHUD.h
    ``` C++
    public:
        UPROPERTY(EditAnywhere, Category = "Announcements");
        TSubclassOf<UUserWidget> AnnouncemenetClass;
        UPROPERTY()
        class UAnnouncement* Announcement;
        void AddAnnouncement();
    ```
- BlasterHUD.cpp
    ``` C++
    #include "Announcement.h"
    void ABlasterHUD::AddAnnouncement()
    {
        APlayerController* PlayerController = GetOwningPlayerController();
        if (PlayerController && AnnouncemenetClass)
        {
            Announcement = CreateWidget<UAnnouncement>(PlayerController, AnnouncemenetClass);
            Announcement->AddToViewport();
        }
    }
    ```
- BP_BlasterHud의 Announcement Class 속성에 WBP_Announcement 추가하기

## Updating Warmpup Time
- BlasterPlayerController.h
    ``` C++
    public:
    	void SetHUDAnnouncementCountdown(float CountdownTime);
    protected:
        UFUNCTION(Server, Reliable)
        void ServerCheckMatchState();
        UFUNCTION(Client, Reliable)
        void ClientJoinMidGame(FName StateOfMatch, float Warmup, float Match, float StartingTime);
    private:
	    float LevelStartingTime = 0.f;
        float MatchTime = 0.f;
        float WarmupTime = 0.f;
    ```
- BlasterPlayerController.cpp
    ``` C++
    #include "Kismet/GameplayStatics.h"

    void ABlasterPlayerController::BeginPlay()
    {
        Super::BeginPlay();
        BlasterHUD = Cast<ABlasterHUD>(GetHUD());
        ServerCheckMatchState();
    }
    void ABlasterPlayerController::SetHUDAnnouncementCountdown(float CountdownTime)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->Announcement &&
            BlasterHUD->Announcement->WarmupTime;
        if (bHUDValid)
        {
            if (CountdownTime < 0.f)
            {
                BlasterHUD->Announcement->WarmupTime->SetText(FText());
			    return;
            }
            int32 Minutes = FMath::FloorToInt(CountdownTime / 60.f);
            int32 Seconds = CountdownTime - Minutes * 60;
            FString CountdownText = FString::Printf(TEXT("%02d:%02d"), Minutes, Seconds);
            BlasterHUD->Announcement->WarmupTime->SetText(FText::FromString(CountdownText));
        }
    }
    void ABlasterPlayerController::SetHUDTime()
    {
        float TimeLeft = 0.f;
        if (MatchState == MatchState::WaitingToStart) TimeLeft = WarmupTime - GetServerTime() + LevelStartingTime;
        else if (MatchState == MatchState::InProgress) TimeLeft = MatchTime - GetServerTime() + LevelStartingTime;
        uint32 SecondsLeft = FMath::CeilToInt(TimeLeft);
        if (CountdownInt != SecondsLeft)
        {
            if (MatchState == MatchState::WaitingToStart)
            {
                SetHUDAnnouncementCountdown(TimeLeft);
            }
            if (MatchState == MatchState::InProgress)
            {
                SetHUDMatchCountdown(TimeLeft);
            }
        }
        CountdownInt = SecondsLeft;
    }
    void ABlasterPlayerController::ServerCheckMatchState_Implementation()
    {
        ABlasterGameMode* GameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
        if (GameMode)
        {
            WarmupTime = GameMode->WarmupTime;
            MatchTime = GameMode->MatchTime;
		    LevelStartingTime = GameMode->LevelStartingTime;
		    MatchState = GameMode->GetMatchState();
		    ClientJoinMidGame(MatchState, WarmupTime, MatchTime, LevelStartingTime);
            if (BlasterHUD && MatchState == MatchState::WaitingToStart)
            {
                BlasterHUD->AddAnnouncement();
            }
        }
    }
    void ABlasterPlayerController::ClientJoinMidGame_Implementation(FName StateOfMatch, float Warmup, float Match, float StartingTime)
    {
        WarmupTime = Warmup;
        MatchTime = Match;
        LevelStartingTime = StartingTime;
        MatchState = StateOfMatch;
        OnMatchStateSet(MatchState);
		if (BlasterHUD && MatchState == MatchState::WaitingToStart)
		{
			BlasterHUD->AddAnnouncement();
		}
    }

    ```
- BlasterGameMode.h
    ``` C++
    public:
    	UPROPERTY(EditDefaultsOnly)
		float MatchTime = 120.f;
	    UPROPERTY(EditDefaultsOnly)
		float WarmupTime = 10.f;
    ```

## Custom Match State
- BlasterGameMode.h
    ``` C++
    namespace MatchState
    {
        extern BLASTER_API const FName Cooldown;	// Match duration has been reached. Display Winner and begin cooldown timer
    }
    ...
    public:
    	UPROPERTY(EditDefaultsOnly)
		float CooldownTime = 10.f;
    public:
        FORCEINLINE float GetCountdownTime() const { return CountdownTime; }
    ```
- BlasterGameMode.cpp
    ``` C++
    namespace MatchState
    {
        const FName Cooldown = FName("Cooldown");
    }
    void ABlasterGameMode::Tick(float DeltaTime)
    {
        ...
        else if (MatchState == MatchState::InProgress)
        {
            CountdownTime = WarmupTime + MatchTime - GetWorld()->GetTimeSeconds() + LevelStartingTime;
            if (CountdownTime <= 0.f)
            {
                SetMatchState(MatchState::Cooldown);
            }
        }
    }

    void ABlasterPlayerController::HandleCooldown()
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        if (BlasterHUD)
        {
            BlasterHUD->CharacterOverlay->RemoveFromParent();
            if (BlasterHUD->Announcement)
            {
                BlasterHUD->Announcement->SetVisibility(ESlateVisibility::Visible);
            }
        }
    }
    ```
- BlasterPlayerController.h
    ``` C++
    public:
    	void HandleCooldown();
    protected:
        UFUNCTION(Client, Reliable)
    	void ClientJoinMidGame(FName StateOfMatch, float Warmup, float Match, float Cooldown, float StartingTime);
    private:
        UPROPERTY()
        class ABlasterGameMode* BlasterGameMode;
    	float CooldownTime = 0.f;
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::ServerCheckMatchState_Implementation()
    {
        ABlasterGameMode* GameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
        if (GameMode)
        {
            ...
            CooldownTime = GameMode->CooldownTime;
		    ClientJoinMidGame(MatchState, WarmupTime, MatchTime, CooldownTime, LevelStartingTime);
            ...
        }
    }
    void ABlasterPlayerController::ClientJoinMidGame_Implementation(FName StateOfMatch, float Warmup, float Match, float Cooldown, float StartingTime)
    {
        CooldownTime = Cooldown;
        ...
    }
    void ABlasterPlayerController::OnMatchStateSet(FName State)
    {
        MatchState = State;
        if (MatchState == MatchState::InProgress)
        {
            HandleMatchHasStarted();
        }
        else if (MatchState == MatchState::Cooldown)
        {
            HandleCooldown();
        }
    }
    void ABlasterPlayerController::OnRep_MatchState()
    {
        if (MatchState == MatchState::InProgress)
        {
            HandleMatchHasStarted();
        }
        else if (MatchState == MatchState::Cooldown)
        {
            HandleCooldown();
        }
    }
    void ABlasterPlayerController::SetHUDTime()
    {
        ...
        else if (MatchState == MatchState::Cooldown) TimeLeft = CooldownTime + WarmupTime + MatchTime - GetServerTime() + LevelStartingTime;
        uint32 SecondsLeft = FMath::CeilToInt(TimeLeft);
        if (HasAuthority())
        {
            BlasterGameMode = BlasterGameMode == nullptr ? Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this)) : BlasterGameMode;
            if (BlasterGameMode)
            {
                SecondsLeft = FMath::CeilToInt(BlasterGameMode->GetCountdownTime() + LevelStartingTime);
            }
        }
        if (CountdownInt != SecondsLeft)
        {
            if (MatchState == MatchState::WaitingToStart || MatchState == MatchState::Cooldown)
            {
                SetHUDAnnouncementCountdown(TimeLeft);
            }
            ...
        }
        CountdownInt = SecondsLeft;
    }
    void ABlasterPlayerController::HandleCooldown()
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        if (BlasterHUD)
        {
            BlasterHUD->CharacterOverlay->RemoveFromParent();
            bool bHUDValid = BlasterHUD->Announcement &&
                BlasterHUD->Announcement->AnnouncementText &&
                BlasterHUD->Announcement->InfoText;
            if (bHUDValid)
            {
                BlasterHUD->Announcement->SetVisibility(ESlateVisibility::Visible);
                FString AnnouncementText("New Match Starts In :");
                BlasterHUD->Announcement->AnnouncementText->SetText(FText::FromString(AnnouncementText));
			    BlasterHUD->Announcement->InfoText->SetText(FText());
            }
        }
    }
    ```
    - 서버라면 GameMode에서 바로 받아오는게 더 정확함

## RestartGame
- BlasterGameMode.cpp
    ``` C++
    void ABlasterGameMode::Tick(float DeltaTime)
    {
        ...
        else if (MatchState == MatchState::Cooldown)
        {
            CountdownTime = CooldownTime + WarmupTime + MatchTime - GetWorld()->GetTimeSeconds() + LevelStartingTime;
            if (CountdownTime <= 0.f)
            {
                RestartGame();
            }
        }
    }
    ```
    - 이렇게만 하면 서버만 이동하는데(RestartGame이 ServerTravel), 패키징을 하면 제대로 나옴

- Cooldown상태일 때 화면 돌리는것만 되도록 하고싶음
- BlasterCharacter.h
    ``` C++
    public:
	    UPROPERTY(Replicated)   // 서버에서 토글할거라서 클라쪽에도 가려면 replicate해야함
    	bool bDisableGameplay = false;
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME_CONDITION(ABlasterCharacter, OverlappingWeapon, COND_OwnerOnly);
        DOREPLIFETIME(ABlasterCharacter, Health);
        DOREPLIFETIME(ABlasterCharacter, bDisableGameplay);
    }
    void ABlasterCharacter::MoveForward(float Value)
    {
        if (bDisableGameplay) return;
        ...
    }
    ... // 위에처럼 분기 만들어주기
    void ABlasterCharacter::MulticastElim_Implementation()
    {
        ...
        bDisableGameplay = true;
        ...
    }
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::HandleCooldown()
    {
        ...
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(GetPawn());
        if (BlasterCharacter)
        {
            BlasterCharacter->bDisableGameplay = true;
        }
    }
    ```
- FireButton같은 경우 누르고있으면 계속 나가기에 manually하게 꺼줄 필요 있음
- CombatComponent.h
    ``` C++
    public:
    	void FireButtonPressed(bool bPressed);
    ```
- BlasterCharacter.h   
    ``` C++
    public:
    	FORCEINLINE UCombatComponent* GetCombat() const { return Combat; };
    ```
- BlasterPlayerController.cpp
    ``` C++
    #include "Blaster/BlasterComponents/CombatComponent.h"

    void ABlasterPlayerController::HandleCooldown()
    {
        ...
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(GetPawn());
        if (BlasterCharacter && BlasterCharacter->GetCombat())
        {
            BlasterCharacter->bDisableGameplay = true;
            BlasterCharacter->GetCombat()->FireButtonPressed(false);
        }
    }
    ```

- 몸이 회전하는것도 막고싶음
- BlasterCharacter.h
    ``` C++
    protected:
    	void RotateInPlace(float DeltaTime);
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        RotateInPlace(DeltaTime);
        HideCameraIfCharacterClose();
        PollInit();
    }

    void ABlasterCharacter::RotateInPlace(float DeltaTime)
    {
        if (bDisableGameplay)
        {
            bUseControllerRotationYaw = false;
            TurningInPlace = ETurningInPlace::ETIP_NotTurning;
            return;
        }
        if (GetLocalRole() > ENetRole::ROLE_SimulatedProxy && IsLocallyControlled())
        {
            AimOffset(DeltaTime);
        }
        else
        {
            TimeSinceLastMovementReplication += DeltaTime;
            if (TimeSinceLastMovementReplication > 0.25f)
            {
                OnRep_ReplicatedMovement();
            }
            CalculateAO_Pitch();
        }
    }
    ```

- 손이 aim따라 가는것도..막고싶음

- BlasterCharacter.h
    ``` C++
    public:
	    FORCEINLINE bool GetDisableGameplay() const { return bDisableGameplay; };
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bUseFABRIK = BlasterCharacter->GetCombatState() != ECombatState::ECS_Reloading;
        bUseAimOffsets = BlasterCharacter->GetCombatState() != ECombatState::ECS_Reloading && !BlasterCharacter->GetDisableGameplay();
        bTransformRightHand = BlasterCharacter->GetCombatState() != ECombatState::ECS_Reloading && !BlasterCharacter->GetDisableGameplay();
    }
    ```

- 플레이어가 방 나갔을 때 총이 그대로 남아있는 현상
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::Destroyed()
    {
        ...
        if (Combat && Combat->EquippedWeapon)
        {
            Combat->EquippedWeapon->Destroy();
        }
    }
    ```
    - 옛날에는 Elim일때 drop되게 했었음

## Blaster GameState
- Source > Blaster > GameState 폴더에 GameState 상속받는 BlasterGameState 클래스 생성
- BlasterGameState.h
    ``` C++
    public:
        virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
        void UpdateTopScore(class ABlasterPlayerState* ScoringPlayer);

        UPROPERTY(Replicated)
        TArray<class ABlasterPlayerState*> TopScoringPlayers;
    private:
        float TopScore = 0.f;
    ```
- BlasterGameState.cpp
    ``` C++
    #include "BlasterGameState.h"
    #include "Net/UnrealNetwork.h"
    #include "Blaster/PlayerState/BlasterPlayerState.h"

    void ABlasterGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);

        DOREPLIFETIME(ABlasterGameState, TopScoringPlayers);
    }

    void ABlasterGameState::UpdateTopScore(ABlasterPlayerState* ScoringPlayer)
    {
        if (TopScoringPlayers.Num() == 0)
        {
            TopScoringPlayers.Add(ScoringPlayer);
            TopScore = ScoringPlayer->GetScore();
        }
        else if (ScoringPlayer->GetScore() == TopScore)
        {
            TopScoringPlayers.AddUnique(ScoringPlayer);
        }
        else if (ScoringPlayer->GetScore() > TopScore)
        {
            TopScoringPlayers.Empty();
            TopScoringPlayers.AddUnique(ScoringPlayer);
            TopScore = ScoringPlayer->GetScore();
        }
    }
    ```
- BlasterGameMode.cpp
    ``` C++
    #include "Blaster/GameState/BlasterGameState.h"
    ...
    void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
        if (AttackerController == nullptr || AttackerController->PlayerState == nullptr) return;
        if (VictimController == nullptr || VictimController->PlayerState == nullptr) return;
        ABlasterPlayerState* AttackerPlayerState = AttackerController ? Cast<ABlasterPlayerState>(AttackerController->PlayerState) : nullptr;
        ABlasterPlayerState* VictimPlayerState = VictimController ? Cast<ABlasterPlayerState>(VictimController->PlayerState) : nullptr;

        ABlasterGameState* BlasterGameState = GetGameState<ABlasterGameState>();
        if (AttackerPlayerState && AttackerPlayerState != VictimPlayerState && BlasterGameState)
        {
            AttackerPlayerState->AddToScore(1.f);
            BlasterGameState->UpdateTopScore(AttackerPlayerState);
        }
        ...
    }
    ```
- BlasterPlayerController.cpp
    ``` C++
    #include "Blaster/GameState/BlasterGameState.h"
    #include "Blaster/PlayerState/BlasterPlayerState.h"
    void ABlasterPlayerController::HandleCooldown()
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        if (BlasterHUD)
        {
            BlasterHUD->CharacterOverlay->RemoveFromParent();
            bool bHUDValid = BlasterHUD->Announcement &&
                BlasterHUD->Announcement->AnnouncementText &&
                BlasterHUD->Announcement->InfoText;
            if (bHUDValid)
            {
                BlasterHUD->Announcement->SetVisibility(ESlateVisibility::Visible);
                FString AnnouncementText("New Match Starts In :");
                BlasterHUD->Announcement->AnnouncementText->SetText(FText::FromString(AnnouncementText));

                ABlasterGameState* BlasterGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
                ABlasterPlayerState* BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
                if (BlasterGameState && BlasterPlayerState)
                {
                    TArray<ABlasterPlayerState*> TopPlayers = BlasterGameState->TopScoringPlayers;
                    FString InfoTextString;
                    if (TopPlayers.Num() == 0)
                    {
                        InfoTextString = FString("There is no winner.");
                    }
                    else if (TopPlayers.Num() == 1 && TopPlayers[0] == BlasterPlayerState)
                    {
                        InfoTextString = FString("You are the winner.");
                    }
                    else if (TopPlayers.Num() == 1)
                    {
					    InfoTextString = FString::Printf(TEXT("Winner : \n%s"),*(TopPlayers[0]->GetPlayerName()));
                    }
                    else if (TopPlayers.Num() > 1)
                    {
                        InfoTextString = FString("Players tied for the win : \n");
                        for (auto TiedPlayer : TopPlayers)
                        {
                            InfoTextString.Append(FString::Printf(TEXT("%s\n"), *(TiedPlayer->GetPlayerName())));
                        }
                    }
                    BlasterHUD->Announcement->InfoText->SetText(FText::FromString(InfoTextString));
                }
            }
        }
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(GetPawn());
        if (BlasterCharacter && BlasterCharacter->GetCombat())
        {
            BlasterCharacter->bDisableGameplay = true;
            BlasterCharacter->GetCombat()->FireButtonPressed(false);
        }
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::MulticastElim_Implementation()
    {
        ...
        bDisableGameplay = true;
        if (Combat)
        {
            Combat->FireButtonPressed(false);
        }
        ...
    }

    void ABlasterCharacter::Destroyed()
    {
        Super::Destroyed();
        if (ElimBotComponent)
        {
            ElimBotComponent->DestroyComponent();
        }

        ABlasterGameMode* BlasterGameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
        bool bMatchNotInProgress = BlasterGameMode && BlasterGameMode->GetMatchState() != MatchState::InProgress;
        if (Combat && Combat->EquippedWeapon && bMatchNotInProgress)
        {
            Combat->EquippedWeapon->Destroy();
        }
    }
    ```
- Blueprints > GameState 폴더에 BlasterGameState 기반으로 BP_BlasterGameState 생성
