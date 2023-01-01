## GameFramework
- GameMode : Server Only
- GameState : Server and all Clients
- PlayerState : Server and all Clients
- Player Controller : Server and Owning Client
- Pawn : Server and all clients
- HUD/Widgets : Owning Client Only

## Health
- Blaster > HUD 폴더에 UserWidget을 상속받은 CharacterOverlay클래스 생성
- CharacterOverlay를 상속받은 WBP_CharacterOverlay 생성
    - ProgressBar 추가("HealthBar")
    - TextBlock 추가("HeatlhText")
- CharacterOverlay.h
    ``` C++
    public:
        UPROPERTY(meta = (BindWidget))
        class UProgressBar* HealthBar;
        UPROPERTY(meta = (BindWidget))
        class UTextBlock* HealthText;
    ```
- BlasterHUD.h
    ``` C++
    public:    
        UPROPERTY(EditAnywhere, Category = "Player Stats");
        TSubclassOf<class UUserWidget> CharacterOverlayClass;
        class UCharacterOverlay* CharacterOverlay;
    protected:
        virtual void BeginPlay() override;
	    void AddCharacterOverlay();
    ```
    - BP_BlasterHUD에서 CharacterOverlayClass에 WBP_CharacterOverlay 설정하기
- BlasterHUD.cpp
    ``` C++
    #include "GameFramework/PlayerController.h"
    #include "CharacterOverlay.h"
    void ABlasterHUD::BeginPlay()
    {
        Super::BeginPlay();
        AddCharacterOverlay();
    }
    void ABlasterHUD::AddCharacterOverlay()
    {
        APlayerController* PlayerController = GetOwningPlayerController();
        if (PlayerController && CharacterOverlayClass)
        {
            CharacterOverlay = CreateWidget<UCharacterOverlay>(PlayerController, CharacterOverlayClass);
            CharacterOverlay->AddToViewport();
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    private:
        // Player Health
        UPROPERTY(EditAnywhere, Category = "Player Stats")
        float MaxHealth = 100.0f;
        UPROPERTY(ReplicatedUsing = OnRep_Health, VisibleAnywhere, Category = "Player Stats")
        float Health = 100.0f;
        UFUNCTION()
        void OnRep_Health();
	    class ABlasterPlayerController* BlasterPlayerController;
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/PlayerController/BlasterPlayerController.h"

    void ABlasterCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME_CONDITION(ABlasterCharacter, OverlappingWeapon, COND_OwnerOnly);
        DOREPLIFETIME(ABlasterCharacter, Health);
    }
    void ABlasterCharacter::BeginPlay()
    {
        Super::BeginPlay();
	    BlasterPlayerController = BlasterPlayerController == nullptr ? Cast <ABlasterPlayerController>(Controller) : BlasterPlayerController;
        if (BlasterPlayerController)
        {
            BlasterPlayerController->SetHUDHealth(Health, MaxHealth);
        }
    }
    ```

- BlasterPlayerController.h
    ``` C++
    public:
        void SetHUDHealth(float Health, float MaxHealth);
    protected:
        virtual void BeginPlay() override;
    private:
        class ABlasterHUD* BlasterHUD;
    ```
- BlsterPlayerController.cpp
    ``` C++
    #include "Blaster/HUD/BlasterHUD.h"
    #include "Blaster/HUD/CharacterOverlay.h"
    #include "Components/ProgressBar.h"
    #include "Components/TextBlock.h"

    void ABlasterPlayerController::BeginPlay()
    {
        Super::BeginPlay();
        BlasterHUD = Cast<ABlasterHUD>(GetHUD());

    }
    void ABlasterPlayerController::SetHUDHealth(float Health, float MaxHealth)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->HealthBar &&
            BlasterHUD->CharacterOverlay->HealthText;
        if (bHUDValid)
        {
            const float HealthPercent = Health / MaxHealth;
            BlasterHUD->CharacterOverlay->HealthBar->SetPercent(HealthPercent);
            FString HealthText = FString::Printf(TEXT("%d/%d"), FMath::CeilToInt(Health), FMath::CeilToInt(MaxHealth));
            BlasterHUD->CharacterOverlay->HealthText->SetText(FText::FromString(HealthText));
        }
    }
    ```

