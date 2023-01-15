## Teams
- Source > BlasterTypes 폴더에 Team.h 생성
- Team.h
    ``` C++
    #pragma once
    UENUM(BlueprintType)
    enum class ETeam : uint8
    {
        ET_RedTeam UMETA(DisplayName = "RedTeam"),
        ET_BlueTeam UMETA(DisplayName = "BlueTeam"),
        ET_NoTeam UMETA(DisplayName = "NoTeam"),
        ET_MAX UMETA(DisplayName = "DefaultMAX")
    };
    ```
- BlasterPlayerState.h
    ``` C++
    #include "Blaster/BlasterTypes/Team.h"
    private:
	    UPROPERTY(Replicated)
        ETeam Team = ETeam::ET_NoTeam;
    public:
        FORCEINLINE ETeam GetTeam() const { return Team; }
        FORCEINLINE void SetTeam(ETeam TeamToSet) { Team = TeamToSet; }
    ```
- BlasterPlayerState.cpp
    ``` C++
    void ABlasterPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);

        DOREPLIFETIME(ABlasterPlayerState, Defeats);
        DOREPLIFETIME(ABlasterPlayerState, Team);
    }
    ```
- BlasterGameState.h
    ``` C++
    public:
        // Teams
        TArray<ABlasterPlayerState*> RedTeam;
        TArray<ABlasterPlayerState*> BlueTeam;

        UPROPERTY(ReplicatedUsing = OnRep_RedTeamScore)
		float RedTeamScore = 0.f;
	    UFUNCTION()
		void OnRep_RedTeamScore();
	    UPROPERTY(ReplicatedUsing = OnRep_BlueTeamScore)
		float BlueTeamScore = 0.f;
	    UFUNCTION()
		void OnRep_BlueTeamScore();
    ```
- BlasterGameState.cpp
    ``` C++
    void ABlasterGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);

        DOREPLIFETIME(ABlasterGameState, TopScoringPlayers);
        DOREPLIFETIME(ABlasterGameState, RedTeamScore);
        DOREPLIFETIME(ABlasterGameState, BlueTeamScore);
    }
    ```

- BlasterGameMode를 기반으로 TeamGameMode생성
- TeamsGameMode.h
    ``` C++
    public:
        virtual void PostLogin(APlayerController* NewPlayer) override;
	    virtual void Logout(AController* Exiting) override;
    protected:
	    virtual void HandleMatchHasStarted() override;
    ```
- TeamsGameMode.cpp
    ``` C++
    #include "TeamsGameMode.h"
    #include "Blaster/GameState/BlasterGameState.h"
    #include "Blaster/PlayerState/BlasterPlayerState.h"
    #include "Kismet/GameplayStatics.h"
    void ATeamsGameMode::PostLogin(APlayerController* NewPlayer)
    {
        Super::PostLogin(NewPlayer);

        ABlasterGameState* BGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
        if (BGameState)
        {
            ABlasterPlayerState* BPState = NewPlayer->GetPlayerState<ABlasterPlayerState>();
            if (BPState && BPState->GetTeam() == ETeam::ET_NoTeam)
            {
                if (BGameState->BlueTeam.Num() >= BGameState->RedTeam.Num())
                {
                    BGameState->RedTeam.AddUnique(BPState);
                    BPState->SetTeam(ETeam::ET_RedTeam);
                }
                else
                {
                    BGameState->BlueTeam.AddUnique(BPState);
                    BPState->SetTeam(ETeam::ET_BlueTeam);
                }
            }
        }
    }
    void ATeamsGameMode::Logout(AController* Exiting)
    {
        ABlasterGameState* BGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
        ABlasterPlayerState* BPState = Exiting->GetPlayerState<ABlasterPlayerState>();
        if (BGameState && BPState)
        {
            if (BGameState->RedTeam.Contains(BPState))
            {
                BGameState->RedTeam.Remove(BPState);
            }
            if (BGameState->BlueTeam.Contains(BPState))
            {
                BGameState->BlueTeam.Remove(BPState);
            }
        }
        Super::Logout(Exiting);
    }
    void ATeamsGameMode::HandleMatchHasStarted()
    {
        Super::HandleMatchHasStarted();
        ABlasterGameState* BGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
        if (BGameState)
        {
            for (auto PState : BGameState->PlayerArray)
            {
                ABlasterPlayerState* BPState = Cast<ABlasterPlayerState>(PState.Get());
                if (BPState && BPState->GetTeam() == ETeam::ET_NoTeam)
                {
                    if (BGameState->BlueTeam.Num() >= BGameState->RedTeam.Num())
                    {
                        BGameState->RedTeam.AddUnique(BPState);
                        BPState->SetTeam(ETeam::ET_RedTeam);
                    }
                    else
                    {
                        BGameState->BlueTeam.AddUnique(BPState);
                        BPState->SetTeam(ETeam::ET_BlueTeam);
                    }
                }
            }
        }
    }
    ```
