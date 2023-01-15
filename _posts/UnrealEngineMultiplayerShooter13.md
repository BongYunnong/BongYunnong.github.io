# Capture the flag
## Hold Flag
- Blueprints > Character > Animation 폴더에 blendspace 1d로 torchidle, torchwalk 연결한 Flag 생성
    - Horizontal은 Speed로 0~100
- CombatComponent.cpp
    ``` C++
    private:
    	bool bHoldingTheFlag = false;
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	bool IsHoldingFlag() const;
    ```
- BlasterCharacter.cpp
    ``` C++
    bool ABlasterCharacter::IsHoldingFlag() const
    {
        if (Combat == nullptr) return false;
        return Combat->bHoldingTheFlag;
    }
    ```
- BlasterAnimInstance.h
    ``` C++
    private:
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		bool bHoldingTheFlag;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bHoldingTheFlag = BlasterCharacter->IsHoldingFlag();
        ...
    }
    ```

- BlasterAnimBP에 Holding The Flag라는 StateMachine생성
    - 캐싱
    - HoldingTheFlag 스테이트에 speed와 연결한 Flag 연결
    - Blend Pose By Bool로 holding the flag 기준으로 분기

- Weapon을 상속받아 Flag 클래스 생성
- Flag.h
    ``` C++
    public:
        AFlag();
    private:
        UPROPERTY(VisibleAnywhere)
        UStaticMeshComponent* FlagMesh;
    ```
- Flag.cpp
    ``` C++
    #include "Flag.h"
    #include "Components/StaticMeshComponent.h"
    #include "Components/SphereComponent.h"
    #include "Components/WidgetComponent.h"
    AFlag::AFlag()
    {
        FlagMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("FlagMesh"));
        SetRootComponent(FlagMesh);
        GetAreaSphere()->SetupAttachment(FlagMesh);
        GetPickupWidget()->SetupAttachment(FlagMesh);
        FlagMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
        FlagMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }
    ```
