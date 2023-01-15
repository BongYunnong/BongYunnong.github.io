## Pickup
- Source > Blaster > Pickups 폴더에 Actor기반으로 Pickup 클래스 생성
- Pickup.h
    ```C++
    public:
    	virtual void Destroyed() override;
    protected:
        UFUNCTION()
        virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
    private:
        UPROPERTY(EditAnywhere)
        class USphereComponent* OverlapShere;
        UPROPERTY(EditAnywhere)
        class USoundCue* PickupSound;
	    UPROPERTY(EditAnywhere)
		UStaticMeshComponent* PickupMesh;
    ```
- Pickup.cpp
    ``` C++
    #include "Kismet/GameplayStatics.h"
    #include "Sound/SoundCue.h"
    #include "Components/SphereComponent.h"

    APickup::APickup()
    {
        PrimaryActorTick.bCanEverTick = true;
        bReplicates = true;

        RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));
        OverlapShere = CreateDefaultSubobject<USphereComponent>(TEXT("OverlapSphere"));
        OverlapShere->SetupAttachment(RootComponent);
        OverlapShere->SetSphereRadius(150.f);
        OverlapShere->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
        OverlapShere->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
        OverlapShere->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);

        PickupMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("PickupMesh"));
        PickupMesh->SetupAttachment(OverlapShere);
        PickupMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }
    void APickup::BeginPlay()
    {
        Super::BeginPlay();
        if (HasAuthority())
        {
            OverlapShere->OnComponentBeginOverlap.AddDynamic(this, APickup::OnSphereOverlap);
        }
    }
    void APickup::Destroyed()
    {
        Super::Destroyed();
        if (PickupSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                this,
                PickupSound,
                GetActorLocation()
            );
        }
    }
    ```
- Blueprints >Pickups 폴더에 BP_Pickup만들기
    - StaticMesh 설정, 크기 설정, CustomDepth250으로 설정

## Ammo Pickup
- Pickup을 상속받아 AmmoPickup 클래스 생성
- AmmoPickup.h
    ``` C++
    #include "Blaster/Weapon/WeaponTypes.h"
    protected:
        virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult) override;

    private:
        UPROPERTY(EditAnywhere)
        int32 AmmoAmount = 30;
        UPROPERTY(EditAnywhere)
        EWeaponType WeaponType;
    ```
- AmmoPickup.cpp
    ``` C++
    #include "AmmoPickup.h"
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Blaster/BlasterComponents/CombatComponent.h"
    void AAmmoPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        Super::OnSphereOverlap(OverlappedComponente, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            UCombatComponent* Combat = BlasterCharacter->GetCombat();
            if (Combat)
            {
                Combat->PickupAmmo(WeaponType, AmmoAmount);
            }
        }
        Destroy();
    }
    ```
- CombatComponent.h
    ``` C++
    public:
        void PickupAmmo(EWeaponType WeaponType, int32 AmmoAmount);
    private:
        UPROPERTY(EditAnywhere)
        int32 MaxCarriedAmmo = 500;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::PickupAmmo(EWeaponType WeaponType, int32 AmmoAmount)
    {
        if (CarriedAmmoMap.Contains(WeaponType))
        {
            CarriedAmmoMap[WeaponType] = FMath::Clamp(CarriedAmmoMap[WeaponType] + AmmoAmount, 0, MaxCarriedAmmo);
		    UpdateCarriedAmmo();
        }
        if (EquippedWeapon && EquippedWeapon->IsEmpty() && EquippedWeapon->GetWeaponType() == WeaponType)
        {
            Reload();
        }
    }
    ```
- Blueprints > Pickups에 AmmoPickup 상속받은 BP_ARAmmo 생성
    - Mesh 설정, 크기 설정, Custom Depth 설정, PickupSound 추가

- Pickup.h
    ``` C++
    protected:
    	UPROPERTY(EditAnywhere)
		float BaseTurnRate = 45.f;
    ```
- Pickup.cpp
    ``` C++
    #include "Blaster/Weapon/WeaponTypes.h"
    APickup::APickup()
    {
        ...
        OverlapShere->AddLocalOffset(FVector(0.f,0.f,85.0f));
        ...
	    PickupMesh->SetRelativeScale3D(FVector(3.f, 3.f, 3.0f));
        PickupMesh->SetCustomDepthStencilValue(true);
        PickupMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_PURPLE);
    }
    void APickup::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        if (PickupMesh)
        {
            PickupMesh->AddWorldRotation(FRotator(0.f, BaseTurnRate * DeltaTime, 0.f));
        }
    }
    ```

- BP_PistolAmmo도 추가, 적절한 AmmoCount와 WeaponType 설정

## Buff Component
- Source > Blaster > BlasterComponents 폴더에 Actor Componenet를 상속받은 BuffComponent추가
- BuffComponent.h
    ``` C++
    private:
        UPROPERTY()
        class ABlasterCharacter* Character;
    ```

