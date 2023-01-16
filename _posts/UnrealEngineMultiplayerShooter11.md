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