- WeaponTypes.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        ...
        EWT_Flag UMETA(DisplayName = "Flag"),
        ...
    };
    ```
- Weapon.h
    ``` C++
    public:
    	FORCEINLINE UWidgetComponent* GetPickupWidget() const { return PickupWidget; }
    ```

- Blueprints > Weapon 폴더에 Flag를 상속받은 BP_Flag생성

## Pickup Flag
- CombatComponent.h
    ``` C++
    protected:
    	void AttachFlagToLeftHand(AWeapon* Flag);
    private:
        UPROPERTY(ReplicatedUsing = OnRep_HoldingTheFlag)
        bool bHoldingTheFlag = false;
        UFUNCTION()
        void OnRep_HoldingTheFlag();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        ...
        DOREPLIFETIME(UCombatComponent, bHoldingTheFlag);
    }
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        if (Character == nullptr || WeaponToEquip == nullptr) return;
        if (CombatState != ECombatState::ECS_Unoccupied) return;

        if (WeaponToEquip->GetWeaponType() == EWeaponType::EWT_Flag)
        {
		    Character->Crouch();
		    bHoldingTheFlag = true;
		    AttachFlagToLeftHand(WeaponToEquip);
		    WeaponToEquip->SetWeaponState(EWeaponState::EWS_Equipped);
		    WeaponToEquip->SetOwner(Character);
        }
        else
        {
            if (EquippedWeapon != nullptr && SecondaryWeapon == nullptr)
            {
                EquipSecondaryWeapon(WeaponToEquip);
            }
            else
            {
                EquipPrimaryWeapon(WeaponToEquip);
            }
            Character->GetCharacterMovement()->bOrientRotationToMovement = false;
            Character->bUseControllerRotationYaw = true;
        }
    }
    void UCombatComponent::AttachFlagToLeftHand(AWeapon* Flag)
    {
        if (Character == nullptr || Character->GetMesh() == nullptr || Flag == nullptr) return;
        const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(FName("FlagSocket"));
        if (HandSocket)
        {
            HandSocket->AttachActor(Flag, Character->GetMesh());
        }
    }
    void UCombatComponent::OnRep_HoldingTheFlag()
    {
        if (bHoldingTheFlag && Character && Character->IsLocallyControlled()) 
        {
            Character->Crouch();
        }
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::RotateInPlace(float DeltaTime)
    {
        if (Combat && Combat->bHoldingTheFlag)
        {
            bUseControllerRotationYaw = false;
            GetCharacterMovement()->bOrientRotationToMovement = true;
            TurningInPlace = ETurningInPlace::ETIP_NotTurning;
            return;
        }
        ...
    }
    ```
## Input 막기
- BlasterCharacter.cpp  
    ``` C++
    void ABlasterCharacter::EquipButtonPressed()
    {
        if (bDisableGameplay) return;
        if (Combat)
        {
            if (Combat->bHoldingTheFlag) return;
            ...
        }
    }
    void ABlasterCharacter::CrouchButtonPressed()
    {
        if (Combat == nullptr ||Combat->bHoldingTheFlag) return;
        ...
    }
    void ABlasterCharacter::AimButtonReleased()
    {
        if (Combat == nullptr || Combat->bHoldingTheFlag) return;
        ...
    }
    void ABlasterCharacter::FireButtonPressed()
    {
        if (Combat == nullptr || Combat->bHoldingTheFlag) return;
        ...
    }

    void ABlasterCharacter::FireButtonReleased()
    {
        if (Combat == nullptr || Combat->bHoldingTheFlag) return;
        ...
    }
    void ABlasterCharacter::ReloadButtonPressed()
    {
        if (Combat == nullptr || Combat->bHoldingTheFlag) return;
        ...
    }
    void ABlasterCharacter::GrenadeButtonPressed()
    {
        if (Combat == nullptr || Combat->bHoldingTheFlag) return;
        ...
    }
    void ABlasterCharacter::Jump()
    {
        if (Combat == nullptr || Combat->bHoldingTheFlag) return;
        ...
    }
    ```

## Drop Flag
- Flag는 SkeletalMesh안 쓰니까 Dropped 함수 재정의 필요
- Flag.h
    ``` C++
    public:
    	virtual void Dropped() override;
    protected:
        virtual void OnEquipped() override;
        virtual void OnDropped() override;
    ```
- Flag.cpp
    ``` C++
    void AFlag::Dropped()
    {
        SetWeaponState(EWeaponState::EWS_Dropped);
        FDetachmentTransformRules DetachRules(EDetachmentRule::KeepWorld, true);
        FlagMesh->DetachFromComponent(DetachRules);
        SetOwner(nullptr);
        BlasterOwnerCharacter = nullptr;
        BlasterOwnerController = nullptr;
    }
    void AFlag::OnEquipped()
    {
        ShowPickupWidget(false);
        GetAreaSphere()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        FlagMesh->SetSimulatePhysics(false);
        FlagMesh->SetEnableGravity(false);
        FlagMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        EnableCustomDepth(false);
    }
    void AFlag::OnDropped()
    {
        if (HasAuthority())
        {
            GetAreaSphere()->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
        }
        FlagMesh->SetSimulatePhysics(true);
        FlagMesh->SetEnableGravity(true);
        FlagMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
        FlagMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
        FlagMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
        FlagMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
        FlagMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
        FlagMesh->MarkRenderStateDirty();
        EnableCustomDepth(true);
    }
    ```
- CombatComponent.h
    ``` C++
    private:
        UPROPERTY()
		AWeapon* TheFlag;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        if (Character == nullptr || WeaponToEquip == nullptr) return;
        if (CombatState != ECombatState::ECS_Unoccupied) return;

        if (WeaponToEquip->GetWeaponType() == EWeaponType::EWT_Flag)
        {
            Character->Crouch();
            bHoldingTheFlag = true;
            WeaponToEquip->SetWeaponState(EWeaponState::EWS_Equipped);
            AttachFlagToLeftHand(WeaponToEquip);
            WeaponToEquip->SetOwner(Character);
            TheFlag = WeaponToEquip;
        }
        ...
    }

    void ABlasterCharacter::DropOrDestroyWeapons()
    {
        if (Combat)
        {
            ...
            if (Combat->TheFlag)
            {
                Combat->TheFlag->Dropped();
            }
        }
    }
    ```

## Team Flag
- Weapon.h
    ``` C++
    #include "Blaster/BlasterTypes/Team.h"
    private:
    	UPROPERTY(EditAnywhere)
		ETeam Team;
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::OnSphereOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            if (WeaponType == EWeaponType::EWT_Flag && BlasterCharacter->GetTeam() == Team) return;
            if (BlasterCharacter->IsHoldingFlag()) return;
            BlasterCharacter->SetOverlappingWeapon(this);
        }
    }
    void AWeapon::OnSphereEndOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
    {
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            if (WeaponType == EWeaponType::EWT_Flag && BlasterCharacter->GetTeam() == Team) return;
            if (BlasterCharacter->IsHoldingFlag()) return;
            BlasterCharacter->SetOverlappingWeapon(nullptr);
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	ETeam GetTeam();
    ```
- BlasterCharacter.cpp
    ``` C++
    ETeam ABlasterCharacter::GetTeam()
    {
        BlasterPlayerState = BlasterPlayerState==nullptr ? GetPlayerState<ABlasterPlayerState>() : BlasterPlayerState;
        if (BlasterPlayerState == nullptr) return ETeam::ET_NoTeam;
        return BlasterPlayerState->GetTeam();
    }
    ```
- BP_BlueFlag, BP_RedFlag를 만들어서 Team설정

## TeamPlayerStarts
- Source > Blaster > PlayerStart 폴더에 PlayerStart상속받는 TeamPlayerStart 생성

- TeamPlayerStart.h
    ``` C++
    #include "Blaster/BlasterTypes/Team.h"
    ...
    public:
        UPROPERTY(EditAnywhere)
        ETeam Team;
    ```
- ChoosePlayerStart가 있긴 하지만, 우리는 Team을 캐릭터 생성 이후에 결정하기에 너무 빨라서 안 쓸 것임
- BlasterCharacter.h
    ``` C++
    protected:
	    void SetSpawnPoint();
	    void OnPlayerStateInitialized();
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/PlayerStart/TeamPlayerStart.h"
    void ABlasterCharacter::PollInit()
    {
        if (BlasterPlayerState == nullptr)
        {
            BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
            if (BlasterPlayerState)
            {
                OnPlayerStateInitialized();
            }
        }
    }
    void ABlasterCharacter::OnPlayerStateInitialized()
    {
        BlasterPlayerState->AddToScore(0.0f);
        BlasterPlayerState->AddToDefeats(0);
        SetTeamColor(BlasterPlayerState->GetTeam());
        SetSpawnPoint();
    }
    void ABlasterCharacter::SetSpawnPoint()
    {
        if (HasAuthority() && BlasterPlayerState->GetTeam() != ETeam::ET_NoTeam)
        {
            TArray<AActor*> PlayerStarts;
            UGameplayStatics::GetAllActorsOfClass(this, ATeamPlayerStart::StaticClass(), PlayerStarts);
            TArray<ATeamPlayerStart*> TeamPlayerStarts;
            for (auto Start : PlayerStarts)
            {
                ATeamPlayerStart* TeamStart = Cast<ATeamPlayerStart>(Start);
                if (TeamStart && TeamStart->Team == BlasterPlayerState->GetTeam())
                {
                    TeamPlayerStarts.Add(TeamStart);
                }
            }
            if (TeamPlayerStarts.Num() > 0)
            {
                ATeamPlayerStart* ChoosenPlayerStart = TeamPlayerStarts[FMath::RandRange(0, TeamPlayerStarts.Num() - 1)];
                SetActorLocationAndRotation(ChoosenPlayerStart->GetActorLocation(), ChoosenPlayerStart->GetActorRotation());
            }
        }
    }
    ```
- Blueprints > PlayerStart 폴더에 TeamPlayerStart를 상속받는 BP_RedTeamStart, BP_BlueTeamStart생성
    - detail에서 Team설정

## Capture the flag gamemode
- TeamsGameMode를 상속받는 CaptureTheFlagGameMode 클래스 생성
- CaptureTheFlagGameMode.h
    ``` C++
    public:
        virtual void PlayerEliminated(class ABlasterCharacter* ElimedCharacer, class ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController) override;
    ```
- CaptureTheFlagGameMode.cpp
    ``` C++
    void ACaptureTheFlagGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
    }
    ```
- Source > Blaster > CaptureTheFlag 폴더에 AActor상속받는 FlagZone생성

- FlagZone.h
    ``` C++
    #include "Blaster/BlasterTypes/Team.h"
    public:	
        AFlagZone();
        UPROPERTY(EditAnywhere)
        ETeam Team;
    protected:
        UFUNCTION()
        virtual void OnSphereOverlap(class UPrimitiveComponent* OverlappedComp, class AActor* OtherActor, class UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
    private:
        UPROPERTY(EditAnywhere)
        class USphereComponent* ZoneSphere;
    ```
- FlagZone.cpp
    ``` C++
    #include "FlagZone.h"
    #include "Components/SphereComponent.h"
    #include "Blaster/Weapon/Flag.h"
    #include "Blaster/GameMode/CaptureTheFlagGameMode.h"
    #include "Blaster/Character/BlasterCharacter.h"
    AFlagZone::AFlagZone()
    {
        // Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
        PrimaryActorTick.bCanEverTick = true;
        ZoneSphere = CreateDefaultSubobject<USphereComponent>(TEXT("ZoneSphere"));
        SetRootComponent(ZoneSphere);
    }
    void AFlagZone::BeginPlay()
    {
        Super::BeginPlay();
        ZoneSphere->OnComponentBeginOverlap.AddDynamic(this, &AFlagZone::OnSphereOverlap);
    }
    void AFlagZone::OnSphereOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        AFlag* OverlappingFlag = Cast<AFlag>(OtherActor);
        if (OverlappingFlag && OverlappingFlag->GetTeam() != Team)
        {
            ACaptureTheFlagGameMode* GameMode = GetWorld()->GetAuthGameMode<ACaptureTheFlagGameMode>();
            if (GameMode)
            {
                GameMode->FlagCaptured(OverlappingFlag, this);
            }
			OverlappingFlag->ResetFlag();
        }
    }
    ```
- Weapon.h
    ``` C++
    public:
    	FORCEINLINE ETeam GetTeam() const { return Team; }
    ```
- Flag.h
    ``` C++
    public:
    	void ResetFlag();
    protected:
    	virtual void BeginPlay() override;
    private:
        FTransform InitialTransform;
    public:
        FORCEINLINE FTransform GetInitialLocation() const { return InitialTransform; }
    ```
- Flag.cpp
    ``` C++
    #include "Blaster/Character/BlasterCharacter.h"
    void AFlag::BeginPlay()
    {
        Super::BeginPlay();
        InitialTransform = GetActorTransform();
    }
    void AFlag::ResetFlag()
    {
        ABlasterCharacter* FlagBearer = Cast<ABlasterCharacter>(GetOwner());
        if (FlagBearer)
        {
            FlagBearer->SetHoldingTheFlag(false);
		    FlagBearer->SetOverlappingWeapon(nullptr);
		    FlagBearer->UnCrouch();
        }
	    if (!HasAuthority()) return;
        FDetachmentTransformRules DetachRules(EDetachmentRule::KeepWorld, true);
        FlagMesh->DetachFromComponent(DetachRules);
	    SetWeaponState(EWeaponState::EWS_Initial);
        GetAreaSphere()->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
        GetAreaSphere()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);
        SetOwner(nullptr);
        BlasterOwnerCharacter = nullptr;
        BlasterOwnerController = nullptr;
        SetActorTransform(InitialTransform);
    }
    void AFlag::OnEquipped()
    {
        ...
        FlagMesh->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
        FlagMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_WorldDynamic, ECollisionResponse::ECR_Overlap);
        ...
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
	    void SetHoldingTheFlag(bool bHolding);
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::SetHoldingTheFlag(bool bHolding)
    {
        if (Combat == nullptr) return;
        Combat->bHoldingTheFlag = bHolding;
    }
    ```
- CaptureTheFlagGameMode.h
    ``` C++
    public:
    	void FlagCaptured(class AFlag* Flag, class AFlagZone* Zone);
    ```
- CaptureTheFlagGameMode.cpp
    ``` C++
    #include "Blaster/Weapon/Flag.h"
    #include "Blaster/CaptureTheFlag/FlagZone.h"
    #include "Blaster/GameState/BlasterGameState.h"

    void ACaptureTheFlagGameMode::FlagCaptured(AFlag* Flag, AFlagZone* Zone)
    {
        bool bValidCapture = Flag->GetTeam() != Zone->Team;
        ABlasterGameState* BGameState = Cast<ABlasterGameState>(GameState);
        if (BGameState)
        {
            if (Zone->Team == ETeam::ET_BlueTeam)
            {
                BGameState->BlueTeamScores();
            }
            if (Zone->Team == ETeam::ET_RedTeam)
            {
                BGameState->RedTeamScores();
            }
        }
    }
    ```
- Blueprints > CaptureTheFlag 폴더에 BP_BlueFlagZone, BP_RedFlagZone 생성
- Blueprints > GameModes 폴더에 BP_CTFGameMode생성


- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::RotateInPlace(float DeltaTime)
    {
        ...
        if (Combat && Combat->EquippedWeapon) GetCharacterMovement()->bOrientRotationToMovement = false;
        if (Combat && Combat->EquippedWeapon) bUseControllerRotationYaw = true;
        ...
    }
    ```