- Blueprints > GameModes 폴더에 TeamGameMode 상속받는 BP_TeamsGameMode생성

## Team Color
- BlasterCharacter.h
    ``` C++
    #include "Blaster/BlasterTypes/Team.h"
    public:
    	void SetTeamColor(ETeam Team);
    private:
        // Team Color
        UPROPERTY(EditAnywhere, Category = Elim)
		UMaterialInstance* RedDissolveMatInst;
	    UPROPERTY(EditAnywhere, Category = Elim)
		UMaterialInstance* RedMaterial;
	    UPROPERTY(EditAnywhere, Category = Elim)
		UMaterialInstance* BlueDissolveMatInst;
	    UPROPERTY(EditAnywhere, Category = Elim)
		UMaterialInstance* BlueMaterial;
	    UPROPERTY(EditAnywhere, Category = Elim)
		UMaterialInstance* OriginalMaterial;
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PollInit()
    {
        if (BlasterPlayerState == nullptr)
        {
            BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
            if (BlasterPlayerState)
            {
                ...
                SetTeamColor(BlasterPlayerState->GetTeam());
            }
        }
    }

    void ABlasterCharacter::SetTeamColor(ETeam Team)
    {
        if (GetMesh() == nullptr || OriginalMaterial == nullptr) return;
        switch (Team)
        {
        case ETeam::ET_NoTeam:
            GetMesh()->SetMaterial(0, OriginalMaterial);
            DissolveMaterialInstance = BlueDissolveMatInst;
            break;
        case ETeam::ET_RedTeam:
            GetMesh()->SetMaterial(0, RedMaterial);
            DissolveMaterialInstance = RedDissolveMatInst;
            break;
        case ETeam::ET_BlueTeam:
            GetMesh()->SetMaterial(0, BlueMaterial);
            DissolveMaterialInstance = BlueDissolveMatInst;
            break;
        }
    }
    ```
    - 팀 색깔을 Beginplay에서 정하지 않은 것ㅇ느 PlayerState에 Team 정보가 있는데, BeginPlay에서 초기화되지 않았을 수 있기 때문
- BP_BlasterCharacter에 알맞은 Material추가하기

- BlasterPlayerState.h
    ``` C++
    private:
    	UPROPERTY(ReplicatedUsing = OnRep_Team)
		ETeam Team = ETeam::ET_NoTeam;
    	UFUNCTION()
		void OnRep_Team();
    public:
    	//FORCEINLINE void SetTeam(ETeam TeamToSet) { Team = TeamToSet; }
	    void SetTeam(ETeam TeamToSet);
    ```
- BlasterPlayerState.cpp
    ``` C++
    void ABlasterPlayerState::SetTeam(ETeam TeamToSet)
    {
        Team = TeamToSet;
        ABlasterCharacter* BCharacter = Cast<ABlasterCharacter>(GetPawn());
        if (BCharacter)
        {
            BCharacter->SetTeamColor(Team);
        }
    }
    void ABlasterPlayerState::OnRep_Team()
    {
        ABlasterCharacter* BCharacter = Cast<ABlasterCharacter>(GetPawn());
        if (BCharacter)
        {
            BCharacter->SetTeamColor(Team);
        }
    }
    ```