## Damage
- Blaster > Weapon 폴더에 Projectile을 상속받는 ProjectileBullet 클래스 생성

- Projectile.h
    ``` C++
    protected:
        UPROPERTY(EditAnywhere)
	    float Damage = 20.0f;
    ```
- Projectile.cpp
    ``` C++
    
    ```
- ProjectileBullet.h
    ``` C++
    protected:
        virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit) override;
    ```
- ProjectileBullet.cpp
    ``` C++
    #include "ProjectileBullet.h"
    #include "Kismet/GamePlayStatics.h"
    #include "GameFramework/Character.h"

    void AProjectileBullet::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        ACharacter* OwnerCharacter = Cast<ACharacter>(GetOwner());
        if (OwnerCharacter)
        {
            AController* OwnerController = OwnerCharacter->Controller;
            if (OwnerController)
            {
                UGameplayStatics::ApplyDamage(OtherActor, Damage, OwnerController, this, UDamageType::StaticClass());
            }
        }
        Super::OnHit(HitComp, OtherActor, OtherComp, NormalImpulse, Hit);
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
	    void Elim();
        UFUNCTION(NetMulticast,Reliable)
        void MulticastElim();
    protected:
	    UFUNCTION()
        void ReceiveDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, class AController* InstigatorController, AActor* DamageCauser);
	    void UpdateHUDHealth();
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::BeginPlay()
    {
        Super::BeginPlay();
        UpdateHUDHealth();
        if (HasAuthority())
        {
            OnTakeAnyDamage.AddDynamic(this, &ABlasterCharacter::ReceiveDamage);
        }
    }
    void ABlasterCharacter::UpdateHUDHealth()
    {
        BlasterPlayerController = BlasterPlayerController == nullptr ? Cast <ABlasterPlayerController>(Controller) : BlasterPlayerController;
        if (BlasterPlayerController)
        {
            BlasterPlayerController->SetHUDHealth(Health, MaxHealth);
        }
    }
    void ABlasterCharacter::ReceiveDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatorController, AActor* DamageCauser)
    {
        Health = FMath::Clamp(Health-Damage, 0.f, MaxHealth);
	    UpdateHUDHealth();
	    PlayHitReactMontage();
    }
    void ABlasterCharacter::OnRep_Health()
    {
	    UpdateHUDHealth();
        PlayHitReactMontage();
    }
    void ABlasterCharacter::Elim()
    {
        MulticastElim();
    }

    void ABlasterCharacter::MulticastElim_Implementation()
    {
        bElimned = true;
        PlayElimMontage();
    }

    ```
    - RPC는 최대한 줄여야 함 -> MulticastHit 부분 없애기
        - Projectile의 OnHit에서도 MulticastHIt 삭제
- Blueprints > Weapon > Projectiles 폴더에 ProjectileBullet을 상속받는 BP_ProjectileBullet 생성
    - 이펙트 등 적절하게 설정
- BP_AssaultRifle에 Projectile을 BP_ProjectileBullet으로 설정


## Blaster GameMode
- 기존의 BlasterGameMode는 GameModeBase를 상속받은것이었는데, 사실 우리가 원하는 것은 GameModeBase임
    - BlasterGameMode 파일 지우고, Binaries, Intermediate, Saved 폴더 지운 후 Generate Project files

- Blaster > GmeMode 폴더에 GameMode를 상속받는 BlasterGameMode 클래스 생성
    - BP_BlasterGameMode를 BlasterGameMode로부터 상속받도록 설정
    - Default Class들 잘 들어가있는지 확인

- BlasterGameMode.h
    ``` C++
    public:
        virtual void PlayerEliminated(class ABlasterCharacter* ElimedCharacer, class ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController);
    ```
