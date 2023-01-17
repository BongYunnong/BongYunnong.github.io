## Return to Main Menu
- Source > Blaster > HUD 폴더에 UserWidget을 상속받은 ReturnToMainMenu 클래스 생성
- Blueprints > HUD에 ReturnToMainMenu 상속받은 WBP_ReturnToMainMenu 생성

- ReturnToMainMenu.h
    ``` C++
    public:
        void MenuSetup();
        void MenuTearDown();
    protected:
        virtual bool Initialize() override;
        UFUNCTION()
        void OnDestroySession(bool bWasSuccessful);
    private:
        UPROPERTY(meta = (BindWidget))
        class UButton* ReturnButton;

        UFUNCTION()
        void ReturnButtonClicked();
        UPROPERTY()
        class UMultiplayerSessionSubsystem* MultiplayerSessionsSubsystem;
        UPROPERTY()
        class APlayerController* PlayerController;
    ```
- ReturnToMainMenu.cpp
    ``` C++
    #include "ReturnToMainMenu.h"
    #include "GameFramework/PlayerController.h"
    #include "GameFramework/GameModeBase.h"
    #include "Components/Button.h"
    #include "MultiplayerSessionsSubsystem.h"
    void UReturnToMainMenu::MenuSetup()
    {
        AddToViewport();
        SetVisibility(ESlateVisibility::Visible);
        bIsFocusable = true;
        UWorld* World = GetWorld();
        if (World)
        {
            PlayerController = PlayerController == nullptr ? World->GetFirstPlayerController() : PlayerController;
            if (PlayerController)
            {
                FInputModeGameAndUI InputModeData;
                InputModeData.SetWidgetToFocus(TakeWidget());
                PlayerController->SetInputMode(InputModeData);
                PlayerController->SetShowMouseCursor(true);
            }
        }
        if (ReturnButton && !ReturnButton->OnClicked.IsBound())
        {
            ReturnButton->OnClicked.AddDynamic(this, &UReturnToMainMenu::ReturnButtonClicked);
        }
        UGameInstance* GameInstance = GetGameInstance();
        if (GameInstance)
        {
            MultiplayerSessionsSubsystem = GameInstance->GetSubsystem<UMultiplayerSessionsSubsystem>();
            if (MultiplayerSessionsSubsystem)
            {
                MultiplayerSessionsSubsystem->MultiplayerOnDestroySessionComplete.AddDynamic(this, &UReturnToMainMenu::OnDestroySession);
            }
        }
    }
    bool UReturnToMainMenu::Initialize()
    {
        if (!Super::Initialize())
        {
            return false;
        }
        return true;
    }
    void UReturnToMainMenu::OnDestroySession(bool bWasSuccessful)
    {
        if (!bWasSuccessful)
        {
            ReturnButton->SetIsEnabled(true);
            return;
        }
        UWorld* World = GetWorld();
        if (World)
        {
            AGameModeBase* GameMode = World->GetAuthGameMode<AGameModeBase>();
            if (GameMode)
            {
                GameMode->ReturnToMainMenuHost();
            }
            else
            {
                PlayerController = PlayerController == nullptr ? World->GetFirstPlayerController() : PlayerController;
                if (PlayerController)
                {
                    PlayerController->ClientReturnToMainMenuWithTextReason(FText());
                }
            }
        }
    }
    void UReturnToMainMenu::MenuTearDown()
    {
        RemoveFromParent();
        UWorld* World = GetWorld();
        if (World)
        {
            PlayerController = PlayerController == nullptr ? World->GetFirstPlayerController() : PlayerController;
            if (PlayerController)
            {
                FInputModeGameOnly InputModeData;
                PlayerController->SetInputMode(InputModeData);
                PlayerController->SetShowMouseCursor(false);
            }
        }
        if (ReturnButton && ReturnButton->OnClicked.IsBound())
        {
            ReturnButton->OnClicked.RemoveDynamic(this, &UReturnToMainMenu::ReturnButtonClicked);
        }
        if (MultiplayerSessionsSubsystem && MultiplayerSessionsSubsystem->MultiplayerOnDestroySessionComplete.IsBound())
        {
            MultiplayerSessionsSubsystem->MultiplayerOnDestroySessionComplete.RemoveDynamic(this, &UReturnToMainMenu::OnDestroySession);
        }
    }
    void UReturnToMainMenu::ReturnButtonClicked()
    {
        ReturnButton->SetIsEnabled(false);
        if (MultiplayerSessionsSubsystem)
        {
            MultiplayerSessionsSubsystem->DestroySession();
        }
    }
    ```