- BP_TeamsGameMode로 설정하고 시작해보자


## Prevent Teammates Fire
- BlasterCharacter.h
    ``` C++
    private:
    	UPROPERTY()
		class ABlasterGameMode* BlasterGameMode;
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::ReceiveDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatorController, AActor* DamageCauser)
    {
        BlasterGameMode = BlasterGameMode == nullptr ?  GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
        if (bElimned || BlasterGameMode == nullptr) return;
        Damage = BlasterGameMode->CalculateDamage(InstigatorController, Controller, Damage);
        ...
    }
    void ABlasterCharacter::ElimTimerFinished()
    {
        BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
        //ABlasterGameMode* BlasterGameMode = GetWorld()->GetAuthGameMode<ABlasterGameMode>();
        ...
    }
    void ABlasterCharacter::Destroyed()
    {
        ...
        BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
        //ABlasterGameMode* BlasterGameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
        ...
    }
    void ABlasterCharacter::SpawnDefaultWeapon()
    {
        BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
        //ABlasterGameMode* BlasterGameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
        ...
    }
    ```
- BlasterGameMode.h
    ``` C++
    public:
    	virtual float CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage);
    ```
- BlasterGameMode.cpp
    ``` C++
    float ABlasterGameMode::CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage)
    {
        return BaseDamage;
    }
    ```
- TeamsGameMode.h
    ``` C++
    public:
    	virtual float CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage) override;
    ```
- TeamsGameMode.cpp
    ``` C++
    float ATeamsGameMode::CalculateDamage(AController* Attacker, AController* Victim, float BaseDamage)
    {
        ABlasterPlayerState* AttackerPState = Attacker->GetPlayerState<ABlasterPlayerState>();
        ABlasterPlayerState* VictimPState = Victim->GetPlayerState<ABlasterPlayerState>();
        if (AttackerPState == nullptr || VictimPState == nullptr) return BaseDamage;
        if (VictimPState == AttackerPState)
        {
            return BaseDamage;
        }
        if (AttackerPState->GetTeam() == VictimPState->GetTeam())
        {
            return 0.f;
        }
        return BaseDamage;
    }
    ```

## Team Score
- BlasterGameState.h
    ``` C++
    public:
        void RedTeamScores();
    	void BlueTeamScores();
    ```
- BlasterGameState.cpp
    ``` C++
    #include "Blaster/PlayerController/BlasterPlayerController.h"
    void ABlasterGameState::RedTeamScores()
    {
        ++RedTeamScore;
        ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
        if (BPlayer)
        {
            BPlayer->SetHUDRedTeamScore(RedTeamScore);
        }
    }
    void ABlasterGameState::BlueTeamScores()
    {
        ++BlueTeamScore;
        ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
        if (BPlayer)
        {
            BPlayer->SetHUDBlueTeamScore(BlueTeamScore);
        }
    }
    void ABlasterGameState::OnRep_RedTeamScore()
    {
        ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
        if (BPlayer)
        {
            BPlayer->SetHUDRedTeamScore(RedTeamScore);
        }
    }
    void ABlasterGameState::OnRep_BlueTeamScore()
    {
        ABlasterPlayerController* BPlayer = Cast<ABlasterPlayerController>(GetWorld()->GetFirstPlayerController());
        if (BPlayer)
        {
            BPlayer->SetHUDBlueTeamScore(BlueTeamScore);
        }
    }
    ```
- WBP_CharacterOvverlay에 RedTeamScore, BlueTeamScore Text추가
- CharacterOverlay.h
    ``` C++
    public:
    	UPROPERTY(meta = (BindWidget))
		UTextBlock* RedTeamScore;
	    UPROPERTY(meta = (BindWidget))
		UTextBlock* BlueTeamScore;
	    UPROPERTY(meta = (BindWidget))
		UTextBlock* ScoreSpacerText;
    ```