- BlasterGameMode.cpp
    ``` C++
    void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
        if (ElimedCharacer)
        {
            ElimedCharacer->Elim();
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	void PlayElimMontage();
    	void Elim();
    private:
	    UPROPERTY(EditAnywhere, Category = Combat)
		UAnimMontage* ElimMontage;
	    bool bElimned = false;
    public:
    	FORCEINLINE bool IsElimned() const { return bElimned; };
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/GameMode/BlasterGameMode.h"
    
    void ABlasterCharacter::ReceiveDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatorController, AActor* DamageCauser)
    {
        Health = FMath::Clamp(Health-Damage, 0.f, MaxHealth);
        UpdateHUDHealth();
        PlayHitReactMontage();

        if (Health == 0.f)
        {
            ABlasterGameMode* BlasterGameMode = GetWorld()->GetAuthGameMode<ABlasterGameMode>();
            if (BlasterGameMode)
            {
                BlasterPlayerController = BlasterPlayerController == nullptr ? Cast<ABlasterPlayerController>(Controller) : BlasterPlayerController;
                ABlasterPlayerController* AttackerController = Cast<ABlasterPlayerController>(InstigatorController);
                BlasterGameMode->PlayerEliminated(this, BlasterPlayerController, AttackerController);
            }
        }
    }
    void ABlasterCharacter::Elim()
    {
	    bElimned = true;
	    PlayElimMontage();
    }
    void ABlasterCharacter::PlayElimMontage()
    {
        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && ElimMontage)
        {
            AnimInstance->Montage_Play(ElimMontage);
        }
    }
    ```
- BlasterAnimInstance.h
    ``` C++
    private:
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
        bool bElimned
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bElimned = BlasterCharacter->IsElimned();
        ...
    }
    ```
- Blueprints > Character > Animation 폴더에 Elim이라는 AnimationMontage 생성
    - ElimSlot이라는 Slot 생성, Elim의 Slot 변경

- BlasterCharacter에서 ElimMontage 설정
- BlasterAnimBP에서 Elimned 값에 따라서 Play Elim->ElimSlot과 연결하던지, 원래대로 연결하던지 분기 설정

## Respawn
- BlasterCharacter.h
    ``` C++
    private:
        FTimerHandle ElimTimer;
        UPROPERTY(EditDefaultsOnly)
        float ElimDelay = 3.f;
        void ElimTimerFinished();
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "TimerManager.h"
    void ABlasterCharacter::Elim()
    {
        MulticastElim();
        GetWorldTimerManager().SetTimer(
            ElimTimer,
            this,
            &ABlasterCharacter::ElimTimerFinished,
            ElimDelay
        );
    }
    void ABlasterCharacter::ElimTimerFinished()
    {
        ABlasterGameMode* BlasterGameMode = GetWorld()->GetAuthGameMode<ABlasterGameMode>();
        if (BlasterGameMode)
        {
            BlasterGameMode->RequestRespawn(this, Controller);
        }
    }
    ```
- BlasterGameMode.h
    ``` C++
	virtual void RequestRespawn(class ACharacter* ElimnedCharacter, AController* ElimnedController);
    ```
- BlasterGameMode.cpp
    ``` C++
    #include "Kismet/GameplayStatics.h"
    #include "GameFramework/PlayerStart.h"

    void ABlasterGameMode::RequestRespawn(ACharacter* ElimnedCharacter, AController* ElimnedController)
    {
        if (ElimnedCharacter)
        {
            ElimnedCharacter->Reset();
            ElimnedCharacter->Destroy();
        }
        if (ElimnedController)
        {
            TArray<AActor*> PlayerStarts;
            UGameplayStatics::GetAllActorsOfClass(this, APlayerStart::StaticClass(), PlayerStarts);
            int32 Selection = FMath::RandRange(0, PlayerStarts.Num() - 1);
            RestartPlayerAtPlayerStart(ElimnedController, PlayerStarts[Selection]);
        }
    }
    ```
- BP_BlasterCharacter에서  SpawnCollisionHandlingMethod를 지정할 수 있음
    - Try to adjust Location, But Always Spawn으로 지정