- BlasterBuild.cs
    ``` C++
    PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "Niagara", "MultiplayerSessions", "OnlineSubsystem", "OnlineSubsystemSteam" });
    ```

- Quit 액션을 Escape, Q키에 매핑


- BlasterPlayerController.h
    ``` C++
    protected:
        virtual void SetupInputComponent() override;
	    void ShowReturnToMainMenu();
    private:
        // Return to Main Menu
        UPROPERTY(EditAnywhere, Category = HUD)
        TSubclassOf<class UUserWidget> ReturnToMainMenuWidget;

        UPROPERTY()
        class UReturnToMainMenu* ReturnToMainMenu;

        bool bReturnToMainMenuOpen = false;
    ```
- BlasterPlayerController.cpp
    ``` C++
    #include "Blaster/HUD/ReturnToMainMenu.h"
    void ABlasterPlayerController::SetupInputComponent()
    {
        Super::SetupInputComponent();
        if (InputComponent == nullptr) return;
        InputComponent->BindAction("Quit", IE_Pressed, this, &ABlasterPlayerController::ShowReturnToMainMenu);
    }
    void ABlasterPlayerController::ShowReturnToMainMenu()
    {
        if (ReturnToMainMenuWidget == nullptr) return;
        if (ReturnToMainMenu == nullptr)
        {
            ReturnToMainMenu = CreateWidget<UReturnToMainMenu>(this, ReturnToMainMenuWidget);
        }
        if (ReturnToMainMenu)
        {
            bReturnToMainMenuOpen = !bReturnToMainMenuOpen;
            if (bReturnToMainMenuOpen)
            {
                ReturnToMainMenu->MenuSetup();
            }
            else
            {
                ReturnToMainMenu->MenuTearDown();
            }
        }
    }
    ```

- BP_BlasterPlayerController에서 ReturnToMainMenuWIdget설정


- WBP_Menu에 ExitButton추가
    - OnClicked되면 QuitGame하도록


## Player Bookkeeping
- BlasterCharacter.h
    ``` C++
    DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLeftGame);
    public:
        void Elim(bool bPlayerLeftGame);
        //void Elim();
        UFUNCTION(NetMulticast,Reliable)
        void MulticastElim(bool bPlayerLeftGame);
        //void MulticastElim();
        UFUNCTION(Server, Reliable)
        void ServerLeaveGame();
	    FOnLeftGame OnLeftGame;
    private:
        bool bLeftGame = false;
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::Elim(bool bPlayerLeftGame)
    {
        ...
        MulticastElim(bPlayerLeftGame);
        /*
        GetWorldTimerManager().SetTimer(
            ElimTimer,
            this,
            &ABlasterCharacter::ElimTimerFinished,
            ElimDelay
        );
        */
    }
    void ABlasterCharacter::MulticastElim_Implementation(bool bPlayerLeftGame)
    {
        bLeftGame = bPlayerLeftGame;
        ...
        GetWorldTimerManager().SetTimer(
            ElimTimer,
            this,
            &ABlasterCharacter::ElimTimerFinished,
            ElimDelay
        );
    }
    void ABlasterCharacter::ElimTimerFinished()
    {
        BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
        if (BlasterGameMode && !bLeftGame)
        {
            BlasterGameMode->RequestRespawn(this, Controller);
        }
        if (bLeftGame && IsLocallyControlled())
        {
            OnLeftGame.Broadcast();
        }
    }
    void ABlasterCharacter::ServerLeaveGame_Implementation()
    {
        BlasterGameMode = BlasterGameMode == nullptr ? GetWorld()->GetAuthGameMode<ABlasterGameMode>() : BlasterGameMode;
        BlasterPlayerState = BlasterPlayerState == nullptr ? GetPlayerState<ABlasterPlayerState>() : BlasterPlayerState;
        if (BlasterGameMode && BlasterPlayerState)
        {
            BlasterGameMode->PlayerLeftGame(BlasterPlayerState);
        }
    }
    ```
- BlasterGameMode.h
    ``` C++
    public:
	    void PlayerLeftGame(class ABlasterPlayerState* PlayerLeaving);
    ```