- BlasterPlayerController.h
    ``` C++
    public:
        void HideTeamScores();
        void InitTeamScores();
        void SetHUDRedTeamScore(int32 RedScore);
        void SetHUDBlueTeamScore(int32 BlueScore);
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::HideTeamScores()
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->RedTeamScore &&
            BlasterHUD->CharacterOverlay->BlueTeamScore &&
            BlasterHUD->CharacterOverlay->ScoreSpacerText;
        if (bHUDValid)
        {
            FString Zero("0");
            FString Spacer("|");
            BlasterHUD->CharacterOverlay->RedTeamScore->SetText(FText::FromString(Zero));
            BlasterHUD->CharacterOverlay->BlueTeamScore->SetText(FText::FromString(Zero));
            BlasterHUD->CharacterOverlay->ScoreSpacerText->SetText(FText::FromString(Spacer));
        }
    }
    void ABlasterPlayerController::SetHUDRedTeamScore(int32 RedScore)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->RedTeamScore;
        if (bHUDValid)
        {
            FString ScoreText = FString::Printf(TEXT("%d"), RedScore);
            BlasterHUD->CharacterOverlay->RedTeamScore->SetText(FText::FromString(ScoreText));
        }
    }
    void ABlasterPlayerController::SetHUDBlueTeamScore(int32 BlueScore)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->BlueTeamScore;
        if (bHUDValid)
        {
            FString ScoreText = FString::Printf(TEXT("%d"), BlueScore);
            BlasterHUD->CharacterOverlay->BlueTeamScore->SetText(FText::FromString(ScoreText));
        }
    }
    ```
- BlasterGameMode.h
    ``` C++
    public:
    	bool bTeamsMatch = false;
    ```
- TeamsGameMode.h
    ``` C++
    public:
        ATeamsGameMode();
	    virtual void PlayerEliminated(class ABlasterCharacter* ElimedCharacer, class ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController) override;
    ```
- TeamsGameMode.cpp
    ``` C++
    #include "Blaster/PlayerController/BlasterPlayerController.h"
    ATeamsGameMode::ATeamsGameMode()
    {
        bTeamsMatch = true;
    }
    void ATeamsGameMo
    
    ```
- BlasterPlayerController.h
    ``` C++
    public:
    	void OnMatchStateSet(FName State, bool bTeamsMatch = false);
	    void HandleMatchHasStarted(bool bTeamsMatch = false);
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
                BlasterPlayer->OnMatchStateSet(MatchState, bTeamsMatch);
            }
        }
    }
    ```
- BlasterPlayerController.h
    ``` C++
    protected:
        UPROPERTY(ReplicatedUsing = OnRep_ShowTeamScores)
        bool bShowTeamScores = false;
        UFUNCTION()
		void OnRep_ShowTeamScores();
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(ABlasterPlayerController, MatchState);
        DOREPLIFETIME(ABlasterPlayerController, bShowTeamScores);
    }
    void ABlasterPlayerController::HandleMatchHasStarted(bool bTeamsMatch)
    {
	    if(HasAuthority()) bShowTeamScores = bTeamsMatch;
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        if (BlasterHUD)
        {
            ...
		    if (!HasAuthority()) return;
            if (bTeamsMatch)
            {
                InitTeamScores();
            }
            else
            {
                HideTeamScores();
            }
        }
    }
    void ABlasterPlayerController::OnRep_ShowTeamScores()
    {
        if (bShowTeamScores)
        {
            InitTeamScores();
        }
        else
        {
            HideTeamScores();
        }
    }
    ```

## Teams Cooldown Announcement
- Source >Blaster > BlasterTypes에 Announcement.h 생성
- Announcement.h
    ``` C++
    #pragma once
    namespace Announcement
    {
        const FString NewMatchStartsIn(TEXT("New match starts in :"));
        const FString ThereIsNoWinner(TEXT("There is No Winner."));
        const FString YouAreTheWinner(TEXT("You are the winner!"));
        const FString PlayersTiedForTheWin(TEXT("Players tied for the win:"));
	    const FString TeamsTiedForTheWin(TEXT("Teams tied for the win:"));
        const FString RedTeam(TEXT("Red Team"));
        const FString BlueTeam(TEXT("Blue Team"));
        const FString RedTeamWins(TEXT("Red Team Wins!"));
        const FString BlueTeamWins(TEXT("Blue Team Wins!"));
    }
    ```