- BlasterCharacter.h
    ``` C++
    public:	
        friend class ABlasterCharacter;
    private:
        UPROPERTY(VisibleAnywhere)
        class UBuffComponent* Buff;
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/BlasterComponents/BuffComponent.h"
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
        Buff = CreateDefaultSubobject<UBuffComponent>(TEXT("BuffComponent"));
        Buff->SetIsReplicated(true);
        ...
    }
    void ABlasterCharacter::PostInitializeComponents()
    {
        Super::PostInitializeComponents();
        if (Combat)
        {
            Combat->Character = this;
        }
        if (Buff)
        {
            Buff->Character = this;
        }
    }
    ```

## Health Pickup
- 나이아가라 시스템에서 이펙트가 계속 살아있게 하고싶다면? Kill Particles when lifetime has elapsed를 체크 해제
- Source > Blaster > Pickups에 Pickup을 상속받은 HealthPickup생성
- HealthPickup.h
    ``` C++
    public:
        AHealthPickup();
    protected:
        virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult) override;
    private:
        UPROPERTY(EditAnywhere)
            float HealAmount = 100.0f;
        UPROPERTY(EditAnywhere)
            float HealingTime = 5.0f;
    ```
- HealthPickup.cpp
    ``` C++
    #include "HealthPickup.h"
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Blaster/BlasterComponents/BuffComponent.h"
    AHealthPickup::AHealthPickup()
    {
        bReplicates = true;
    }
    void AHealthPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        Super::OnSphereOverlap(OverlappedComponente, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            UBuffComponent* Buff = BlasterCharacter->GetBuff();
            if (Buff)
            {
                Buff->Heal(HealAmount, HealingTime);
            }
        }
        Destroy();
    }
    ```
- Pickup.h
    ``` C++
    private:
    	UPROPERTY(VisibleAnywhere)
		class UNiagaraComponent* PickupEffectComponent;
	    UPROPERTY(EditAnywhere)
		class UNiagaraSystem* PickupEffect;
    ```
- Pickup.cpp
    ``` C++
    #include "NiagaraComponent.h"
    #include "NiagaraFunctionLibrary.h"
    APickup::APickup()
    {
        ...
        PickupEffectComponent = CreateDefaultSubobject<UNiagaraComponent>(TEXT("PickupEffectComponent"));
        PickupEffectComponent->SetupAttachment(RootComponent);
    }
    void APickup::Destroyed()
    {
        Super::Destroyed();
        if (PickupEffect)
        {
            UNiagaraFunctionLibrary::SpawnSystemAtLocation(
                this,
                PickupEffect,
                GetActorLocation(),
                GetActorRotation()
            );
        }
        if (PickupSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                this,
                PickupSound,
                GetActorLocation()
            );
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	void UpdateHUDHealth(); // protected에서 public으로
    private:
        UFUNCTION()
        void OnRep_Health(float LastHealth);
    public:
    	FORCEINLINE UBuffComponent* GetBuff() const { return Buff; };
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::OnRep_Health(float LastHealth)
    {
        UpdateHUDHealth();
        if (Health < LastHealth)
        {
            PlayHitReactMontage();
        }
    }
    ```
- BuffComponent.h
    ``` C++
    public:
    	void Heal(float HealAmount, float HealingTime);
    protected:
    	void HealRampUp(float DeltaTime);
    private:
        bool bHealing = false;
        float HealingRate = 0;
        float AmmountToHeal = 0.f;
    public:
    	FORCEINLINE void SetHealth(float Amount) { Health = Amount; };
    ```
- BuffComponent.cpp
    ``` C++
    void UBuffComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
        HealRampUp(DeltaTime);
    }
    void UBuffComponent::HealRampUp(float DeltaTime)
    {
        if (!bHealing || Character == nullptr || Character->IsElimmed()) return;

        const float HealThisFrame = HealingRate * DeltaTime;
        Character->SetHealth(FMath::Clamp(Character->GetHealth() + HealThisFrame, 0.f, Character->GetMaxHealth()));
	    Character->UpdateHUDHealth();
        AmmountToHeal -= HealThisFrame;
        if (AmmountToHeal <= 0.f || Character->GetHealth() >= Character->GetMaxHealth())
        {
            bHealing = false;
            AmmountToHeal = 0.f;
        }
    }
    void UBuffComponent::Heal(float HealAmount, float HealingTime)
    {
        bHealing = true;
        HealingRate = HealAmount / HealingTime;
        AmmountToHeal += HealAmount;
    }
    ```
- Blueprints > Pickups > BuffPickups 폴더에 BP_HealthPickup 생성

## Speed Buffs
- Pickup을 기반으로 SpeedPickup클래스 생성
- SpeedPickup.h
    ``` C++
    protected:
        virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult) override;
    private:
        UPROPERTY(EditAnywhere)
            float BaseSpeedBuff = 1600.f;
        UPROPERTY(EditAnywhere)
            float CrouchBaseSpeedBuff = 850.0f;
        UPROPERTY(EditAnywhere)
            float SpeedBuffTime = 30.f;
    ```