## Select Match Type
- WBP_Menu에 Checkbox 추가(FreeForAllCheckbox, TeamsCheckbox, CaptureTheFlagCheckbox)
    - FreeForAllCheckbox는 CheckedState를 Checked로 설정
    - Blueprint로 CheckboxStateChangedEvent마다 다른 Checkbox uncheck되게 하고, Menu의 MatchType을 변경
- Menu.h
    ``` C++
    private:
        UPROPERTY(BlueprintReadWrite, meta = (AllowPrivateAccess = "true"))
        int32 NumPublicConnections{4};
        UPROPERTY(BlueprintReadWrite, meta = (AllowPrivateAccess="true"))
        FString MatchType{TEXT("FreeForAll")};
    ```
- WBP에 TextBox 추가(NumPlayersTextBox)
    - OnTextChanged에 Set NumPublicConnections 연결


## Accessing Subsystem
- MultiplayerSessionSubsystem.h
    ``` C++
    public:
        int32 DesiredNumPublicConnections{};
        FString DesiredMatchType{};
    ```
- MultiplayerSessionSubsystem.cpp
    ``` C++
    void UMultiplayerSessionsSubsystem::CreateSession(int32 NumPublicConnections, FString MatchType)
    {
        DesiredNumPublicConnections = NumPublicConnections;
        DesiredMatchType = MatchType;
        ...
    }
    ```