## Dissolve Material
- 1키를 누르면서 클릭을 하면 Scalar 노드 추가 가능
- 2키를 누르면서 클릭을 하면 2Vector 노드 추가 가능
- 3키를 누르면서 클릭을 하면 Color 노드 추가 가능
- 어떤 노드를 우클릭하고 Start Previewing Node 클릭하면 해당 노드까지의 진행상황 확인 가능
- CheapContrast : cutoff같은 느낌
- Scalar노드를 우클릭하고 Convert to Parameter 누르면 변수 선언 가능

- Material의 Instance를 만들어서 활용 가능

- 기존의 캐릭터 Material을 복제하고, 아까 만든 Dissolve Effect 노드들을 연결
    - BlendMOde를 Masked로 변경해야함
    - Material Instance만들기

- Curve 폴더에 DissolveCurve 생성

- BlasterCharacter.h
    ``` C++
    #include "Components/TimelineComponent.h"
    private:
        // Dissolve Effect
        UPROPERTY(VisibleAnywhere)
        UTimelineComponent* DissolveTimeline;
        FOnTimelineFloat DissolveTrack;
        UPROPERTY(EditAnywhere)
        UCurveFloat* DissolveCurve;
        UFUNCTION()
        void UpdateDissolveMaterial(float DissolveValue);
        void StartDissolve();
        // Dynamic Instance that we can change at runtime
        UPROPERTY(VisibleAnywhere, Category = Elim)
        UMaterialInstanceDynamic* DynamicDissolveMaterialInstance;
        // Material instsance set on the Blueprint
        UPROPERTY(EditAnywhere, Category = Elim)
        UMaterialInstance* DissolveMaterialInstance;
    ```
- BlasterCharacter.cpp
    ``` C++
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
        DissolveTimeline = CreateDefaultSubobject<UTimelineComponent>(TEXT("DissolveTimelineCoponent"));
    }
    void ABlasterCharacter::MulticastElim_Implementation()
    {
        bElimned = true;
        PlayElimMontage();

        if (DissolveMaterialInstance)
        {
            DynamicDissolveMateralIntance = UMaterialInstanceDynamic::Create(DissolveMaterialInstance, this);
            GetMesh()->SetMaterial(0, DynamicDissolveMateralIntance);
            DynamicDissolveMateralIntance->SetScalarParameterValue(TEXT("Dissolve"), 0.55f);
            DynamicDissolveMateralIntance->SetScalarParameterValue(TEXT("Glow"), 200f);
        }
        StartDissolve();
    }
    void ABlasterCharacter::UpdateDissolveMaterial(float DissolveValue)
    {
        if (DynamicDissolveMaterialInstance)
        {
            DynamicDissolveMaterialInstance->SetScalarParameterValue(TEXT("Dissolve"), DissolveValue);
        }
    }

    void ABlasterCharacter::StartDissolve()
    {
        DissolveTrack.BindDynamic(this, &ABlasterCharacter::UpdateDissolveMaterial);
        if (DissolveCurve)
        {
            DissolveTimeline->AddInterpFloat(DissolveCurve, DissolveTrack);
            DissolveTimeline->Play();
        }
    }
    ```
- BP_BlasterCharacter에서 DissolveCurve 설정하고, DissolveMaterialInstance 설정
    - Dynamic Dissolve Instance는 동적으로 생성해줄거임
- Mesh를 Optimized로 바꿔야 Material 하나로 퉁쳐서 Dissolve가 먹힘
    - PhysicsAsset 잘 들어가있는지 확인 필요