- SpeedPickup.cpp
    ``` C++
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Blaster/BlasterComponents/BuffComponent.h"

    void ASpeedPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        Super::OnSphereOverlap(OverlappedComponente, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            UBuffComponent* Buff = BlasterCharacter->GetBuff();
            if (Buff)
            {
                Buff->BuffSpeed(BaseSpeedBuff, CrouchBaseSpeedBuff, SpeedBuffTime);
            }
        }
        Destroy();
    }
    ```

- BuffComponent.h
    ``` C++
    public:
    	void BuffSpeed(float BuffBaseSpeed, float BuffCrouchSpeed, float BuffTime);
	    void SetInitialSpeeds(float BaseSpeed, float CrouchSpeed);
    private:
        // Speed Buff
        FTimerHandle SpeedBuffTimer;
        void ResetSpeeds();
        float InitialBaseSpeed;
        float InitialCrouchSpeed;
        UFUNCTION(NetMulticast, Reliable)
        void MulticastSpeedBuff(float BaseSpeed, float CrouchSpeed);
    ```
- BuffComponent.cpp
    ``` C++
    #include "GameFramework/CharacterMovementComponent.h"

    void UBuffComponent::SetInitialSpeeds(float BaseSpeed, float CrouchSpeed)
    {
        InitialBaseSpeed = BaseSpeed;
        InitialCrouchSpeed = CrouchSpeed;
    }
    void UBuffComponent::BuffSpeed(float BuffBaseSpeed, float BuffCrouchSpeed, float BuffTime)
    {
        if (Character == nullptr) return;
        Character->GetWorldTimerManager().SetTimer(
            SpeedBuffTimer,
            this,
            &UBuffComponent::ResetSpeeds,
            BuffTime
        );
        if (Character->GetCharacterMovement())
        {
            Character->GetCharacterMovement()->MaxWalkSpeed = BuffBaseSpeed;
            Character->GetCharacterMovement()->MaxWalkSpeedCrouched = BuffCrouchSpeed;
        }
	    MulticastSpeedBuff(BuffBaseSpeed, BuffCrouchSpeed);
    }
    void UBuffComponent::ResetSpeeds()
    {
        if (Character == nullptr || Character->GetCharacterMovement() == nullptr) return;
        Character->GetCharacterMovement()->MaxWalkSpeed = InitialBaseSpeed;
        Character->GetCharacterMovement()->MaxWalkSpeedCrouched = InitialCrouchSpeed;
	    MulticastSpeedBuff(InitialBaseSpeed, InitialCrouchSpeed);
    }
    void UBuffComponent::MulticastSpeedBuff_Implementation(float BaseSpeed, float CrouchSpeed)
    {
        Character->GetCharacterMovement()->MaxWalkSpeed = BaseSpeed;
        Character->GetCharacterMovement()->MaxWalkSpeedCrouched = CrouchSpeed;
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PostInitializeComponents()
    {
        ...
        if (Buff)
        {
            Buff->Character = this;
            Buff->SetInitialSpeeds(GetCharacterMovement()->MaxWalkSpeed, GetCharacterMovement()->MaxWalkSpeedCrouched);
        }
    }
    void UBuffComponent::SetInitialSpeeds(float BaseSpeed, float CrouchSpeed)
    {
        InitialBaseSpeed = BaseSpeed;
        InitialCrouchSpeed = CrouchSpeed;
    }
    ```
- Blueprints > Pickups > BuffPickups 폴더에 SpeedPickup을 상속받는 BP_SpeedPickup생성

## Jump Buffs
- Source > Blaster > Pickups폴더에 Pickup을 상속받은 JumpPickup생성
- JumpPickup.h
    ``` C++
    protected:
        virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult) override;
    private:
        UPROPERTY(EditAnywhere)
            float JumpZVelocityBuff = 4000.f;
        UPROPERTY(EditAnywhere)
            float JumpBuffTime = 30.f;
    ```
- JumpPickup.cpp
    ``` C++

    ```
- BuffComponent.h
    ``` C++
    public:
        void BuffJump(float BuffJumpVelocity, float BuffTime);
    	void SetInitialJumpVelocity(float Velocity);
    private:
        // Jump Buff
        FTimerHandle JumpBuffTimer;
        void ResetJump();
        float InitialJumpVelocity;
        UFUNCTION(NetMulticast, Reliable)
        void MulticastSpeedBuff(float JumpVelocity);
    ```