- BlasterGameMode.cpp
    ``` C++
    void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
        ...
        if (ElimedCharacer)
        {
            ElimedCharacer->Elim(false);
        }
    }
    void ABlasterGameMode::PlayerLeftGame(ABlasterPlayerState* PlayerLeaving)
    {
        if (PlayerLeaving == nullptr) return;
        ABlasterGameState* BlasterGameState = GetGameState<ABlasterGameState>();
        if (BlasterGameState && BlasterGameState->TopScoringPlayers.Contains(PlayerLeaving))
        {
            BlasterGameState->TopScoringPlayers.Remove(PlayerLeaving);
        }
        ABlasterCharacter* CharacterLeaving = Cast<ABlasterCharacter>(PlayerLeaving->GetPawn());
        if (CharacterLeaving)
        {
            CharacterLeaving->Elim(true);
        }
    }
    ```
- ReturnToMainMenu.h
    ``` C++
    protected:
	    UFUNCTION()
    	void OnPlayerLeftGame();
    ```
- ReturnToMainMenu.cpp
    ``` C++
    #include "Blaster/Character/BlasterCharacter.h"
    void UReturnToMainMenu::ReturnButtonClicked()
    {
        ReturnButton->SetIsEnabled(false);
        UWorld* World = GetWorld();
        if (World)
        {
            APlayerController* FirstPlayerControrller = World->GetFirstPlayerController();
            if (FirstPlayerControrller)
            {
                ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FirstPlayerControrller->GetPawn());
                if (BlasterCharacter)
                {
                    BlasterCharacter->ServerLeaveGame();
				    BlasterCharacter->OnLeftGame.AddDynamic(this, &UReturnToMainMenu::OnPlayerLeftGame);
                }
                else
                {
                    ReturnButton->SetIsEnabled(true);
                }
            }
        }
    }
    void UReturnToMainMenu::OnPlayerLeftGame()
    {
        if (MultiplayerSessionsSubsystem)
        {
            MultiplayerSessionsSubsystem->DestroySession();
        }
    }
    ```

## Gaining The Lead
- 언리얼의 애셋을 FBX로 export할 수 있다.
- BlasterCharacer.h 
    ``` C++
    public:
	    UFUNCTION(NetMulticast, Reliable)
        void MulticastGainedTheLead();
	    UFUNCTION(NetMulticast, Reliable)
        void MulticastLostTheLead();
    private:
        UPROPERTY(EditAnywhere)
        class UNiagaraSystem* CrownSystem;
        UPROPERTY()
        class UNiagaraComponent* CrownComponent;
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "NiagaraComponent.h"
    #include "NiagaraFunctionLibrary.h"
    #include "Blaster/GameState/BlasterGameState.h"
    void ABlasterCharacter::MulticastGainedTheLead_Implementation()
    {
        if (CrownSystem == nullptr) return;
        if (CrownComponent == nullptr)
        {
            CrownComponent = UNiagaraFunctionLibrary::SpawnSystemAttached(
                CrownSystem,
                GetCapsuleComponent(),
                FName(),
                GetMesh()->GetComponentLocation() + FVector(0.f, 0.f, 55.f),
                GetActorRotation(),
                EAttachLocation::KeepWorldPosition,
                false
            );
        }
        if (CrownComponent)
        {
            CrownComponent->Activate();
        }
    }

    void ABlasterCharacter::MulticastLostTheLead_Implementation()
    {
        if (CrownComponent)
        {
            CrownComponent->DestroyComponent();
        }
    }
    // 새로 스폰된 플레이어에게 crown을 붙이기 위함
    void ABlasterCharacter::PollInit()
    {
        if (BlasterPlayerState == nullptr)
        {
            BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
            if (BlasterPlayerState)
            {
                ...
                ABlasterGameState* BlasterGameState = Cast<ABlasterGameState>(UGameplayStatics::GetGameState(this));
                if (BlasterGameState && BlasterGameState->TopScoringPlayers.Contains(BlasterPlayerState))
                {
                    MulticastGainedTheLead();
                }
            }
        }
    }
    void ABlasterCharacter::MulticastElim_Implementation(bool bPlayerLeftGame)
    {
        ...
        if (CrownComponent)
        {
            CrownComponent->DestroyComponent();
        }
        ...
    }

    ```