## Handle Elim
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::MulticastElim_Implementation()
    {
        ...
        // Disable Character Movement
        GetCharacterMovement()->DisableMovement();
	    GetCharacterMovement()->StopMovementImmediately();
        if (BlasterPlayerController)
        {
            DisableInput(BlasterPlayerController);
        }
        // Disable Collision
        GetCapsuleComponent()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        GetMesh()->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }
    ```
- Weapon.h
    ``` C++
    public:
    	void Dropped();
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::SetWeaponState(EWeaponState State)
    {
        WeaponState = State;

        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ShowPickupWidget(false);
            AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
		    WeaponMesh->SetSimulatePhysics(false);
            WeaponMesh->SetEnableGravity(false);
            WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
            break;
        case EWeaponState::EWS_Dropped:
            if (HasAuthority())
            {
                AreaSphere->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
            }
            WeaponMesh->SetSimulatePhysics(true);
            WeaponMesh->SetEnableGravity(true);
            WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
            break;
        }
    }
    void AWeapon::OnRep_WeaponState()
    {
        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ShowPickupWidget(false);
            WeaponMesh->SetSimulatePhysics(false);
            WeaponMesh->SetEnableGravity(false);
            WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
            break;
        case EWeaponState::EWS_Dropped:
            WeaponMesh->SetSimulatePhysics(true);
            WeaponMesh->SetEnableGravity(true);
            WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
            break;
        }
    }
    void AWeapon::Dropped()
    {
        SetWeaponState(EWeaponState::EWS_Dropped);
        FDetachmentTransformRules DetachRules(EDetachmentRule::KeepWorld, true);
        WeaponMesh->DetachFromComponent(DetachRules);
        SetOwner(nullptr);
    }
    ```
    - GetLifetimeReplicatedProps에서 State가 바뀔 때마다 처리를 해줌
- CombatComponent.cpp   
    ``` C++
    void UCombatComponent::OnRep_EquippedWeapon()
    {
        if (EquippedWeapon && Character)
        {
            EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
            const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(FName("RightHandSocket"));
            if (HandSocket)
            {
                HandSocket->AttachActor(EquippedWeapon, Character->GetMesh());
            }
            Character->GetCharacterMovement()->bOrientRotationToMovement = false;
            Character->bUseControllerRotationYaw = true;
        }
    }
    ```
    - Weapon이 Replicated되는 속도가 Server와 Client가 다르므로, OnRep를 통해 클라이언트에서도 확실하게 순서를 명시
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::Elim()
    {
        if (Combat && Combat->EquippedWeapon)
        {
            Combat->EquippedWeapon->Dropped();
        }
        ...
    }
    ```

## Elim Effect
- BlasterCharacter.h
    ``` C++
    public:
    	virtual void Destroyed() override;
    private:
        // Elim Bot
	    UPROPERTY(EditAnywhere)
	    UParticleSystem* ElimBotEffect;
        UPROPERTY(VisibleAnywhere)
        UParticleSystemComponent* ElimBotComponent;
        UPROPERTY(EditAnywhere)
        class USoundCue* ElimBotSound;
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Kismet/GameplayStatics.h"
    #include "Sound/SoundCue.h"
    #include "Particles/ParticleSystemComponent.h"

    void ABlasterCharacter::MulticastElim_Implementation()
    {
        ...
        // Spawn elem bot
        if (ElimBotEffect)
        {
            FVector ElimBotSpawnPoint(GetActorLocation().X, GetActorLocation().Y, GetActorLocation().Z + 200.f);
            ElimBotComponent = UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ElimBotEffect, ElimBotSpawnPoint, GetActorRotation());
        }
        if (ElimBotSound)
        {
            UGameplayStatics::SpawnSoundAtLocation(this, ElimBotSound, GetActorLocation());
        }
    }
    void ABlasterCharacter::Destroyed()
    {
        Super::Destroyed();
        if (ElimBotComponent)
        {
            ElimBotComponent->DestroyComponent();
        }
    }
    ```
- BP_BlasterCharacter에 ElimBotEffect와 ElimBotSound 설정
- Bot 이펙트에 Lifetime 주기

## OnPossess
- 지금 캐릭터가 죽고 리스폰되면 HealthBar가 원상복구가 안 되는 현상이 있음

- BlasterPlayerController.h
    ``` C++
    public:
        virtual void OnPossess(APawn* InPawn) override;
    ```