- BuffComponent.cpp
    ``` C++
    void UBuffComponent::SetInitialJumpVelocity(float Velocity)
    {
	    InitialJumpVelocity = Velocity;
    }
    void UBuffComponent::BuffJump(float BuffJumpVelocity, float BuffTime)
    {
        if (Character == nullptr) return;
        Character->GetWorldTimerManager().SetTimer(
            JumpBuffTimer,
            this,
            &UBuffComponent::ResetJump,
            BuffTime
        );
        if (Character->GetCharacterMovement())
        {
            Character->GetCharacterMovement()->JumpZVelocity = BuffJumpVelocity;
        }
	    MulticastJumpBuff(BuffJumpVelocity);
    }
    void UBuffComponent::ResetJump()
    {
        if (Character->GetCharacterMovement())
        {
            Character->GetCharacterMovement()->JumpZVelocity = InitialJumpVelocity;
        }
	    MulticastJumpBuff(InitialJumpVelocity);
    }
    void UBuffComponent::MulticastJumpBuff_Implementation(float JumpVelocity)
    {
        if (Character == nullptr || Character->GetCharacterMovement() == nullptr) return;

        Character->GetCharacterMovement()->JumpZVelocity = JumpVelocity;
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PostInitializeComponents()
    {
        ...
        if (Buff)
        {
            Buff->Character = this;
            Buff->SetInitialSpeeds(GetCharacterMovement()->MaxWalkSpeed, GetCharacterMovement()->MaxWalkSpeedCrouched);
            Buff->SetInitialJumpVelocity(GetCharacterMovement()->JumpZVelocity);
        }
    }
    ```

- Blueprints > Pickups > BuffPickups에 JumpPickup 상속받는 BP_JumpPickup생성


## Shield Bar
- WBP_CharacteOverlay에 ShieldBar ShieldText추가
- CharacterOverlay.h
    ``` C++
    public:
    	UPROPERTY(meta = (BindWidget))
		UProgressBar* ShieldBar;
    	UPROPERTY(meta = (BindWidget))
		UTextBlock* ShieldText;
    ```
- BlasterPlayerController.h
    ``` C++
    public:
    	void SetHUDShield(float Shield, float MaxShield);
    private:
	    //bool bInitializeCharacterOverlay = false;
	    bool bInitializeHealth = false;
        bool bInitializeScore = false;
        bool bInitializeDefeats = false;
        bool bInitializeGrenades = false;
	    bool bInitializeShield = false;
        float HUDShield;
        float HUDMaxShield;
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::SetHUDShield(float Shield, float MaxShield)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->ShieldBar &&
            BlasterHUD->CharacterOverlay->ShieldText;
        if (bHUDValid)
        {
            const float ShieldPercent = Shield / MaxShield;
            BlasterHUD->CharacterOverlay->ShieldBar->SetPercent(ShieldPercent);
            FString ShieldText = FString::Printf(TEXT("%d/%d"), FMath::CeilToInt(Shield), FMath::CeilToInt(MaxShield));
            BlasterHUD->CharacterOverlay->ShieldText->SetText(FText::FromString(ShieldText));
        }
        else
        {
            bInitializeShield = true;
            HUDShield = Shield;
            HUDMaxShield = MaxShield;
        }
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
                    if(bInitializeHealth) SetHUDHealth(HUDHealth, HUDMaxHealth);
                    if (bInitializeShield) SetHUDShield(HUDShield, HUDMaxShield);
                    if (bInitializeScore) SetHUDScore(HUDScore);
                    if (bInitializeDefeats) SetHUDDefeats(HUDDefeats);
                    ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(GetPawn());
                    if (BlasterCharacter && BlasterCharacter->GetCombat())
                    {
                        if (bInitializeGrenades) SetHUDGrenades(BlasterCharacter->GetCombat()->GetGrenades());
                    }
                }
            }
        }
    }
    ```
    - 각 SetHUd에 적절한 bInitiailizeXXX = true;넣기
- BlasterCharacter.h
    ``` C++
    public:
    	void UpdateHUDShield();
    private:
        // Player Shield
        UPROPERTY(EditAnywhere, Category = "Player Stats")
		float MaxShield = 100.0f;
	    UPROPERTY(ReplicatedUsing = OnRep_Shield, EditAnywhere, Category = "Player Stats")
		float Shield = 100.0f;
	    UFUNCTION()
		void OnRep_Shield(float LastShield);
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::BeginPlay()
    {
        ...
        UpdateHUDShield();
        ...
    }
    void ABlasterCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        ...
        DOREPLIFETIME(ABlasterCharacter, Shield);
    }
    void ABlasterCharacter::UpdateHUDShield()
    {
        BlasterPlayerController = BlasterPlayerController == nullptr ? Cast <ABlasterPlayerController>(Controller) : BlasterPlayerController;
        if (BlasterPlayerController)
        {
            BlasterPlayerController->SetHUDShield(Shield, MaxShield);
        }
    }
    void ABlasterCharacter::OnRep_Shield(float LastShield)
    {
        UpdateHUDShield();
        if (Shield < LastShield)
        {
            PlayHitReactMontage();
        }
    }
    void ABlasterCharacter::ReceiveDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatorController, AActor* DamageCauser)
    {
        if (bElimned) return;
        float DamageToHealth = Damage;
        if (Shield > 0.f)
        {
            if (Shield >= Damage)
            {
                Shield = FMath::Clamp(Shield - Damage, 0.f, MaxShield);
                DamageToHealth = 0.f;
            }
            else
            {
                Shield = 0.f;
                DamageToHealth = FMath::Clamp(DamageToHealth - Shield, 0.f, Damage);
            }
        }
        Health = FMath::Clamp(Health- DamageToHealth, 0.f, MaxHealth);
        UpdateHUDHealth();
        UpdateHUDShield();
        ...
    }
    ```