- BlasterGameMode.cpp
    ``` C++
    void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
        ...
        if (AttackerPlayerState && AttackerPlayerState != VictimPlayerState && BlasterGameState)
        {
            TArray <ABlasterPlayerState*> PlayersCurrentlyInTheLead;
            for (auto LeadPlayer : BlasterGameState->TopScoringPlayers)
            {
                PlayersCurrentlyInTheLead.Add(LeadPlayer);
            }
            AttackerPlayerState->AddToScore(1.f);
            BlasterGameState->UpdateTopScore(AttackerPlayerState);
            if (BlasterGameState->TopScoringPlayers.Contains(AttackerPlayerState))
            {
                ABlasterCharacter* Leader = Cast<ABlasterCharacter>(AttackerPlayerState->GetPawn());
                if (Leader)
                {
                    Leader->MulticastGainedTheLead();
                }
            }

            for (int32 i = 0; i < PlayersCurrentlyInTheLead.Num(); i++)
            {
                if (!BlasterGameState->TopScoringPlayers.Contains(PlayersCurrentlyInTheLead[i]))
                {
                    ABlasterCharacter* Loser = Cast<ABlasterCharacter>(PlayersCurrentlyInTheLead[i]->GetPawn());
                    if (Loser)
                    {
                        Loser->MulticastLostTheLead();
                    }
                }
            }
        }
        ...
    }
    ```
- BP_BlasterCharacter에 적절한 Crown 이펙트 추가하기
## Elim Announcement
- Source > Blaster > HUD 폴더에 UserWidget을 상속받는 ElimAnnouncement 클래스 생성
- ElimAnnouncement.h
    ``` C++
    public:
	    void SetElimAnnouncementText(FString AttackerName, FString VictimName);
        UPROPERTY(meta = (BindWidget))
            class UHorizontalBox* AnnouncementBox;
        UPROPERTY(meta = (BindWidget))
            class UTextBlock* AnnouncementText;
    ```
- ElimAnnouncement.cpp
    ``` C++
    #include "ElimAnnouncement.h"
    #include "Components/TextBlock.h"
    void UElimAnnouncement::SetElimAnnouncementText(FString AttackerName, FString VictimName)
    {
        FString ElimAnnouncementText = FString::Printf(TEXT("%s elimmed %s!"), *AttackerName, *VictimName);
        if (AnnouncementText)
        {
            AnnouncementText->SetText(FText::FromString(ElimAnnouncementText));
        }
    }
    ```
- Blueprints > HUD 폴더에 ElimAnnouncement를 상속받는 WBP_ElimAnnouncement 생성
    - HorizontalBox 추가(AnnouncementBox)
    - Text추가(AnnouncementText)

- BlasterHUD.h
    ``` C++
    public:
	    void AddElimAnnouncement(FString Attacker, FString Victim);
    private:
	    UPROPERTY()
		class APlayerController* OwningPlayer;
    	UPROPERTY(EditAnywhere)
		TSubclassOf<class UElimAnnouncement> ElimAnnouncementClass;
    ```
- BlasterHUD.cpp
    ``` C++
    #include "ElimAnnouncement.h"
    void ABlasterHUD::AddElimAnnouncement(FString Attacker, FString Victim)
    {
        OwningPlayer = OwningPlayer == nullptr ? GetOwningPlayerController() : OwningPlayer;
        if (OwningPlayer && ElimAnnouncementClass)
        {
            UElimAnnouncement* ElimAnnouncementWidget = CreateWidget<UElimAnnouncement>(OwningPlayer, ElimAnnouncementClass);
            if (ElimAnnouncementWidget)
            {
                ElimAnnouncementWidget->SetElimAnnouncementText(Attacker, Victim);
                ElimAnnouncementWidget->AddToViewport();
            }
        }
    }
    ```
- BP_BlasterHUD에 WBP_ElimAnnouncement 설정

- BlasterPlayerController.h
    ``` C++
    public:
    	void BroadcastElim(APlayerState* Attacker, APlayerState* Victim);
    protected:
        UFUNCTION(Client, Reliable)
        void ClientElimAnnouncement(APlayerState* Attacker, APlayerState* Victim);
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::BroadcastElim(APlayerState* Attacker, APlayerState* Victim)
    {
        ClientElimAnnouncement(Attacker, Victim);
    }
    void ABlasterPlayerController::ClientElimAnnouncement_Implementation(APlayerState* Attacker, APlayerState* Victim)
    {
        APlayerState* Self = GetPlayerState<APlayerState>();
        if (Attacker && Victim && Self)
        {
            BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
            if (BlasterHUD)
            {
                if (Attacker == Self && Victim != Self)
                {
                    BlasterHUD->AddElimAnnouncement("You", Victim->GetPlayerName());
                    return;
                }
                if (Victim == Self && Attacker != Self)
                {
                    BlasterHUD->AddElimAnnouncement(Attacker->GetPlayerName(), "You");
                    return;
                }
                if (Attacker == Victim && Attacker == Self)
                {
                    BlasterHUD->AddElimAnnouncement("You", "Yourself");
                    return;
                }
                if (Attacker == Victim && Attacker != Self)
                {
                    BlasterHUD->AddElimAnnouncement(Attacker->GetPlayerName(), "themselves");
                    return;
                }
                BlasterHUD->AddElimAnnouncement(Attacker->GetPlayerName(), Victim->GetPlayerName());
            }
        }
    }
    ```