- BlasterPlayerController.cpp
    ``` C++
    #include "Blaster/Character/BlasterCharacter.h"
    void ABlasterPlayerController::OnPossess(APawn* InPawn)
    {
        Super::OnPossess(InPawn);
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(InPawn);
        if (BlasterCharacter)
        {
            SetHUDHealth(BlasterCharacter->GetHealth(), BlasterCharacter->GetMaxHealth());
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	FORCEINLINE float GetHealth() const { return Health; };
	    FORCEINLINE float GetMaxHealth() const { return MaxHealth; };
    ```
- BlasterCharacter의 BeginPlay에도 UpdateHUDHealth()는 있어야 함
    - OnPossess가 되었을 때 HUD가 초기화가 안 되어있을 수 있기 때문

## Blaster Player State
- Souce > Blaster > PlayerState 폴더에 PlayerState를 상속받는 BlasterPlayerState 클래스 생성

- PlayerState에는 이미 Score, PlayerName, IsInactive, PlayerId, UniqueId가 replicated처리 되어있음

- BlasterPlayerState.h
    ``` C++
    public:
        virtual void OnRep_Score() override;
	    void AddToScore(float ScoreAmount);
    private:
        class ABlasterCharacter* Character;
        class ABlasterPlayerController* Controller;
    ```
- BlasterPlayerState.cpp
    ``` C++
    #include "BlasterPlayerState.h"
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Blaster/PlayerController/BlasterPlayerController.h"
    void ABlasterPlayerState::OnRep_Score()
    {
        Super::OnRep_Score();

        Character = Character == nullptr ? Cast<ABlasterCharacter>(GetPawn()) : Character;
        if (Character)
        {
            Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
            if (Controller)
            {
                Controller->SetHUDScore(GetScore());
            }
        }
    }

    void ABlasterPlayerState::AddToScore(float ScoreAmount)
    {
	    SetScore(GetScore() + ScoreAmount);
        Character = Character == nullptr ? Cast<ABlasterCharacter>(GetPawn()) : Character;
        if (Character)
        {
            Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
            if (Controller)
            {
                Controller->SetHUDScore(GetScore());
            }
        }
    }
    ```
- WBP_CharacterOverlay에 ScoreText,ScoreAmount 추가

- CharacterOverlay.h
    ``` C++
    public:
	    UPROPERTY(meta = (BindWidget))
		class UTextBlock* ScoreAmount;
    ```
- BlasterPlayerController.h
    ``` C++
    public:
        void SetHUDScore(float Score);
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::SetHUDScore(float Score)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->ScoreAmount;
        if (bHUDValid)
        {
            FString ScoreText = FString::Printf(TEXT("%d"), FMath::FloorToInt(Score));
            BlasterHUD->CharacterOverlay->ScoreAmount->SetText(FText::FromString(ScoreText));
        }
    }
    ```
- BlasterGameMode.cpp
    ``` C++
    #include "Blaster/PlayerState/BlasterPlayerState.h"
    void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
        ABlasterPlayerState* AttackerPlayerState = AttackerController ? Cast<ABlasterPlayerState>(AttackerController->PlayerState) : nullptr;
        ABlasterPlayerState* VictimPlayerState = VictimController ? Cast<ABlasterPlayerState>(VictimController->PlayerState) : nullptr;

        if (AttackerPlayerState && AttackerPlayerState != VictimPlayerState)
        {
            AttackerPlayerState->AddToScore(1.f);
        }
        ...
    }
    ```
- Blueprints > PlayerState 폴더에 BlasterPlayerState를 상속받는 BP_PlayerState 생성
    - BP_BlasterGameMode에서 PlayerState Class를 BP_PlayerState로 설정

- 현재는 게임을 시작 할 때 Score가 업데이트 되지 않아서 이상한 값이 들어가있음
    - BlasterCharacter에서 Poll을 하면서 테스트를 해줄거임