## Shield Buff
- Source > Blaster > Pickups폴더에 Pickup을 상속받는 ShieldPickup 생성
- ShieldPickup.h
    ``` C++
    protected:
        virtual void OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult) override;
    private:
        UPROPERTY(EditAnywhere)
            float ShieldReplenishAmount = 100.0f;
        UPROPERTY(EditAnywhere)
            float ShieldReplenishTime = 5.0f;
    ```
- ShieldPickup.cpp
    ``` C++
    #include "ShieldPickup.h"
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Blaster/BlasterComponents/BuffComponent.h"
    void AShieldPickup::OnSphereOverlap(UPrimitiveComponent* OverlappedComponente, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        Super::OnSphereOverlap(OverlappedComponente, OtherActor, OtherComp, OtherBodyIndex, bFromSweep, SweepResult);

        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            UBuffComponent* Buff = BlasterCharacter->GetBuff();
            if (Buff)
            {
                Buff->ReplenishShield(ShieldReplenishAmount, ShieldReplenishTime);
            }
        }
        Destroy();
    }
    ```
- BuffComponent.h
    ``` C++
    public:
    	void ReplenishShield(float ShieldAmount, float ReplenishTime);
    protected:
    	void ShieldRampUp(float DeltaTime);
    private:
        // Shield Buff
        bool bReplenishingShield = false;
        float ShieldReplenishRate = 0.f;
        float ShieldReplenishAmmount = 0.f;
    ```
- BuffComponent.cpp
    ``` C++
    void UBuffComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
        HealRampUp(DeltaTime);
        ShieldRampUp(DeltaTime);
    }
    void UBuffComponent::ReplenishShield(float ShieldAmount, float ReplenishTime)
    {
        bReplenishingShield = true;
        ShieldReplenishRate = ShieldAmount / ReplenishTime;
        ShieldReplenishAmmount += ShieldReplenishRate;
    }
    void UBuffComponent::ShieldRampUp(float DeltaTime)
    {
        if (!bReplenishingShield || Character == nullptr || Character->IsElimmed()) return;

        const float ReplenishThisFrame = ShieldReplenishRate * DeltaTime;
        Character->SetShield(FMath::Clamp(Character->GetShield() + ReplenishThisFrame, 0.f, Character->GetMaxShield()));
        Character->UpdateHUDShield();
        ShieldReplenishAmount -= ReplenishThisFrame;
        if (ShieldReplenishAmount <= 0.f || Character->GetShield() >= Character->GetMaxShield())
        {
            bReplenishingShield = false;
            ShieldReplenishAmount = 0.f;
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
        FORCEINLINE float GetShield() const { return Shield; };
        FORCEINLINE void SetShield(float Amount) { Shield = Amount; };
	    FORCEINLINE float GetMaxShield() const { return MaxShield; };
    ```

- Blueprints > Pickups > BufffPickups 폴더에 ShieldPickup을 상속받은 BP_ShieldPickup생성

## Pickup SpawnPoint
- Source > Blaster > Pickups 폴더에 Actor상속받은 PickupSpawnPoint생성

- PickupSpawnPoint.h
    ``` C++
    protected:
        virtual void BeginPlay() override;
        UPROPERTY(EditAnywhere)
        TArray<TSubclassOf<class APickup>> PickupClasses;
	    UPROPERTY()
		APickup* SpawnedPickup;

        void SpawnPickup();
        void SpawnPickupTimerFinished();
	    UFUNCTION()
        void StartSpawnPickupTimer(AActor* DestroyedActor);
    private:
        FTimerHandle SpawnPickupTimer;
        UPROPERTY(EditAnywhere)
        float SpawnPickupTimeMin;
        UPROPERTY(EditAnywhere)
        float SpawnPickupTimeMax;
    ```
- PickupSpawnPoint.cpp
    ``` C++
    #include "Pickup.h"
    APickupSpawnPoint::APickupSpawnPoint()
    {
        PrimaryActorTick.bCanEverTick = true;
        bReplicates = true;
    }
    void APickupSpawnPoint::BeginPlay()
    {
        Super::BeginPlay();
	    StartSpawnPickupTimer((AActor*)nullptr);
    }
    void APickupSpawnPoint::SpawnPickup()
    {
        int32 NumPickupClasses = PickupClasses.Num();
        if (NumPickupClasses > 0)
        {
            int32 Selection = FMath::RandRange(0, NumPickupClasses - 1);
            SpawnedPickup = GetWorld()->SpawnActor<APickup>(PickupClasses[Selection], GetActorTransform());
            if (HasAuthority() && SpawnedPickup)
            {
                SpawnedPickup->OnDestroyed.AddDynamic(this, &APickupSpawnPoint::StartSpawnPickupTimer);
            }
        }
    }
    void APickupSpawnPoint::SpawnPickupTimerFinished()
    {
        if (HasAuthority())
        {
            SpawnPickup();
        }
    }
    void APickupSpawnPoint::StartSpawnPickupTimer(AActor* DestroyedActor)
    {
        const float SpawnTime = FMath::FRandRange(SpawnPickupTimeMin, SpawnPickupTimeMax);
        GetWorldTimerManager().SetTimer(
            SpawnPickupTimer,
            this,
            &APickupSpawnPoint::SpawnPickupTimerFinished,
            SpawnTime
        );
    }
    ```