- BlasterGameMode.cpp
    ``` C++
    void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
        ...
        for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; It++)
        {
            ABlasterPlayerController* BlasterPlayer = Cast<ABlasterPlayerController>(*It);
            if (BlasterPlayer && AttackerPlayerState && VictimPlayerState)
            {
                BlasterPlayer->BroadcastElim(AttackerPlayerState, VictimPlayerState);
            }
        }
    }
    ```

## Dynamic Elim Announcement
- BlasterHUD.h
    ``` C++
    private:
	    UPROPERTY(EditAnywhere)
		float ElimAnnouncementTime = 1.5f;
        UFUNCTION()
        void ElimAnnouncementTimerFinished(UElimAnnouncement* MsgToRemove);
    ```
- BlasterHUD.cpp
    ``` C++
    void ABlasterHUD::AddElimAnnouncement(FString Attacker, FString Victim)
    {
        ...
        if (OwningPlayer && ElimAnnouncementClass)
        {
            ...
            if (ElimAnnouncementWidget)
            {
                ElimAnnouncementWidget->SetElimAnnouncementText(Attacker, Victim);
                ElimAnnouncementWidget->AddToViewport();

                FTimerHandle ElimMsgTimer;
                FTimerDelegate ElimMsgDelegate;
                ElimMsgDelegate.BindUFunction(this, FName("ElimAnnouncementTimerFinished"), ElimAnnouncementWidget);
                GetWorldTimerManager().SetTimer(
                    ElimMsgTimer,
                    ElimMsgDelegate,
                    ElimAnnouncementTime,
                    false
                );
            }
        }
    }
    void ABlasterHUD::ElimAnnouncementTimerFinished(UElimAnnouncement* MsgToRemove)
    {
        if (MsgToRemove)
        {
            MsgToRemove->RemoveFromParent();
        }
    }
    ```

- 여러개를 동시에 보여주자
- BlasterHUD.h
    ``` C++
	UPROPERTY()
	TArray<UElimAnnouncement*> ElimMessages;
    ```
- BlasterHUD.cpp
    ``` C++
    #include "Components/HorizontalBox.h"
    #include "Components/CanvasPanelSlot.h"
    #include "Blueprint/WidgetLayoutLibrary.h"
    void ABlasterHUD::AddElimAnnouncement(FString Attacker, FString Victim)
    {
        ...
        if (OwningPlayer && ElimAnnouncementClass)
        {
            ...
            if (ElimAnnouncementWidget)
            {
                ElimAnnouncementWidget->SetElimAnnouncementText(Attacker, Victim);
                ElimAnnouncementWidget->AddToViewport();
                for (UElimAnnouncement* Msg : ElimMessages)
                {
                    if (Msg && Msg->AnnouncementBox)
                    {
                        UCanvasPanelSlot* CanvasSlot = UWidgetLayoutLibrary::SlotAsCanvasSlot(Msg->AnnouncementBox);
                        if (CanvasSlot)
                        {
                            FVector2D Position = CanvasSlot->GetPosition();
                            FVector2D NewPosition(CanvasSlot->GetPosition().X, Position.Y - CanvasSlot->GetSize().Y);
                            CanvasSlot->SetPosition(NewPosition);
                        }
                    }
                }
                ElimMessages.Add(ElimAnnouncementWidget);
                ...
            }
        }
    }
    ```

## Head Shot
- Weapon.h
    ``` C++
    protected:
    	UPROPERTY(EditAnywhere)
		float Damage = 20.f;
	    UPROPERTY(EditAnywhere)
		float HeadShotDamage = 40.f;
	public:
        FORCEINLINE float GetDamage() const { return Damage; }
        FORCEINLINE float GetHeadShotDamage() const { return HeadShotDamage; }
    ```