- LobbyGameMode.cpp
    ``` C++
    #include "LobbyGameMode.h"
    #include "GameFramework/GameStateBase.h"
    #include "MultiplayerSessionsSubsystem.h"

    void ALobbyGameMode::PostLogin(APlayerController* NewPlayer)
    {
        Super::PostLogin(NewPlayer);
        int32 NumberOfPlayers = GameState.Get()->PlayerArray.Num();

        UGameInstance* GameInstance = GetGameInstance();
        if (GameInstance)
        {
            UMultiplayerSessionsSubsystem* Subsystem = GameInstance->GetSubsystem<UMultiplayerSessionsSubsystem>();
            check(Subsystem);

            if (NumberOfPlayers == Subsystem->DesiredNumPublicConnections)
            {
                UWorld* World = GetWorld();
                if (World)
                {
                    bUseSeamlessTravel = true;

                    FString MatchType = Subsystem->DesiredMatchType;
                    if (MatchType == "FreeForAll")
                    {
                        World->ServerTravel(FString("/Gme/Maps/BlasterMap?listen"));
                    }
                    else if (MatchType == "Teams")
                    {
                        World->ServerTravel(FString("/Gme/Maps/Teams?listen"));
                    }
                    else if (MatchType == "CaptureTheFlag")
                    {
                        World->ServerTravel(FString("/Gme/Maps/CaptureTheFlag?listen"));
                    }
                }
            }
        }
    }
    ```
- Blaster.Build.cs
    ``` C++
    PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "Niagara", "MultiplayerSessions", "OnlineSubsystem", "OnlineSubsystemSteam" });
    ```

- Teams, CaptureTheFlag 레벨 만들기
    - 적절한 게임모드 설정
- List of maps for packaging에 추가