- Blueprints > Pickups 폴더에 PickupSpawnPoint를 상속받는 BP_BuffSpawnPoint, BP_AmmoSpawnPoint생성

- 여기서, OverlapEvent를 bind하기 전에 플레이어가 Overlap에 들어가있어서 우리가 만든 OnSphereOverlap을 호출하지도 못하고 Destroy되는 문제 해결
- Pickup.h
    ``` C++
    private:
        FTimerHandle BindOverlapTimer;
        float BindOverlapTime = 0.25f;
        UFUNCTION()
        void BindOverlapTimerFinished();
    ```
- Pickup.cpp
    ``` C++
    void APickup::BeginPlay()
    {
        Super::BeginPlay();
        if (HasAuthority())
        {
            GetWorldTimerManager().SetTimer(BindOverlapTimer, this, &APickup::BindOverlapTimerFinished, BindOverlapTime);
        }
    }
    void APickup::BindOverlapTimerFinished()
    {
        OverlapShere->OnComponentBeginOverlap.AddDynamic(this, &APickup::OnSphereOverlap);
    }
    ```

## DefaultWeapon
- BlasterCharacter.h
    ``` C++
    public:
    	void SpawnDefaultWeapon();
    private:
        // Default Weapon
        UPROPERTY(EditAnywhere)
        TSubclassOf<AWeapon> DefaultWeaponClass;
    protected:
    	void UpdateHUDAmmo();
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::BeginPlay()
    {
        Super::BeginPlay();
        SpawnDefaultWeapon();
	    UpdateHUDAmmo();
        ...
    }
    void ABlasterCharacter::SpawnDefaultWeapon()
    {
        ABlasterGameMode* BlasterGameMode = Cast<ABlasterGameMode>(UGameplayStatics::GetGameMode(this));
        UWorld* World = GetWorld();
        if (BlasterGameMode && World && !bElimned && DefaultWeaponClass)
        {
            AWeapon* StartingWeapon = World->SpawnActor<AWeapon>(DefaultWeaponClass);
		    StartingWeapon->bDestroyWeapon = true;
            if (Combat)
            {
                Combat->EquipWeapon(StartingWeapon);
            }
        }
    }
    void ABlasterCharacter::UpdateHUDAmmo()
    {
        BlasterPlayerController = BlasterPlayerController == nullptr ? Cast <ABlasterPlayerController>(Controller) : BlasterPlayerController;
        if (BlasterPlayerController && Combat && Combat->EquippedWeapon)
        {
            BlasterPlayerController->SetHUDCarriedAmmo(Combat->CarriedAmmo);
            BlasterPlayerController->SetHUDWeaponAmmo(Combat->EquippedWeapon->GetAmmo());
        }
    }
    void ABlasterCharacter::Elim()
    {
        if (Combat && Combat->EquippedWeapon)
        {
            if (Combat->EquippedWeapon->bDestroyWeapon)
            {
			    Combat->EquippedWeapon->Destroy();
            }
            else
            {
                Combat->EquippedWeapon->Dropped();
            }
        }
        ...
    }
    ```
- BlasterPlayerController.h
    ``` C++
    private:
        bool bInitializeCarriedAmmo = false;
        float HUDCarriedAmmo;
        bool bInitializeWeaponAmmo = false;
        float HUDWeaponAmmo;
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::SetHUDWeaponAmmo(int32 Ammo)
    {
        ...
        if (bHUDValid)
        {
            ...
        }
        else
        {
            bInitializeWeaponAmmo = true;
            HUDWeaponAmmo = Ammo;
        }
    }
    void ABlasterPlayerController::SetHUDCarriedAmmo(int32 Ammo)
    {
        ...
        if (bHUDValid)
        {
            ...
        }
        else
        {
            bInitializeCarriedAmmo = true;
            HUDCarriedAmmo = Ammo;
        }
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
                    ...
                    if (bInitializeWeaponAmmo) SetHUDWeaponAmmo(HUDWeaponAmmo);
                    if (bInitializeCarriedAmmo) SetHUDCarriedAmmo(HUDCarriedAmmo);
                    ...
                }
            }
        }
    }
    ```
- Weapon.h
    ``` C++
    public:
    	bool bDestroyWeapon = false;
    ```