- HitScanWeapon.cpp
    ``` C++
    void AHitScanWeapon::Fire(const FVector& HitTarget)
    {
        ....
        if (MuzzleFlashSocket)
        {
            ...
            if (BlasterCharacter && HasAuthority() && InstigatorController)
            {
                const float DamageToCause = FireHit.BoneName.ToString() == FString("head") ? HeadShotDamage : Damage;

                UGameplayStatics::ApplyDamage(
                    BlasterCharacter,
                    DamageToCause,
                    InstigatorController,
                    this,
                    UDamageType::StaticClass()
                );
            }
            ...
        }
    }
    void AHitScanWeapon::WeaponTraceHit(const FVector& TraceStart, const FVector& HitTarget, FHitResult& OutHit)
    {
        ...
        if (World)
        {
            ...
            FVector BeamEnd = End;
            if (OutHit.bBlockingHit)
            {
                BeamEnd = OutHit.ImpactPoint;
            }
            else
            {
                OutHit.ImpactPoint = End;
            }
            ...
        }
    }
    ```
- Shotgun.cpp
    ``` C++
    void AShotgun::FireShotgun(const TArray<FVector_NetQuantize>& HitTargets)
    {
        ...
        if (MuzzleFlashSocket)
        {
            ...
            TMap<ABlasterCharacter*, uint32> HitMap;
            TMap<ABlasterCharacter*, uint32> HeadshotHitMap;
            for (FVector_NetQuantize HitTarget : HitTargets)
            {
                FHitResult FireHit;
                WeaponTraceHit(Start, HitTarget, FireHit);
                ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FireHit.GetActor());
                if (BlasterCharacter)
                {
                    const bool bHeadShot = FireHit.BoneName.ToString() == FString("head");
                    if (bHeadShot)
                    {
                        if (HeadshotHitMap.Contains(BlasterCharacter)) HeadshotHitMap[BlasterCharacter]++;
                        else HeadshotHitMap.Emplace(BlasterCharacter, 1);
                    }
                    else
                    {
                        if (HitMap.Contains(BlasterCharacter)) HitMap[BlasterCharacter]++;
                        else HitMap.Emplace(BlasterCharacter, 1);
                    }
                }
                ...
            }
            TArray< ABlasterCharacter*> HitCharacters;
            TMap<ABlasterCharacter*, float> DamageMap;
            for (auto HitPair : HitMap)
            {
                if (HitPair.Key)
                {
                    DamageMap.Emplace(HitPair.Key, HitPair.Value * Damage);
                    HitCharacters.AddUnique(HitPair.Key);
                }
            }
            for (auto HeadshotHitPair : HeadshotHitMap)
            {
                if (HeadshotHitPair.Key)
                {
                    if (DamageMap.Contains(HeadshotHitPair.Key)) DamageMap[HeadshotHitPair.Key]+=HeadshotHitPair.Value*HeadShotDamage;
                    else DamageMap.Emplace(HeadshotHitPair.Key, HeadshotHitPair.Value * HeadShotDamage);
                    HitCharacters.AddUnique(HeadshotHitPair.Key);
                }
            }
            for (auto DamagePair : DamageMap)
            {
                if (DamagePair.Key && InstigatorController)
                {
                    UGameplayStatics::ApplyDamage(
                        DamagePair.Key,
                        DamagePair.Value,
                        InstigatorController,
                        this,
                        UDamageType::StaticClass()
                    );
                }
            }
        }
    }

    ```

## Projectile HeadShot
- Projectile.h
    ``` C++
    protected:
    	UPROPERTY(EditAnywhere)
		float Damage = 20.0f;
	    UPROPERTY(EditAnywhere)
		float HeadShotDamage = 20.0f;
    ```
- ProjectileBullet.cpp
    ``` C++
    void AProjectileBullet::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
        if (OwnerCharacter)
        {
            AController* OwnerController = OwnerCharacter->Controller;
            if (OwnerController)
            {
                const float DamageToCause = Hit.BoneName.ToString() == FString("head") ? HeadShotDamage : Damage;

                UGameplayStatics::ApplyDamage(OtherActor, Damage, OwnerController, this, UDamageType::StaticClass());
            }
        }
        Super::OnHit(HitComp, OtherActor, OtherComp, NormalImpulse, Hit);
    }
    ```