- BlasterPlayerController.h
    ``` C++
    protected:
    	FString GetInfoText(const TArray<class ABlasterPlayerState*>& Players);
	    FString GetTeamsInfoText(class ABlasterGameState* BlasterGameState);
    ```
- BlasterPlayerController.cpp
    ``` C++
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
                FString AnnouncementText = Announcement::NewMatchStartsIn;
                BlasterHUD->Announcement->AnnouncementText->SetText(FText::FromString(AnnouncementText));

                ABlasterGameState* BlasterGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
                ABlasterPlayerState* BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
                if (BlasterGameState && BlasterPlayerState)
                {
                    TArray<ABlasterPlayerState*> TopPlayers = BlasterGameState->TopScoringPlayers;
                    FString InfoTextString = bShowTeamScores ? GetTeamsInfoText(BlasterGameState) : GetInfoText(TopPlayers);
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
    FString ABlasterPlayerController::GetInfoText(const TArray<class ABlasterPlayerState*>& Players)
    {
        ABlasterPlayerState* BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
        if (BlasterPlayerState == nullptr) return FString();
        FString InfoTextString;
        if (Players.Num() == 0)
        {
            InfoTextString = Announcement::ThereIsNoWinner;
        }
        else if (Players.Num() == 1 && Players[0] == BlasterPlayerState)
        {
            InfoTextString = Announcement::YouAreTheWinner;
        }
        else if (Players.Num() == 1)
        {
            InfoTextString = FString::Printf(TEXT("Winner : \n%s"), *(Players[0]->GetPlayerName()));
        }
        else if (Players.Num() > 1)
        {
            InfoTextString = Announcement::PlayersTiedForTheWin;
            InfoTextString.Append(FString("\n"));
            for (auto TiedPlayer : Players)
            {
                InfoTextString.Append(FString::Printf(TEXT("%s\n"), *(TiedPlayer->GetPlayerName())));
            }
        }
        return InfoTextString;
    }
    FString ABlasterPlayerController::GetTeamsInfoText(ABlasterGameState* BlasterGameState)
    {
        if (BlasterGameState == nullptr) return FString();
        FString InfoTextString;
        const int32 RedTeamScore = BlasterGameState->RedTeamScore;
        const int32 BlueTeamScore = BlasterGameState->BlueTeamScore;
        
        if (RedTeamScore == 0 && BlueTeamScore == 0)
        {
            InfoTextString = Announcement::ThereIsNoWinner;
        }
        else if (RedTeamScore == BlueTeamScore)
        {
            InfoTextString = FString::Printf(TEXT("%s\n"), *Announcement::ThereIsNoWinner);
            InfoTextString.Append(Announcement::RedTeam);
            InfoTextString.Append(TEXT("\n"));
            InfoTextString.Append(Announcement::BlueTeam);
            InfoTextString.Append(TEXT("\n"));
        }
        else if (RedTeamScore > BlueTeamScore)
        {
            InfoTextString = Announcement::RedTeamWins;
            InfoTextString.Append(TEXT("\n"));
            InfoTextString.Append(FString::Printf(TEXT("%s : %d"), *Announcement::RedTeam, RedTeamScore));
            InfoTextString.Append(FString::Printf(TEXT("%s : %d"), *Announcement::BlueTeam, BlueTeamScore));
        }
        else if (RedTeamScore < BlueTeamScore)
        {
            InfoTextString = Announcement::BlueTeamWins;
            InfoTextString.Append(TEXT("\n"));
            InfoTextString.Append(FString::Printf(TEXT("%s : %d"), *Announcement::BlueTeam, BlueTeamScore));
            InfoTextString.Append(FString::Printf(TEXT("%s : %d"), *Announcement::RedTeam, RedTeamScore));
        }
        return InfoTextString;
    }
    ```