## Secondary Weapon
- CombatComponent.h
    ``` C++
    private:
    	UPROPERTY(ReplicatedUsing = OnRep_SecondaryWeapon)
		class AWeapon* SecondaryWeapon;
    protected:
    	UFUNCTION()
		void OnRep_SecondaryWeapon();
	    void EquipPrimaryWeapon(AWeapon* WeaponToEquip);
	    void EquipSecondaryWeapon(AWeapon* WeaponToEquip);

	    void PlayEquipWeaponSound(AWeapon* WeaponToEquip);
	    void AttachActorToBackpack(AActor* ActorToAttach);
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(UCombatComponent, EquippedWeapon);
        DOREPLIFETIME(UCombatComponent, SecondaryWeapon);
        ...
    }
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        ...
        if (EquippedWeapon != nullptr && SecondaryWeapon == nullptr)
        {
            EquipSecondaryWeapon(WeaponToEquip);
        }
        else
        {
            EquipPrimaryWeapon(WeaponToEquip);
        }
        ...
    }
    void UCombatComponent::EquipPrimaryWeapon(AWeapon* WeaponToEquip)
    {
	    if (WeaponToEquip == nullptr) return;
        DropEquippedWeapon();
        EquippedWeapon = WeaponToEquip;
        EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
        AttachActorToRightHand(EquippedWeapon);
        EquippedWeapon->SetOwner(Character);
        EquippedWeapon->SetHUDAmmo();
        UpdateCarriedAmmo();
        PlayEquipWeaponSound();
        ReloadEmptyWeapon();
	    EquippedWeapon->EnableCustomDepth(false);
    }
    void UCombatComponent::EquipSecondaryWeapon(AWeapon* WeaponToEquip)
    {
        if (WeaponToEquip == nullptr) return;
        SecondaryWeapon = WeaponToEquip;
        SecondaryWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
        AttachActorToBackpack(WeaponToEquip);
        PlayEquipWeaponSound(WeaponToEquip);
        if (EquippedWeapon == nullptr) return;
        EquippedWeapon->SetOwner(Character);
    }
    void UCombatComponent::AttachActorToLeftHand(AActor* ActorToAttach)
    {
        if (Character == nullptr || Character->GetMesh() == nullptr || ActorToAttach == nullptr) return;
        const USkeletalMeshSocket* BackpackSocket = Character->GetMesh()->GetSocketByName(FName("BackpackSocket"));
        if (BackpackSocket)
        {
            BackpackSocket->AttachActor(ActorToAttach, Character->GetMesh());
        }
    }
    void UCombatComponent::PlayEquipWeaponSound(AWeapon* WeaponTpEquip)
    {
        if (Character && WeaponTpEquip && WeaponTpEquip->EquipSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                this,
                WeaponTpEquip->EquipSound,
                Character->GetActorLocation()
            );
        }
    }
    void UCombatComponent::OnRep_EquippedWeapon()
    {
        if (EquippedWeapon && Character)
        {
            EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
            AttachActorToRightHand(EquippedWeapon);
            Character->GetCharacterMovement()->bOrientRotationToMovement = false;
            Character->bUseControllerRotationYaw = true;
            PlayEquipWeaponSound(EquippedWeapon);
	    EquippedWeapon->EnableCustomDepth(false);
        }
    }

    void UCombatComponent::OnRep_SecondaryWeapon()
    {
        if (SecondaryWeapon && Character)
        {
            SecondaryWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
            AttachActorToBackpack(SecondaryWeapon);
            PlayEquipWeaponSound(SecondaryWeapon);
        }
    }
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::OnRep_Owner()
    {
        Super::OnRep_Owner();
        if (Owner == nullptr)
        {
            ...
        }
        else
        {
            BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(Owner) : BlasterOwnerCharacter;
            if (BlasterOwnerCharacter && BlasterOwnerCharacter->GetEquippedWeapon() && BlasterOwnerCharacter->GetEquippedWeapon() == this)
            {
                SetHUDAmmo();
            }
        }
    }
    ```