- BlasterCharacter.h
    ``` C++
    protected:
        // Poll for any relevant classes and initialize our HUD
        void PollInit();
    private:
    	class ABlasterPlayerState* BlasterPlayerState;
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/PlayerState/BlasterPlayerState.h"

    void ABlasterCharacter::PollInit()
    {
        if (BlasterPlayerState == nullptr)
        {
            BlasterPlayerState = GetPlayerState<ABlasterPlayerState>();
            if (BlasterPlayerState)
            {
                BlasterPlayerState->AddToScore(0.0f);
            }
        }
    }
    void ABlasterCharacter::Tick(float DeltaTime)
    {
        ...
        PollInit();
    }
    ```

## Defeats
- CharacterOverlay에 DefeatsText와 DefeatsAmount 추가

- CharacterOverlay.h
    ``` C++
    public:
        UPROPERTY(meta = (BindWidget))
		UTextBlock* DefeatsAmount;
    ```
- BlasterPlayerController.h
    ``` C++
    public:
        void SetHUDDefeats(int32 Defeats);
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::SetHUDDefeats(int32 Defeats)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->DefeatsAmount;
        if (bHUDValid)
        {
            FString DefeatsText = FString::Printf(TEXT("%d"), Defeats);
            BlasterHUD->CharacterOverlay->DefeatsAmount->SetText(FText::FromString(DefeatsText));
        }
    }
    ```
- BlasterPlayerState.h
    ``` C++
    public:
        virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
	    UFUNCTION()
	    virtual void OnRep_Defeats();
	    void AddToDefeats(int32 DefeatsAmount);
    private:
        UPROPERTY(ReplicatedUsing = OnRep_Defeats)
        int32 Defeats;
    ```
- BlasterPlayerState.cpp
    ``` C++
    #include "Net/UnrealNetwork.h"
        
    void ABlasterPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);

        DOREPLIFETIME(ABlasterPlayerState, Defeats);
    }
    void ABlasterPlayerState::AddToDefeats(int32 DefeatsAmount)
    {
        Defeats += DefeatsAmount;
        Character = Character == nullptr ? Cast<ABlasterCharacter>(GetPawn()) : Character;
        if (Character)
        {
            Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
            if (Controller)
            {
                Controller->SetHUDDefeats(Defeats);
            }
        }
    }
    ```
- BlasterGameMode.cpp
    ``` C++
    void ABlasterGameMode::PlayerEliminated(ABlasterCharacter* ElimedCharacer, ABlasterPlayerController* VictimController, ABlasterPlayerController* AttackerController)
    {
        ...
        if (VictimPlayerState)
        {
            VictimPlayerState->AddToDefeats(1);
        }
        ...
    }
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
                BlasterPlayerState->AddToScore(0.0f);
                BlasterPlayerState->AddToDefeats(0);
            }
        }
    }
    ```

- TIP) BlasterPlayerState의 Character, Controller는 사실 nullptr로 초기화가 안 되어있을 수 있음
    - 첫 번째 방법은 class ABlasterCharacter* Character = nullptr이라 해주는 것
    - 두 번쨰 방법은 UPROPERTY()를 넣어주는 것
    - BlasterPlayerState.h
        ``` C++
        private:
            UPROPERTY()
            class ABlasterCharacter* Character;
            UPROPERTY()
            class ABlasterPlayerController* Controller;
        ```
    - BlasterCharacter.h
        ``` C++
        private:
            UPROPERTY()
            class ABlasterPlayerController* BlasterPlayerController;
            UPROPERTY()
            class ABlasterPlayerState* BlasterPlayerState;
        ```
    - BlasterPlayerController.h
        ``` C++
        private:
            UPROPERTY()
            class ABlasterHUD* BlasterHUD;
        ```
    - CombatComponent.h
        ``` C++
        private:
            UPROPERTY()
            class ABlasterCharacter* Character;
            UPROPERTY()
            class ABlasterPlayerController* Controller;
            UPROPERTY()
            class ABlasterHUD* HUD;
        ```
    - Projectile.h
        ``` C++
        private:
            UPROPERTY()
            class UParticleSystemComponent* TracerComponent;
        ```