## Swap Weapon
- Weapon.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponState :uint8
    {
        EWS_Initial UMETA(Displayname = "Initial State"),
        EWS_Equipped UMETA(Displayname = "Equipped"),
	    EWS_EquippedSecondary UMETA(Displayname = "EquippedSecondary"),
        EWS_Dropped UMETA(Displayname = "Dropped"),
        EWS_MAX UMETA(Displayname = "DefaultMAX"),
    };
    ...
    protected:
    	virtual void OnWeaponStateSet();
        virtual void OnEquipped();
        virtual void OnDropped();
	    virtual void OnEquippedSecondary();
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::SetWeaponState(EWeaponState State)
    {
        WeaponState = State;
        OnWeaponStateSet();
    }

    void AWeapon::OnRep_WeaponState()
    {
        OnWeaponStateSet();
    }

    void AWeapon::OnWeaponStateSet()
    {
        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            OnEquipped();
            break;
        case EWeaponState::EWS_EquippedSecondary:
            OnEquippedSecondary();
            break;
        case EWeaponState::EWS_Dropped:
            OnDropped();
            break;
        }
    }

    void AWeapon::OnEquipped()
    {
        ShowPickupWidget(false);
        AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        WeaponMesh->SetSimulatePhysics(false);
        WeaponMesh->SetEnableGravity(false);
        WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        if (WeaponType == EWeaponType::EWT_SubmachineGun)
        {
            WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
            WeaponMesh->SetEnableGravity(true);
            WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
        }
    }

    void AWeapon::OnDropped()
    {
        if (HasAuthority())
        {
            AreaSphere->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
        }
        WeaponMesh->SetSimulatePhysics(true);
        WeaponMesh->SetEnableGravity(true);
        WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);

        WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
        WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
        WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
        WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
        WeaponMesh->MarkRenderStateDirty();
        EnableCustomDepth(true);
    }
    void AWeapon::OnEquippedSecondary()
    {
        ShowPickupWidget(false);
        AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        WeaponMesh->SetSimulatePhysics(false);
        WeaponMesh->SetEnableGravity(false);
        WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        if (WeaponType == EWeaponType::EWT_SubmachineGun)
        {
            WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
            WeaponMesh->SetEnableGravity(true);
            WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
        }
		WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_TAN);
		WeaponMesh->MarkRenderStateDirty();
    }
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::EquipSecondaryWeapon(AWeapon* WeaponToEquip)
    {
        ...
        SecondaryWeapon->SetWeaponState(EWeaponState::EWS_EquippedSecondary);
        ...
    }
    void UCombatComponent::OnRep_SecondaryWeapon()
    {
        if (SecondaryWeapon && Character)
        {
            SecondaryWeapon->SetWeaponState(EWeaponState::EWS_EquippedSecondary);
            ...
        }
    }
    ```

- 바꾸

- CombatComponent.h
    ``` C++
    public:
        void SwapWeapon();
	    bool ShouldSwapWeapon();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::SwapWeapon()
    {
	    if (CombatState != ECombatState::ECS_Unoccupied) return;    
        AWeapon* TempWeapon = EquippedWeapon;
        EquippedWeapon = SecondaryWeapon;
        SecondaryWeapon = TempWeapon;

        EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
        AttachActorToRightHand(EquippedWeapon);
        EquippedWeapon->SetHUDAmmo();
        UpdateCarriedAmmo();
        PlayEquipWeaponSound(EquippedWeapon);

        SecondaryWeapon->SetWeaponState(EWeaponState::EWS_EquippedSecondary);
        AttachActorToBackpack(SecondaryWeapon);
    }
    bool UCombatComponent::ShouldSwapWeapon()
    {
        return (EquippedWeapon != nullptr && SecondaryWeapon != nullptr);
    }
    void UCombatComponent::EquipSecondaryWeapon(AWeapon* WeaponToEquip)
    {
        if (WeaponToEquip == nullptr) return;
        SecondaryWeapon = WeaponToEquip;
        SecondaryWeapon->SetWeaponState(EWeaponState::EWS_EquippedSecondary);
        AttachActorToBackpack(WeaponToEquip);
        PlayEquipWeaponSound(WeaponToEquip);
        SecondaryWeapon->SetOwner(Character);
    }
    void UCombatComponent::OnRep_EquippedWeapon()
    {
        if (EquippedWeapon && Character)
        {
            ...
            EquippedWeapon->SetHUDAmmo();
        }
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::ServerEquipButtonPressed_Implementation()
    {
        if (Combat)
        {
            if (OverlappingWeapon)
            {
                Combat->EquipWeapon(OverlappingWeapon);
            }
            else if(Combat->ShouldSwapWeapon())
            {
                Combat->SwapWeapon();
            }
        }
    }
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::OnEquipped()
    {
        ...
        EnableCustomDepth(false);
    }
    void AWeapon::OnEquippedSecondary()
    {
        ...
        EnableCustomDepth(true);
    }
    ```

## 죽었을 때 Secondary Weapon 떨어뜨리기
- BlasterCharacter.h
    ``` C++
    protected:
        void DropOrDestroyWeapon(AWeapon* Weapon);
	    void DropOrDestroyWeapons();
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::Elim()
    {
	    DropOrDestroyWeapons();
        MulticastElim();
        GetWorldTimerManager().SetTimer(
            ElimTimer,
            this,
            &ABlasterCharacter::ElimTimerFinished,
            ElimDelay
        );
    }
    void ABlasterCharacter::DropOrDestroyWeapon(AWeapon* Weapon)
    {
        if (Weapon->bDestroyWeapon)
        {
            Weapon->Destroy();
        }
        else
        {
            Weapon->Dropped();
        }
    }
    void ABlasterCharacter::DropOrDestroyWeapons()
    {
        if (Combat)
        {
            if (Combat->EquippedWeapon)
            {
                DropOrDestroyWeapon(Combat->EquippedWeapon);
            }
            if (Combat->SecondaryWeapon)
            {
                DropOrDestroyWeapon(Combat->SecondaryWeapon);
            }
        }
    }
    ```