## Rocket Projectile
- Source > Weapon에 Projectile을 상속받는 ProjectileRocket 생성
- ProjectileRocket.h
    ``` C++
    public:
        AProjectileRocket();
    protected:
        virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit) override;
    private:
	    UPROPERTY(VisibleAnywhere)
		UStaticMeshComponent* RocketMesh;
    ```
- ProjectileRocket.cpp
    ``` C++
    #include "ProjectileRocket.h"
    #include "Kismet/GameplayStatics.h"

    AProjectileRocket::AProjectileRocket()
    {
        RocketMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Rocket Mesh"));
        RocketMesh->SetupAttachment(RootComponent);
        RocketMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }

    void AProjectileRocket::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        APawn* FiringPawn = GetInstigator();
        if (FiringPawn)
        {
            AController* FiringController = FiringPawn->GetController();
            if (FiringController)
            {
                UGameplayStatics::ApplyRadialDamageWithFalloff(
                    this,
                    Damage,
                    10.f,
                    GetActorLocation(),
                    200.f,
                    500.f,
                    1.f,
                    UDamageType::StaticClass(),
                    TArray<AActor*>(),
                    this,
				    FiringController
                );
            }
        }
        Super::OnHit(HitComp, OtherActor, OtherComp, NormalImpulse, Hit);
    }
    ```
- Blueprints > Weapon > Projectiles 폴더에 ProjectileRocket을 상속받는 BP_Rocket 생성
    - 적절한 메쉬와 이펙트, 데미지 설정
- Blueprints > Weapon 폴더에 ProjectileWeapon을 상속받는 BP_RocketLauncher 생성

- CombatComponent.h
    ``` C++
    private:
        UPROPERTY(EditDefaultsOnly)
		int32 StartingARAmmo = 30;
	    UPROPERTY(EditDefaultsOnly)
		int32 StartingRocketAmmo = 30;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::InitializeCarriedAmmo()
    {
        CarriedAmmoMap.Emplace(EWeaponType::EWT_AssaultRifle, StartingARAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_RocketLauncher, StartingRocketAmmo);
    }
    ```
- WeaponTypes.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        EWT_AssaultRifle UMETA(DisplayName = "AssaultRifle"),
        EWT_RocketLauncher UMETA(DisplayName = "RocketLauncher"),
        EWT_MAX UMETA(DisplayName = "DefaultMax")
    };
    ```

## Rocket Trails
- 나이아가라를 사용해서 이펙트를 만들것임
- DepthFade 노드는 벽이나 천장같은곳에 가까워질수록 Fade되는 효과를 줄 것임
- 우클릭 > FX > Niagara Emitter > New Emitter from a template > Empty > NE_TrailSmoke 생성
    - Render 속성에 SpriteRenderer적용 > MI_TrailSmoke로 설정
        - Sub UV속성에 Sub Image Size를 8,8로 설정
    - Emitter Update 속성에 Spawn Rate 적용
    - Particle Spawn 속성에 Set new or existing parameter directly 적용 > SubImageIndex
        - 이거를 랜덤으로 정해서 64개의 이미지 중 어떤 것을 spawn할 지 정할 수 있다.
        - Initialize Particle 속성에 Sprite Size Mode를 Random Uniform으로 설정, lifetime도, rotation도..
    - Particle Update 속성에 
        - Scale Sprite Size 설정
            - Vector 2DFrom Float > Float From Curve > Curve for floats
        - Scale Color도 설정 
- 우클릭 > FX > Niagara System > Create Empty System > NS_TrailSmoke 생성
    - NE_TrailSmoke의 노드를 복붙
    - 하나 더 복붙해서 Firie를 만듦(색깔 좀 바꾸기)


- Blaster.Build.cs
    ``` C++
	PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "Niagara" });
    ```
- ProjectileRocket.h
    ``` C++
    public:
    	virtual void Destroyed() override;
    protected:
        virtual void BeginPlay() override;
	    void DestroyTimerFinished();
        UPROPERTY(EditAnywhere)
        class UNiagaraSystem* TrailSystem;
	    UPROPERTY()
		class UNiagaraComponent* TrailSystemComponent;
    private:
        FTimerHandle DestroyTimer;
        UPROPERTY(EditAnywhere)
            float DestroyTime = 3.f;
    ```
- ProjectileRocket.h
    ``` C++
    #include "NiagaraFunctionLibrary.h"
    #include "NiagaraComponent.h"
    #include "Sound/SoundCue.h"
    #include "Components/BoxComponent.h"
    void AProjectileRocket::BeginPlay()
    {
        Super::BeginPlay();
        if (!HasAuthority())
        {
            CollisionBox->OnComponentHit.AddDynamic(this, &AProjectileRocket::OnHit);
        }
        if (TrailSystem)
        {
            TrailSystemComponent = UNiagaraFunctionLibrary::SpawnSystemAttached(
                TrailSystem,
                GetRootComponent(),
                FName(),
                GetActorLocation(),
                GetActorRotation(),
                EAttachLocation::KeepWorldPosition,
                false
            );
        }
    }
    void AProjectileRocket::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        ...
	    if (FiringPawn && HasAuthority())
        {
            ...
        }
        //Super::OnHit(HitComp, OtherActor, OtherComp, NormalImpulse, Hit);
        GetWorldTimerManager().SetTimer(
            DestroyTimer,
            this,
            &AProjectileRocket::DestroyTimerFinished,
            DestroyTime
        );
        if (ImpactParticles)
        {
            UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ImpactParticles, GetActorTransform());
        }
        if (ImpactSound)
        {
            UGameplayStatics::PlaySoundAtLocation(this, ImpactSound, GetActorLocation());
        }
        if (RocketMesh)
        {
            RocketMesh->SetVisibility(false);
        }
        if (CollisionBox)
        {
            CollisionBox->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        }
        if (TrailSystemComponent)
        {
            TrailSystemComponent->GetSystemInstance()->Deactivate();
        }
    }
    void AProjectileRocket::DestroyTimerFinished()
    {
        Destroy();
    }
    void AProjectileRocket::Destroyed()
    {
    }
    ```
- Projectile.h
    ``` C++
    protected:
    	UPROPERTY(EditAnywhere)
		class UParticleSystem* ImpactParticles;
	    UPROPERTY(EditAnywhere)
		class USoundCue* ImpactSound;
        UPROPERTY(EditAnywhere)
        class UBoxComponent* CollisionBox;
    ```
    - private에 있던거 이동
- BP_Rocket에서 나이아가라시스템 NS_TrailSmoke 추가

- 소리 추가하기
- ProjectileRocket.h
    ``` C++
    protected:
    	UPROPERTY(EditAnywhere)
		USoundCue* ProjectileLoop;
	    UPROPERTY()
        UAudioComponent* ProjectileLoopComponent;
        UPROPERTY(EditAnywhere)
        USoundAttenuation* LoopingSoundAttenuation;
    ```
- ProjectileRocket.cpp
    ``` C++
    #include "Components/AudioComponent.h"
        
    void AProjectileRocket::BeginPlay()
    {
        ...
        if (ProjectileLoop && LoopingSoundAttenuation)
        {
            ProjectileLoopComponent = UGameplayStatics::SpawnSoundAttached(
                ProjectileLoop,
                GetRootComponent(),
                FName(),
                GetActorLocation(),
                EAttachLocation::KeepWorldPosition,
                false,
                1.f,
                1.f,
                0.f,
                LoopingSoundAttenuation,
                (USoundConcurrency*)nullptr,
                false
            );
        }
    }

    void AProjectileRocket::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        ...
        if (ProjectileLoopComponent && ProjectileLoopComponent->IsPlaying()) {
            ProjectileLoopComponent->Stop();
        }
    }
    ```
- BP_Rocket에 알맞은 sound 애셋 적용

## Rocket MovementComponent
- 지금은 Projectile이 발사한 사람을 맞추기도 한다.
- Source > Blaster > Weapon 폴더에 ProjectileMovementComponent를 상속받는 RocketMovementComponent생성

- RocketMovementComponent.h
    ``` C++
    protected:
        virtual EHandleBlockingHitResult HandleBlockingHit(const FHitResult& Hit, float TimeTick, const FVector& MoveDelta, float& SubTickTimeRemaining) override;
        virtual void HandleImpact(const FHitResult& Hit, float TimeSlice = 0.f, const FVector& MoveDelta = FVector::ZeroVector) override;
    ```
- RocketMovementComponent.cpp
    ``` C++
    URocketMovementComponent::EHandleBlockingHitResult URocketMovementComponent::HandleBlockingHit(const FHitResult& Hit, float TimeTick, const FVector& MoveDelta, float& SubTickTimeRemaining)
    {
        Super::HandleBlockingHit(Hit, TimeTick, MoveDelta, SubTickTimeRemaining);
        return EHandleBlockingHitResult::AdvanceNextSubstep;
    }
    void URocketMovementComponent::HandleImpact(const FHitResult& Hit, float TimeSlice, const FVector& MoveDelta)
    {
        // Rockets should not stop; only explode when their collisionBox detects a hit
    }
    ```
    - OnHit에서 Owner면 return하라고 했음에도, 충돌은 했기에 멈춰있는 현상을 막기 위함
- Projectile.h
    ``` C++
    protected:  
    	UPROPERTY(VisibleAnywhere)
		class UProjectileMovementComponent* ProjectileMovementComponent;
    ```
    - private에서 protected로
- Projectile.cpp
    ``` C++
    //#include "GameFramework/ProjectileMovementComponent.h"
    AProjectile::AProjectile()
    {
        ...
        //ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovementComponent"));
        //ProjectileMovementComponent->bRotationFollowsVelocity = true;
        //ProjectileMovementComponent->InitialSpeed = 15000;
        //ProjectileMovementComponent->MaxSpeed = 15000;
    }
    ```
    - Projectile이 아니라 ProjectileBullet에서 할거라함(로켓이랑 따로 만들거기떄문)
- ProjectileBullet.h
    ``` C++
    public:
        AProjectileBullet();
    ```
- ProjectileBullet.cpp
    ``` C++
    #include "GameFramework/ProjectileMovementComponent.h"

    AProjectileBullet::AProjectileBullet()
    {
        ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovementComponent"));
        ProjectileMovementComponent->bRotationFollowsVelocity = true;
        ProjectileMovementComponent->InitialSpeed = 15000;
        ProjectileMovementComponent->MaxSpeed = 15000;
	    ProjectileMovementComponent->SetIsReplicated(true);
    }
    ```
- ProjectileRocket.h
    ``` C++
    protected:
        UPROPERTY(VisibleAnywhere)
	    class URocketMovementComponent* RocketMovementComponent;
    ```
- ProjectileRocket.cpp
    ``` C++
    #include "RocketMovementComponent.h"
    AProjectileRocket::AProjectileRocket()
    {
        RocketMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Rocket Mesh"));
        RocketMesh->SetupAttachment(RootComponent);
        RocketMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

        RocketMovementComponent = CreateDefaultSubobject<URocketMovementComponent>(TEXT("RocketMovementComponent"));
        RocketMovementComponent->bRotationFollowsVelocity = true;
        RocketMovementComponent->InitialSpeed = 15000;
        RocketMovementComponent->MaxSpeed = 15000;
        RocketMovementComponent->SetIsReplicated(true);
    }
    void AProjectileRocket::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        if (OtherActor == GetOwner())
        {
            return;
        }
        ...
    }
    ```

- 무기가 제대로 동기화가 안 되는 문제
    - Weapon.cpp
        ``` C++
        AWeapon::AWeapon()
        {
        	SetReplicateMovement(true);
            ...
        }
        ```
    - BP에서도 제대로 되어있는지 확인하자

## HitScan Weapon
- Sourcce > Blaster > Weapon 폴더에 Weapon상속받는 HitScanWeapon생성

- HitScanWeapon.h
    ``` C++
    public:
        virtual void Fire(const FVector& HitTarget) override;
    private:
        UPROPERTY(EditAnywhere)
        float Damage = 20.f;
        UPROPERTY(EditAnywhere)
        class UParticleSystem* ImpactParticles;
    ```
- HitScanWewapon.cpp
    ``` C++
    #include "HitScanWeapon.h"
    #include "Engine/SkeletalMeshSocket.h"
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Kismet/GameplayStatics.h"
    void AHitScanWeapon::Fire(const FVector& HitTarget)
    {
        Super::Fire(HitTarget);

        APawn* OwnerPawn = Cast<APawn>(GetOwner());
        if (OwnerPawn == nullptr) return;
        AController* InstigatorController = OwnerPawn->GetController();

        const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
        if (MuzzleFlashSocket && InstigatorController)
        {
            FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
            FVector Start = SocketTransform.GetLocation();
            FVector End = Start + (HitTarget - Start) * 1.25f;

            FHitResult FireHit;
            UWorld* World = GetWorld();
            if (World)
            {
                World->LineTraceSingleByChannel(
                    FireHit,
                    Start,
                    End,
                    ECollisionChannel::ECC_Visibility
                );
                if (FireHit.bBlockingHit)
                {
                    ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FireHit.GetActor());
                    if (BlasterCharacter)
                    {
                        if (HasAuthority())
                        {
                            UGameplayStatics::ApplyDamage(
                                BlasterCharacter,
                                Damage,
                                InstigatorController,
                                this,
                                UDamageType::StaticClass()
                            );
                        }
                    }
                    if (ImpactParticles)
                    {
                        UGameplayStatics::SpawnEmitterAtLocation(
                            World,
                            ImpactParticles,
						    FireHit.ImpactPoint,
                            FireHit.ImpactNormal.Rotation()
                        );
                    }
                }
            }
        }
    }
    ```
- Blueprints > Weapon 폴더에 HitScanWeapon을 상속받는 BP_Pistol 생성
- Blueprints > Weapon > Casing 폴더에 Casing 상속받는 BP_PistolCasing 생성

- WeaponTypes.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        EWT_AssaultRifle UMETA(DisplayName = "AssaultRifle"),
        EWT_RocketLauncher UMETA(DisplayName = "RocketLauncher"),
        EWT_Pistol UMETA(DisplayName = "Pistol"),
        EWT_MAX UMETA(DisplayName = "DefaultMax")
    };
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PlayReloadMontage()
    {
        if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && ReloadMontage)
        {
            AnimInstance->Montage_Play(ReloadMontage);
            FName SectionName;
            switch (Combat->EquippedWeapon->GetWeaponType())
            {
            case EWeaponType::EWT_AssaultRifle:
                SectionName = FName("Rifle");
                break;
            case EWeaponType::EWT_RocketLauncher:
                SectionName = FName("Rifle");
                break;
            case EWeaponType::EWT_Pistol:
                SectionName = FName("Rifle");
                break;
            }
            AnimInstance->Montage_JumpToSection(SectionName);
        }
    }
    ```
- CombatComponent.h
    ``` C++
    private:
    	UPROPERTY(EditDefaultsOnly)
		int32 StartingPistolAmmo = 15;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::InitializeCarriedAmmo()
    {
        CarriedAmmoMap.Emplace(EWeaponType::EWT_AssaultRifle, StartingARAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_RocketLauncher, StartingRocketAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_Pistol, StartingPistolAmmo);
    }
    ```

## Beam Effect
- Panner노드를 사용하면 텍스쳐가 이동하는것 같은 느낌 줄 수 있음
- HitScanWeapon.h
    ``` C++
    private:
        UPROPERTY(EditAnywhere)
        UParticleSystem* BeamParticles;
    ```
- HitScanWeapon.cpp
    ``` C++
    #include "particles/ParticleSystemComponent.h"

    void AHitScanWeapon::Fire(const FVector& HitTarget)
    {
        ...
        if (MuzzleFlashSocket)
        {
            ...
            if (World)
            {
                ...
                FVector BeamEnd = End;
                if (FireHit.bBlockingHit)
                {
                    BeamEnd = FireHit.ImpactPoint;
                    ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FireHit.GetActor());
                    if (BlasterCharacter && HasAuthority() && InstigatorController)
                    {
                        UGameplayStatics::ApplyDamage(
                            BlasterCharacter,
                            Damage,
                            InstigatorController,
                            this,
                            UDamageType::StaticClass()
                        );
                    }
                    ...
                }
                if (BeamParticles)
                {
                    UParticleSystemComponent* Beam = UGameplayStatics::SpawnEmitterAtLocation(
                        World,
                        BeamParticles,
                        SocketTransform
                    );
                    if (Beam)
                    {
                        Beam->SetVectorParameter(FName("Target"), BeamEnd);
                    }
                }
            }
        }
    }
    ```
- BP_Pistol의 BealParticles에 P_SmokeTrail 연결

## Submachine gun
- Blueprints > Weapon 폴더에 HitScanWeapon을 상속받는 BP_SubmachineGun 생성
- WeaponTypes.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        EWT_AssaultRifle UMETA(DisplayName = "AssaultRifle"),
        EWT_RocketLauncher UMETA(DisplayName = "RocketLauncher"),
        EWT_Pistol UMETA(DisplayName = "Pistol"),
        EWT_SubmachineGun UMETA(DisplayName = "SubmachineGun"),
        EWT_MAX UMETA(DisplayName = "DefaultMax")
    };
    ```
- CombatComponent.h
    ``` C++
    private:
        UPROPERTY(EditDefaultsOnly)
		int32 StartingSMGAmmo = 20;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::InitializeCarriedAmmo()
    {
        CarriedAmmoMap.Emplace(EWeaponType::EWT_AssaultRifle, StartingARAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_RocketLauncher, StartingRocketAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_Pistol, StartingPistolAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_SubmachineGun, StartingSMGAmmo);
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PlayReloadMontage()
    {
        ...
        if (AnimInstance && ReloadMontage)
        {
            ...
            switch (Combat->EquippedWeapon->GetWeaponType())
            {
            ...
            case EWeaponType::EWT_SubmachineGun:
                SectionName = FName("Rifle");
                break;
            }
            ...
        }
    }
    ```
- 사운드입히기
- HitScanWeapon.h
    ``` C++
    private:
        UPROPERTY(EditAnywhere)
        UParticleSystem* MuzzleFlash;
        UPROPERTY(EditAnywhere)
        USoundCue* FireSound;
        UPROPERTY(EditAnywhere)
        USoundCue* HitSound;
    ```
- HitScanWeapon.cpp
    ``` C++
    #include "Sound/SoundCue.h"

    void AHitScanWeapon::Fire(const FVector& HitTarget)
    {
        ...
        if (MuzzleFlashSocket)
        {
            ...
            if (World)
            {
                ...
                if (FireHit.bBlockingHit)
                {
                    ...
                    if (HitSound)
                    {
                        UGameplayStatics::PlaySoundAtLocation(
                            this,
                            HitSound,
                            FireHit.ImpactPoint
                        );
                    }
                }
                ...
            }
            if (MuzzleFlash)
            {
                UGameplayStatics::SpawnEmitterAtLocation(
                    World,
                    MuzzleFlash,
                    SocketTransform
                );
            }
            if (FireSound)
            {
                UGameplayStatics::PlaySoundAtLocation(
                    this,
                    FireSound,
                    GetActorLocation()
                );
            }
        }
    }

    ```

- Weapon.cpp
    ``` C++
    void AWeapon::OnRep_WeaponState()
    {
        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ShowPickupWidget(false);
            WeaponMesh->SetSimulatePhysics(false);
            WeaponMesh->SetEnableGravity(false);
            WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
            if (WeaponType == EWeaponType::EWT_SubmachineGun)
            {
                WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
                WeaponMesh->SetEnableGravity(true);
                WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
		        WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
            }
            break;
        case EWeaponState::EWS_Dropped:
            ...
            WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
            WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
		    WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
            break;
        }
    }
    void AWeapon::SetWeaponState(EWeaponState State)
    {
        WeaponState = State;

        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ...
            if (WeaponType == EWeaponType::EWT_SubmachineGun)
            {
                WeaponMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
                WeaponMesh->SetEnableGravity(true);
                WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
            }
            break;
        case EWeaponState::EWS_Dropped:
            ...
            WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
            WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
            break;
    }
    ```

## Shotgun
- Source > Blaster > Weapon 폴더에 HitScanWeapon을 상속받는 Shotgun 생성
- Shotgun.h
    ``` C++
    public:
        virtual void Fire(const FVector& HitTarget) override;
    private:
	    UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        uint32 NumberOfPellets = 10;
    ```
- Shotgun.cpp
    ``` C++
    #include "Shotgun.h"
    #include "Engine/SkeletalMeshSocket.h"
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Kismet/GameplayStatics.h"
    #include "particles/ParticleSystemComponent.h"
    #include "Sound/SoundCue.h"
    void AShotgun::Fire(const FVector& HitTarget)
    {
	    AWeapon::Fire(HitTarget);

        APawn* OwnerPawn = Cast<APawn>(GetOwner());
        if (OwnerPawn == nullptr) return;
        AController* InstigatorController = OwnerPawn->GetController();

        const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
        if (MuzzleFlashSocket)
        {
            FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
            FVector Start = SocketTransform.GetLocation();
            FVector End = Start + (HitTarget - Start) * 1.25f;

            TraceEndWithScatter(Start, HitTarget);
        }
    }
    ```
- HitScanWeapon.h
    ``` C++
    protected:
        FVector TraceEndWithScatter(const FVector& TraceStart, const FVector& HitTarget);
    private:
        // Trace End With Scatter
        UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        float DistanceToSphere = 800.f;
        UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        float SphereRadius = 75.f;
        UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        bool bUseScatter = false;
    ```
- HitScanWeapon.cpp
    ``` C++
    #include "DrawDebugHelpers.h"
    #include "Kismet/KismetMathLibrary.h"
    FVector AHitScanWeapon::TraceEndWithScatter(const FVector& TraceStart, const FVector& HitTarget)
    {
        FVector ToTargetNormalized = (HitTarget - TraceStart).GetSafeNormal();
        FVector SphereCenter = TraceStart + ToTargetNormalized * DistanceToSphere;
        DrawDebugSphere(GetWorld(), SphereCenter, SphereRadius, 12, FColor::Red, true);
	    return FVector::ZeroVector;
    }
    ```

- Blueprints > Weapon 폴더에 Shotgun을 상속받는 BP_Shotgun 생성
- Blueprints > Weapon > Casings 폴더에 Casing 상속받는 BP_ShotgunCasing생성

- WeaponTypes.h
    ``` C++
    #define TRACE_LENGTH 80000
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        EWT_AssaultRifle UMETA(DisplayName = "AssaultRifle"),
        EWT_RocketLauncher UMETA(DisplayName = "RocketLauncher"),
        EWT_Pistol UMETA(DisplayName = "Pistol"),
        EWT_SubmachineGun UMETA(DisplayName = "SubmachineGun"),
        EWT_Shotgun UMETA(DisplayName = "Shotgun"),
        EWT_MAX UMETA(DisplayName = "DefaultMax")
    };
    ```
- CombatComponent.h
    ``` C++
    //#define TRACE_LENGTH 80000
    ...
    private:
        UPROPERTY(EditDefaultsOnly)
		int32 StartingShotgunAmmo = 20;
    ```
- HitScanWeapon.cpp
    ``` C++
    #include "WeaponTypes.h"

    FVector AHitScanWeapon::TraceEndWithScatter(const FVector& TraceStart, const FVector& HitTarget)
    {
        FVector ToTargetNormalized = (HitTarget - TraceStart).GetSafeNormal();
        FVector SphereCenter = TraceStart + ToTargetNormalized * DistanceToSphere;
        FVector RandVec = UKismetMathLibrary::RandomUnitVector() * FMath::FRandRange(0.f, SphereRadius);
        FVector EndLoc = SphereCenter + RandVec;
        FVector ToEndLoc = EndLoc - TraceStart;

        DrawDebugSphere(GetWorld(), SphereCenter, SphereRadius, 12, FColor::Red, true);
        DrawDebugSphere(GetWorld(), EndLoc, 4.0f, 12, FColor::Orange, true);
        DrawDebugLine(GetWorld(), TraceStart, FVector(TraceStart + ToEndLoc * TRACE_LENGTH / ToEndLoc.Size()), FColor::Cyan, true);
        return FVector(TraceStart + ToEndLoc * TRACE_LENGTH / ToEndLoc.Size());
    }
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::InitializeCarriedAmmo()
    {
        CarriedAmmoMap.Emplace(EWeaponType::EWT_AssaultRifle, StartingARAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_RocketLauncher, StartingRocketAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_Pistol, StartingPistolAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_SubmachineGun, StartingSMGAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_Shotgun, StartingShotgunAmmo);
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PlayReloadMontage()
    {
        ...
        if (AnimInstance && ReloadMontage)
        {
            ...
            switch (Combat->EquippedWeapon->GetWeaponType())
            {
            ...
            case EWeaponType::EWT_Shotgun:
                SectionName = FName("Rifle");
                break;
            }
            ...
        }
    }
    ```

## WeaponScatter
- HitScanWeapon.h
    ``` C++
    protected:
	    void WeaponTraceHit(const FVector& TraceStart, const FVector& HitTarget, FHitResult& OutHit);
        
        UPROPERTY(EditAnywhere)
        class UParticleSystem* ImpactParticles;
        UPROPERTY(EditAnywhere)
        USoundCue* HitSound;
        UPROPERTY(EditAnywhere)
        float Damage = 20.f;
    ```
- HitScanWeapon.cpp
    ``` C++
    void AHitScanWeapon::Fire(const FVector& HitTarget)
    {
        Super::Fire(HitTarget);

        APawn* OwnerPawn = Cast<APawn>(GetOwner());
        if (OwnerPawn == nullptr) return;
        AController* InstigatorController = OwnerPawn->GetController();

        const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
        if (MuzzleFlashSocket)
        {
            FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
            FVector Start = SocketTransform.GetLocation();

            FHitResult FireHit;
            WeaponTraceHit(Start, HitTarget, FireHit);

            ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FireHit.GetActor());
            if (BlasterCharacter && HasAuthority() && InstigatorController)
            {
                UGameplayStatics::ApplyDamage(
                    BlasterCharacter,
                    Damage,
                    InstigatorController,
                    this,
                    UDamageType::StaticClass()
                );
            }
            if (ImpactParticles)
            {
                UGameplayStatics::SpawnEmitterAtLocation(
                    GetWorld(),
                    ImpactParticles,
                    FireHit.ImpactPoint,
                    FireHit.ImpactNormal.Rotation()
                );
            }
            if (HitSound)
            {
                UGameplayStatics::PlaySoundAtLocation(
                    this,
                    HitSound,
                    FireHit.ImpactPoint
                );
            }
            if (MuzzleFlash)
            {
                UGameplayStatics::SpawnEmitterAtLocation(
                    GetWorld(),
                    MuzzleFlash,
                    SocketTransform
                );
            }
            if (FireSound)
            {
                UGameplayStatics::PlaySoundAtLocation(
                    this,
                    FireSound,
                    GetActorLocation()
                );
            }
        }
    }
    void AHitScanWeapon::WeaponTraceHit(const FVector& TraceStart, const FVector& HitTarget, FHitResult& OutHit)
    {
        UWorld* World = GetWorld();
        if (World)
        {
            FVector End = bUseScatter ? TraceEndWithScatter(TraceStart, HitTarget) : TraceStart + (HitTarget - TraceStart) * 1.25f;
            World->LineTraceSingleByChannel(
                OutHit,
                TraceStart,
                End,
                ECollisionChannel::ECC_Visibility
            );
            FVector BeamEnd = End;
            if (OutHit.bBlockingHit)
            {
                BeamEnd = OutHit.ImpactPoint;
            }
            if (BeamParticles)
            {
                UParticleSystemComponent* Beam = UGameplayStatics::SpawnEmitterAtLocation(
                    World,
                    BeamParticles,
                    TraceStart,
                    FRotator::ZeroRotator,
                    true
                );
                if (Beam)
                {
                    Beam->SetVectorParameter(FName("Target"), BeamEnd);
                }
            }
        }
    }
    ```
- Shotgun.cpp
    ``` C++
    void AShotgun::Fire(const FVector& HitTarget)
    {
        AWeapon::Fire(HitTarget);

        APawn* OwnerPawn = Cast<APawn>(GetOwner());
        if (OwnerPawn == nullptr) return;
        AController* InstigatorController = OwnerPawn->GetController();

        const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
        if (MuzzleFlashSocket)
        {
            FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
            FVector Start = SocketTransform.GetLocation();
            TMap<ABlasterCharacter*, uint32> HitMap;
            for (uint32 i = 0; i < NumberOfPellets; i++)
            {
                FHitResult FireHit;
                WeaponTraceHit(Start, HitTarget, FireHit);

                ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(FireHit.GetActor());
                if (BlasterCharacter && HasAuthority() && InstigatorController)
                {
                    if (HitMap.Contains(BlasterCharacter))
                    {
                        HitMap[BlasterCharacter]++;
                    }
                    else
                    {
                        HitMap.Emplace(BlasterCharacter, 1);
                    }
                }
                if (ImpactParticles)
                {
                    UGameplayStatics::SpawnEmitterAtLocation(
                        GetWorld(),
                        ImpactParticles,
                        FireHit.ImpactPoint,
                        FireHit.ImpactNormal.Rotation()
                    );
                }
                if (HitSound)
                {
                    UGameplayStatics::PlaySoundAtLocation(
                        this,
                        HitSound,
                        FireHit.ImpactPoint,
                        .5f,
                        FMath::FRandRange(-.5f, .5f)
                    );
                }
            }
            for (auto HitPair : HitMap)
            {
                if (HitPair.Key && HasAuthority() && InstigatorController)
                {
                    UGameplayStatics::ApplyDamage(
                        HitPair.Key,
                        Damage * HitPair.Value,
                        InstigatorController,
                        this,
                        UDamageType::StaticClass()
                    );
                }
            }
        }
    }
    ```

## Sniper Rifle
- WeaponTypes.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        EWT_AssaultRifle UMETA(DisplayName = "AssaultRifle"),
        EWT_RocketLauncher UMETA(DisplayName = "RocketLauncher"),
        EWT_Pistol UMETA(DisplayName = "Pistol"),
        EWT_SubmachineGun UMETA(DisplayName = "SubmachineGun"),
        EWT_Shotgun UMETA(DisplayName = "Shotgun"),
        EWT_SniperRifle UMETA(DisplayName = "SniperRifle"),
        EWT_MAX UMETA(DisplayName = "DefaultMax")
    };
    ```
- CombatComponent.h
    ``` C++
    private:    
        UPROPERTY(EditDefaultsOnly)
		int32 StartingSniperAmmo = 1;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::InitializeCarriedAmmo()
    {
        CarriedAmmoMap.Emplace(EWeaponType::EWT_AssaultRifle, StartingARAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_RocketLauncher, StartingRocketAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_Pistol, StartingPistolAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_SubmachineGun, StartingSMGAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_Shotgun, StartingShotgunAmmo);
        CarriedAmmoMap.Emplace(EWeaponType::EWT_SniperRifle, StartingSniperAmmo);
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PlayReloadMontage()
    {
        ...
        if (AnimInstance && ReloadMontage)
        {
            ...
            case EWeaponType::EWT_SniperRifle:
                SectionName = FName("Rifle");
                break;
            }
            ...
        }
    }
    ```
- Blueprints > Weapon에 HitScanWeapon을 상속받는 BP_SniperRifle 생성

- Blueprints > HUD 폴더에 UserWidget을 상속받는 WBP_SniperScope 생성
    - Image로 ScopeOverlay, Background 생성
    - ScopeZoomIn 애니메이션 생성

- BlasterCharacter.h
    ``` C++
    public:
        UFUNCTION(BlueprintImplementableEvent)
        void ShowSniperScopeWidget(bool bShowScope);
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::SetAiming(bool bIsAiming)
    {
        if (Character == nullptr || EquippedWeapon == nullptr) return;
        ...
        if (Character->IsLocallyControlled() && EquippedWeapon->GetWeaponType() == EWeaponType::EWT_SniperRifle)
        {
            Character->ShowSniperScopeWidget(bIsAiming);
        }
    }
    ```
- BP_BlasterCharacter에서 ShowSniperScopeWidget을 Implement해서
    - SniperScopeWidget이 Valid하지 않다면 WBPSniperScope를 생성하고 SniperScopeWidget 변수에 저장
    - Valid체크하고 Scope Zoom In 애니메이션을 Play / ReversePlay
    - PlaySound2D
- 줌 한 채로 죽었을 때 처리하기
    - BlasterCharacter.cpp
        ``` C++
        void ABlasterCharacter::MulticastElim_Implementation()
        {
            ...
            bool bHideSniperScope = IsLocallyControlled() && Combat && Combat->bAiming && Combat->EquippedWeapon && Combat->EquippedWeapon->GetWeaponType() == EWeaponType::EWT_SniperRifle;
            if (bHideSniperScope)
            {
                ShowSniperScopeWidget(false);
            }
        }
        ```

## Grenade Launcher
- WeaponTypes.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        ...
        EWT_GrenadeLauncher UMETA(DisplayName = "GrenadeLauncher"),
        EWT_MAX UMETA(DisplayName = "DefaultMax")
    };
    ```
- CombatComponent.h
    ``` C++
    private:
        UPROPERTY(EditDefaultsOnly)
		int32 StartingGrenadeLauncherAmmo = 1;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::InitializeCarriedAmmo()
    {
        ...
        CarriedAmmoMap.Emplace(EWeaponType::EWT_GrenadeLauncher, StartingGrenadeLauncherAmmo);
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PlayReloadMontage()
    {
        ...
    case EWeaponType::EWT_GrenadeLauncher:
        SectionName = FName("Rifle");
        break;
       ...
    }
    ```
- Blueprints > Weapon 폴더에 ProjectileWeapon을 상속받는 BP_GrenadeLauncher생성


## Projectile Grenade
- 쓸 수 있는거 빼오기
- ProjectileRocket.h
    ``` C++
    protected:
        //UPROPERTY(EditAnywhere)
        //class UNiagaraSystem* TrailSystem;
        //UPROPERTY()
        //class UNiagaraComponent* TrailSystemComponent;

	    //void DestroyTimerFinished();
    private:
        //FTimerHandle DestroyTimer;
        //UPROPERTY(EditAnywhere)
        //float DestroyTime = 3.f;
        
	    //UPROPERTY(VisibleAnywhere)
		//UStaticMeshComponent* RocketMesh;
    ```
- Projectile.h
    ``` C++
    protected:
	    void StartDestroyTimer();
	    void DestroyTimerFinished();
        ...
    	UPROPERTY(EditAnywhere)
		class UNiagaraSystem* TrailSystem;
	    UPROPERTY()
		class UNiagaraComponent* TrailSystemComponent;
	    void SpawnTrailSystem();
        
        UPROPERTY(VisibleAnywhere)
        UStaticMeshComponent* ProjectileMesh;
    private:
        FTimerHandle DestroyTimer;
        UPROPERTY(EditAnywhere)
		float DestroyTime = 3.f;
    ```
- Projectile.cpp
    ``` C++
    #include "NiagaraFunctionLibrary.h"
    #include "NiagaraComponent.h"
    void AProjectile::SpawnTrailSystem()
    {
        if (TrailSystem)
        {
            TrailSystemComponent = UNiagaraFunctionLibrary::SpawnSystemAttached(
                TrailSystem,
                GetRootComponent(),
                FName(),
                GetActorLocation(),
                GetActorRotation(),
                EAttachLocation::KeepWorldPosition,
                false
            );
        }
    }
    void AProjectile::StartDestroyTimer()
    {
        GetWorldTimerManager().SetTimer(
            DestroyTimer,
            this,
            &AProjectile::DestroyTimerFinished,
            DestroyTime
        );
    }

    void AProjectile::DestroyTimerFinished()
    {
        Destroy();
    }
    ```
- ProjectileRocket.cpp
    ``` C++
    AProjectileRocket::AProjectileRocket()
    {
        ProjectileMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Rocket Mesh"));
        ProjectileMesh->SetupAttachment(RootComponent);
        ProjectileMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        ...
    }
    /*    
    void AProjectileRocket::DestroyTimerFinished()
    {
        Destroy();
    }
    */
    void AProjectileRocket::BeginPlay()
    {
        ...
        SpawnTrailSystem();
        ...
    }
    void AProjectileRocket::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        ...
        StartDestroyTimer();
        ...
        if (ProjectileMesh)
        {
            ProjectileMesh->SetVisibility(false);
        }
        ...
    }
    ```

- Source > Blaster > Weapon 폴더에 Projectile을 상속받은 ProjectileGrenade생성

- ProjectileGrenade.h
    ``` C++
    public:
        AProjectileGrenade();
    protected:
        virtual void BeginPlay() override;
        UFUNCTION()
        void OnBounce(const FHitResult& ImpactResult, const FVector& ImpactVelocity);
    private:
        UPROPERTY(EditAnywhere)
            USoundCue* BounceSound;
    ```
- ProjectileGrenade.cpp
    ``` C++
    #include "ProjectileGrenade.h"
    #include "GameFramework/ProjectileMovementComponent.h"
    #include "Kismet/GameplayStatics.h"
    #include "Sound/SoundCue.h"
    AProjectileGrenade::AProjectileGrenade()
    {
        ProjectileMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Grenade Mesh"));
        ProjectileMesh->SetupAttachment(RootComponent);
        ProjectileMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

        ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovementComponent"));
        ProjectileMovementComponent->bRotationFollowsVelocity = true;
        ProjectileMovementComponent->SetIsReplicated(true);
        ProjectileMovementComponent->bShouldBounce = true;
    }
    void AProjectileGrenade::BeginPlay()
    {
        AActor::BeginPlay();

        SpawnTrailSystem();
        StartDestroyTimer();

        ProjectileMovementComponent->OnProjectileBounce.AddDynamic(this, &AProjectileGrenade::OnBounce);
    }
    void AProjectileGrenade::OnBounce(const FHitResult& ImpactResult, const FVector& ImpactVelocity)
    {
        if (BounceSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                this,
                BounceSound,
                GetActorLocation()
            );
        }
    }
    ```

- Blueprints > Weapon > Projectiles에 BP_ProjectileGrenade생성
- NS_TrailSmoke 변형한 NS_TrailSmoke_Grenade생성

- BP_GrenadeLauncher에 적용

- 데미지 적용하기

- ProjectileGrenade.h
    ``` C++
    public:
    	virtual void Destroyed() override;
    ```
- ProjectileGrenade.cpp
    ``` C++
    void AProjectileRocket::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        ...
        ExplodeDamage();
        StartDestroyTimer();
        ...
    }
    ```
- Projectile.h
    ``` C++
    protected:
    	void ExplodeDamage();
	    UPROPERTY(EditAnywhere)
        float DamageInnerRadius = 200.f;
	    UPROPERTY(EditAnywhere)
        float DamageOuterRadius = 500.f;
    ```
- Projectile.cpp
    ``` C++
    void AProjectileGrenade::Destroyed()
    {
        ExplodeDamage();
        Super::Destroyed();
    }
    void AProjectile::ExplodeDamage()
    {
        APawn* FiringPawn = GetInstigator();
        if (FiringPawn && HasAuthority())
        {
            AController* FiringController = FiringPawn->GetController();
            if (FiringController)
            {
                UGameplayStatics::ApplyRadialDamageWithFalloff(
                    this,
                    Damage,
                    10.f,
                    GetActorLocation(),
                    DamageInnerRadius,
                    DamageOuterRadius,
                    1.f,
                    UDamageType::StaticClass(),
                    TArray<AActor*>(),
                    this,
                    FiringController
                );
            }
        }
    }
    ```
- Reload 몽타주에 reload 애니메이션 다 설정하기
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PlayReloadMontage()
    {
        if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && ReloadMontage)
        {
            AnimInstance->Montage_Play(ReloadMontage);
            FName SectionName;
            switch (Combat->EquippedWeapon->GetWeaponType())
            {
            case EWeaponType::EWT_AssaultRifle:
                SectionName = FName("Rifle");
                break;
            case EWeaponType::EWT_RocketLauncher:
                SectionName = FName("RocketLauncher");
                break;
            case EWeaponType::EWT_Pistol:
                SectionName = FName("Pistol");
                break;
            case EWeaponType::EWT_SubmachineGun:
                SectionName = FName("Pistol");
                break;
            case EWeaponType::EWT_Shotgun:
                SectionName = FName("Shotgun");
                break;
            case EWeaponType::EWT_SniperRifle:
                SectionName = FName("SniperRifle");
                break;
            case EWeaponType::EWT_GrenadeLauncher:
                SectionName = FName("GrenadeLaunchers");
                break;
            }
            AnimInstance->Montage_JumpToSection(SectionName);
        }
    }
    ```

## Shotgun Reload
- Reload의 Shotgun쪽에 장전을 할 때마다 Shell Notify삽입, ShotgunEnd 섹션 추가
- CombatComponent.h
    ``` C++
    public:
        UFUNCTION(BlueprintCallable)
        void ShotgunShellReload();
	    void JumpToShotgunEnd();
    private:
    	void UpdateShotgunAmmoValues();
    ```
- CombatComponent.cpp
    ``` C++
    #include "Blaster/Character/BlasterAnimInstance.h"
    void UCombatComponent::ShotgunShellReload()
    {
        if (Character && Character->HasAuthority())
        {
            UpdateShotgunAmmoValues();
        }
    }

    void UCombatComponent::UpdateShotgunAmmoValues()
    {
        if (Character == nullptr && EquippedWeapon != nullptr) return;
        if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
        {
            CarriedAmmoMap[EquippedWeapon->GetWeaponType()] -= 1;
            CarriedAmmo = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
        }
        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            Controller->SetHUDCarriedAmmo(CarriedAmmo);
        }
        EquippedWeapon->AddAmmo(-1);
	    bCanFire = true;
        if (EquippedWeapon->IsFull()|| CarriedAmmo==0)
        {
            JumpToShotgunEnd();
        }
    }
    void UCombatComponent::JumpToShotgunEnd()
    {
        UAnimInstance* AnimInstance = Character->GetMesh()->GetAnimInstance();
        if (AnimInstance && Character->GetReloadMontage())
        {
            AnimInstance->Montage_JumpToSection(FName("ShotgunEnd"));
        }
    }
    void UCombatComponent::OnRep_CarriedAmmo()
    {
        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            Controller->SetHUDCarriedAmmo(CarriedAmmo);
        }
        bool bJumpToShotgunEnd =
            CombatState == ECombatState::ECS_Reloading
            && EquippedWeapon != nullptr
            && EquippedWeapon->GetWeaponType() == EWeaponType::EWT_Shotgun
            && CarriedAmmo == 0;
        if (bJumpToShotgunEnd)
        {
            JumpToShotgunEnd();
        }
    }
    bool UCombatComponent::CanFire()
    {
        if (EquippedWeapon == nullptr) return false;
        if (EquippedWeapon->IsEmpty() && bCanFire && CombatState == ECombatState::ECS_Reloading && EquippedWeapon->GetWeaponType() == EWeaponType::EWT_Shotgun) return true;
        return !EquippedWeapon->IsEmpty() && bCanFire && CombatState == ECombatState::ECS_Unoccupied;
    }
    void UCombatComponent::MulticastFire_Implementation(const FVector_NetQuantize& TraceHitResult)
    {
        if (EquippedWeapon == nullptr) return;
        if (Character && CombatState == ECombatState::ECS_Reloading && EquippedWeapon->GetWeaponType() == EWeaponType::EWT_Shotgun)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire(TraceHitResult);
            CombatState = ECombatState::ECS_Unoccupied;
            return;
        }
        if (Character && CombatState == ECombatState::ECS_Unoccupied)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire(TraceHitResult);
        }
    }
    ```
- Weapon.h
    ``` C++
    public:
    	bool IsFull();
    ```
- Weapon.cpp
    ``` C++
    #include "Blaster/BlasterComponents/CombatComponent.h"
    bool AWeapon::IsFull()
    {
        return Ammo == MagCapacity;
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	FORCEINLINE UAnimMontage* GetReloadMontage() const { return ReloadMontage; };
    ```
- BlasterAnimBP에서 Shell AnimNotify가 트리거되면 Character->Combat의 ShotgunShellReload호출

## outline 효과
- postprocess volume에 post process material을 추가할 수 있음
    - (이미 강의에서 만들어준)pp_hightlight를 추가
    - 이제 어떤 액터를 선택해서 custom depth를 설정 가능
        - Render Custom Depth Pass = true
        - CustomDepth Stencil Value는 250으로 설정
- Weapon.h
    ``` C++
    public:
        // Enable or disable custom depth
        void EnableCustomDepth(bool bEnable);
    ```
- Weapon.cpp
    ``` C++
    AWeapon::AWeapon()
    {
        ...
        WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
        WeaponMesh->MarkRenderStateDirty();
        EnableCustomDepth(true);
        ...
    }
    void AWeapon::EnableCustomDepth(bool bEnable)
    {
        if (WeaponMesh)
        {
            WeaponMesh->SetRenderCustomDepth(bEnable);
        }
    }
    void AWeapon::OnRep_WeaponState()
    {
        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ...
		    EnableCustomDepth(false);
            break;
        case EWeaponState::EWS_Dropped:
            ...
            WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
            WeaponMesh->MarkRenderStateDirty();
            EnableCustomDepth(true);
            break;
        }
    }

    void AWeapon::SetWeaponState(EWeaponState State)
    {
        WeaponState = State;

        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ...
            EnableCustomDepth(false);
            break;
        case EWeaponState::EWS_Dropped:
            ...
            WeaponMesh->SetCustomDepthStencilValue(CUSTOM_DEPTH_BLUE);
            WeaponMesh->MarkRenderStateDirty();
            EnableCustomDepth(true);
            break;
        }
    }
    ```
- WeaponTypes.h
    ``` C++
    #define CUSTOM_DEPTH_PURPLE 250
    #define CUSTOM_DEPTH_BLUE 251
    #define CUSTOM_DEPTH_TAN 252
    ```
- 모든 무기 BP에 가서 RenderPass, Depth 설정하기
- ProjectSettings에서 CustomDepth-StencilPass를 Enabled with Stencil로 설정

## Grenade Throw Animation
- Grenade Toss 애니메이션을 토대로 ThrowGrenade를 제작
    - slot은 WeaponSlot
    - 끝쪽에 FinishGrenadeThrow 노티파이추가
- BP_BlasterCharacter에서 ThrowGrenadeMontage 설정
    - EventGraph에서 Character와 Combat IsValid체크하고 ThrowGrenadeFinished호출
- BlasterCharacter.h
    ``` C++
    public:
    	void PlayThrowGrenadeMontage();
    private:
    	UPROPERTY(EditAnywhere, Category = Combat)
		UAnimMontage* ThrowGrendaeMontage;
    protected:
    	void GrenadeButtonPressed();
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::PlayThrowGrenadeMontage()
    {
        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && ThrowGrendaeMontage)
        {
            AnimInstance->Montage_Play(ThrowGrendaeMontage);
        }
    }
    void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        ...
        PlayerInputComponent->BindAction("ThrowGrenade", IE_Pressed, this, &ABlasterCharacter::GrenadeButtonPressed);
    }
    void ABlasterCharacter::GrenadeButtonPressed()
    {
        if (Combat)
        {
            Combat->ThrowGrenade();
        }
    }
    ```
- ProjectSetting의 Input에서 ThrowGrenade를 T키로 액션 매핑

- CombatComponent.h
    ``` C++
    public:
        UFUNCTION(BlueprintCallable)
        void ThrowGrenadeFinished();
    protected:
	    void ThrowGrenade();
        UFUNCTION(Server,Reliable)
        void ServerThrowGrenade();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::ThrowGrenadeFinished()
    {
        CombatState = ECombatState::ECS_Unoccupied;
    }
    void UCombatComponent::ThrowGrenade()
    {
	    if (CombatState != ECombatState::ECS_Unoccupied) return;
        CombatState = ECombatState::ECS_ThrowingGrenade;
        if (Character)
        {
            Character->PlayThrowGrenadeMontage();
        }
        if (Character && !Character->HasAuthority())
        {
            ServerThrowGrenade();
        }
    }
    void UCombatComponent::ServerThrowGrenade_Implementation()
    {
        CombatState = ECombatState::ECS_ThrowingGrenade;
        if (Character)
        {
            Character->PlayThrowGrenadeMontage();
        }
    }
    void UCombatComponent::OnRep_CombatState()
    {
        switch (CombatState)
        {
        ...
        case ECombatState::ECS_ThrowingGrenade:
            if (Character && !Character->IsLocallyControlled())
            {
                Character->PlayThrowGrenadeMontage();
            }
            break;
        }
    }

    ```
- CombatState.h
    ``` C++
    UENUM(BlueprintType)
    enum class ECombatState : uint8
    {
        ECS_Unoccupied UMETA(DisplayName = "Unoccupied"),
        ECS_Reloading UMETA(DisplayName = "Reloading"),
        ECS_ThrowingGrenade UMETA(DisplayName = "ThrowingGrenade"),
        ECS_MAX UMETA(DisplayName = "DefaultMAX")
    };
    ```

- 수류탄 던질때는 FABRIK안 쓸 것임
    - BlasterAnimInstance.cpp
        ``` C++
        void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
        {
            ...
            bUseFABRIK = BlasterCharacter->GetCombatState() == ECombatState::ECS_Unoccupied;
            bUseAimOffsets = BlasterCharacter->GetCombatState() == ECombatState::ECS_Unoccupied && !BlasterCharacter->GetDisableGameplay();
            bTransformRightHand = BlasterCharacter->GetCombatState() == ECombatState::ECS_Unoccupied && !BlasterCharacter->GetDisableGameplay();
        }
        ```
    - CombatComponent.cpp
        ``` C++
        void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
        {
            if (Character == nullptr || WeaponToEquip == nullptr) return;
            if (CombatState != ECombatState::ECS_Unoccupied) return;
            ...
        }
        void UCombatComponent::Reload()
        {
            if (CarriedAmmo > 0 && CombatState == ECombatState::ECS_Unoccupied)
            {
                ServerReload();
            }
        }
        ```

## Weapon Attachment while Throwing Grenades
- CombatComponent.h
    ``` C++
    protected:
    	void DropEquippedWeapon();
	    void AttachActorToRightHand(AActor* ActorToAttach);
        void AttachActorToLeftHand(AActor* ActorToAttach);
        void UpdateCarriedAmmo();
        void PlayEquipWeaponSound();
        void ReloadEmptyWeapon();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        if (Character == nullptr || WeaponToEquip == nullptr) return;
        if (CombatState != ECombatState::ECS_Unoccupied) return;
        DropEquippedWeapon();
        EquippedWeapon = WeaponToEquip;
        EquippedWeapon->SetWeaponState(EWeaponState::EWS_Equipped);
        AttachActorToRightHand(EquippedWeapon);
        EquippedWeapon->SetOwner(Character);
        EquippedWeapon->SetHUDAmmo();
        UpdateCarriedAmmo();
        PlayEquipWeaponSound();
        ReloadEmptyWeapon();
        Character->GetCharacterMovement()->bOrientRotationToMovement = false;
        Character->bUseControllerRotationYaw = true;
    }
    void UCombatComponent::DropEquippedWeapon()
    {
        if (EquippedWeapon)
        {
            EquippedWeapon->Dropped();
        }
    }
    void UCombatComponent::AttachActorToRightHand(AActor* ActorToAttach)
    {
        if (Character == nullptr || Character->GetMesh() == nullptr || ActorToAttach == nullptr || EquippedWeapon == nullptr) return;
        const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(FName("RightHandSocket"));
        if (HandSocket)
        {
            HandSocket->AttachActor(ActorToAttach, Character->GetMesh());
        }
    }
    void UCombatComponent::AttachActorToLeftHand(AActor* ActorToAttach)
    {
        if (Character == nullptr || Character->GetMesh() == nullptr || ActorToAttach == nullptr || EquippedWeapon == nullptr) return;
        bool bUsePistolSocket = EquippedWeapon->GetWeaponType() == EWeaponType::EWT_Pistol || EquippedWeapon->GetWeaponType() == EWeaponType::EWT_SubmachineGun;
        FName SocketName = bUsePistolSocket ? FName("PistolSocket") : FName("LeftHandSocket");
        const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(SocketName);
        if (HandSocket)
        {
            HandSocket->AttachActor(ActorToAttach, Character->GetMesh());
        }
    }
    void UCombatComponent::UpdateCarriedAmmo()
    {
        if (EquippedWeapon == nullptr) return;
        if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
        {
            CarriedAmmo = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
        }
        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            Controller->SetHUDCarriedAmmo(CarriedAmmo);
        }
    }

    void UCombatComponent::PlayEquipWeaponSound()
    {
        if (Character && EquippedWeapon && EquippedWeapon->EquipSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                this,
                EquippedWeapon->EquipSound,
                Character->GetActorLocation()
            );
        }
    }
    void UCombatComponent::ReloadEmptyWeapon()
    {
	    if (EquippedWeapon && EquippedWeapon->IsEmpty())
        {
            Reload();
        }
    }
    void UCombatComponent::ThrowGrenadeFinished()
    {
        CombatState = ECombatState::ECS_Unoccupied;
        AttachActorToRightHand(EquippedWeapon);
    }
    void UCombatComponent::OnRep_CombatState()
    {
        switch (CombatState)
        {
        ...
        case ECombatState::ECS_ThrowingGrenade:
            if (Character && !Character->IsLocallyControlled())
            {
                Character->PlayThrowGrenadeMontage();
                AttachActorToLeftHand(EquippedWeapon);
            }
            break;
        }
    }
    void UCombatComponent::ThrowGrenade()
    {
        ...
        if (Character)
        {
            Character->PlayThrowGrenadeMontage();
            AttachActorToLeftHand(EquippedWeapon);
        }
        ...
    }
    void UCombatComponent::ServerThrowGrenade_Implementation()
    {
        CombatState = ECombatState::ECS_ThrowingGrenade;
        if (Character)
        {
            Character->PlayThrowGrenadeMontage();
            AttachActorToLeftHand(EquippedWeapon);
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
            PlayEquipWeaponSound();
        }
    }
    void UCombatComponent::FireTimerFinished()
    {
        if (EquippedWeapon == nullptr) return;
        bCanFire = true;
        if (bFireButtonPressed && EquippedWeapon->bAutomatic)
        {
            Fire();
        }
        ReloadEmptyWeapon();
    }
    ```

## Grenade
- BlasterCharacter.h
    ``` C++
    private:
        // Grenade
        UPROPERTY(VisibleAnywhere)
        UStaticMeshComponent* AttachedGrenade;
    ```
- BlasterCharacter.cpp
    ``` C++
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
        AttachedGrenade = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("AttachedGrenade"));
	    AttachedGrenade->SetupAttachment(GetMesh(), FName("GrenadeSocket"));
	    AttachedGrenade->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }
    void ABlasterCharacter::BeginPlay()
    {
        ...
        if (AttachedGrenade)
        {
            AttachedGrenade->SetVisibility(false);
        }
    }
    ```
- CombatComponent.h
    ``` C++
    public:
    	UFUNCTION(BlueprintCallable)
		void LaunchGrenade();
    protected:
    	void ShowAttachedGrenade(bool bShowGrenade);
    public:
	    FORCEINLINE UStaticMeshComponent* GetAttachedGrenade() const { return AttachedGrenade; };
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::LaunchGrenade()
    {
        ShowAttachedGrenade(false);
    }
    void UCombatComponent::Reload()
    {
        if (CarriedAmmo > 0 && CombatState == ECombatState::ECS_Unoccupied && EquippedWeapon && !EquippedWeapon->IsFull())
        {
            ServerReload();
        }
    }
    void UCombatComponent::ThrowGrenade()
    {
        ...
        if (Character)
        {
            Character->PlayThrowGrenadeMontage();
            AttachActorToLeftHand(EquippedWeapon);
            ShowAttachedGrenade(true);
        }
        ...
    }
    void UCombatComponent::ServerThrowGrenade_Implementation()
    {
        CombatState = ECombatState::ECS_ThrowingGrenade;
        if (Character)
        {
            Character->PlayThrowGrenadeMontage();
            AttachActorToLeftHand(EquippedWeapon);
            ShowAttachedGrenade(true);
        }
    }
    void UCombatComponent::OnRep_CombatState()
    {
        switch (CombatState)
        {
        ...
        case ECombatState::ECS_ThrowingGrenade:
            if (Character && !Character->IsLocallyControlled())
            {
                Character->PlayThrowGrenadeMontage();
                AttachActorToLeftHand(EquippedWeapon);
                ShowAttachedGrenade(true);
            }
            break;
        }
    }
    void UCombatComponent::ShowAttachedGrenade(bool bShowGrenade)
    {
        if (Character && Character->GetAttachedGrenade())
        {
            Character->GetAttachedGrenade()->SetVisibility(bShowGrenade);
        }
    }
    ```
- ABP에서 Character와 Combat Valid체크하고 LaunchGrenade 호출


- BP_ProjectileGrenade를 복제해서 BP_ThrowGrenade생성
    - 메쉬는 삭제
    - particle system을 추가해서 이펙트 설정
    - 파티클에 맞게 box extent 변경

- CombatComponent.h
    ``` C++
    protected:
        UPROPERTY(EditAnywhere)
        TSubclassOf<class AProjectile> GrenadeClass;
    ```
- CombatComponent.cpp
    ``` C++
    #include "Blaster/Weapon/Projectile.h"

    void UCombatComponent::LaunchGrenade()
    {
        ShowAttachedGrenade(false);

        if (Character && Character->HasAuthority() && GrenadeClass && Character->GetAttachedGrenade())
        {
            const FVector StartingLocation = Character->GetAttachedGrenade()->GetComponentLocation();
            FVector ToTarget = HitTarget - StartingLocation;
            FActorSpawnParameters SpawnParams;
            SpawnParams.Owner = Character;
            SpawnParams.Instigator = Character;
            UWorld* World = GetWorld();
            if (World)
            {
                World->SpawnActor<AProjectile>(GrenadeClass, StartingLocation, ToTarget.Rotation(), SpawnParams);
            }
        }
    }
    ```
- BP_BlasterCharacter에 GrenadeClass를 BP_ThrowGrenade로 변경

- CombatComponent.h
    ``` C++
    public:
        UFUNCTION(Server,Reliable)
		void ServerLaunchGrenade(const FVector_NetQuantize& Target);
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::LaunchGrenade()
    {
        ShowAttachedGrenade(false);
        if (Character && Character->IsLocallyControlled())
        {
            ServerLaunchGrenade(HitTarget);
        }
    }
    void UCombatComponent::ServerLaunchGrenade_Implementation(const FVector_NetQuantize& Target)
    {
        if (Character && GrenadeClass && Character->GetAttachedGrenade())
        {
            const FVector StartingLocation = Character->GetAttachedGrenade()->GetComponentLocation();
            FVector ToTarget = Target - StartingLocation;
            FActorSpawnParameters SpawnParams;
            SpawnParams.Owner = Character;
            SpawnParams.Instigator = Character;
            UWorld* World = GetWorld();
            if (World)
            {
                World->SpawnActor<AProjectile>(GrenadeClass, StartingLocation, ToTarget.Rotation(), SpawnParams);
            }
        }
    }
    void UCombatComponent::ThrowGrenade(const FVector_NetQuantize& Target)
    {
        if (CombatState != ECombatState::ECS_Unoccupied || EquippedWeapon==nullptr) return;
        ...
    }
    ```

## Grenade HUD
- CharacterOverlay.h
    ``` C++
    public:  
    	UPROPERTY(meta = (BindWidget))
		UTextBlock* GrenadeText
    ```
- BlasterPlayerController.h
    ``` C++
    public:
    	void SetHUDGrenades(int32 Grenades);
    private:
    	int32 HUDGrenades;
    ```
- BlasterPlayerController.cpp
    ``` C++
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
                    ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(GetPawn());
                    if (BlasterCharacter && BlasterCharacter->GetCombat())
                    {
                        SetHUDGrenades(BlasterCharacter->GetCombat()->GetGrenades());
                    }
                }
            }
        }
    }
    void ABlasterPlayerController::SetHUDGrenades(int32 Grenades)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->GrenadeText;
        if (bHUDValid)
        {
            FString GrenadeText = FString::Printf(TEXT("%d"), Grenades);
            BlasterHUD->CharacterOverlay->GrenadeText->SetText(FText::FromString(GrenadeText));
        }
        else
        {
            HUDGrenades = Grenades;
        }
    }
    ```
- CombatComponent.h
    ``` C++
    private:
        UFUNCTION()
        void OnRep_Grenades();
        UPROPERTY(ReplicatedUsing = OnRep_Grenades)
        int32 Grenades = 0;
        UPROPERTY(EditAnywhere)
        int32 MaxGrenades = 4;
	    void UpdateHUDGrenades();
    public:
        FORCEINLINE int32 GetGrenades() const { return Grenades; }
    ```
- CombatComponent.cpp
    ``` C++
    
    void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        ...
        DOREPLIFETIME(UCombatComponent, Grenades);
    }

    void UCombatComponent::ThrowGrenade()
    {
        if (Grenades == 0) return;
        ...
        if (Character && Character->HasAuthority())
        {
            Grenades = FMath::Clamp(Grenades - 1, 0, MaxGrenades);
            UpdateHUDGrenades();
        }
    }
    void UCombatComponent::OnRep_Grenades()
    {
        UpdateHUDGrenades();
    }
    void UCombatComponent::ServerThrowGrenade_Implementation()
    {
        if (Grenades == 0) return;
        ...
	    Grenades = FMath::Clamp(Grenades - 1, 0, MaxGrenades);
	    UpdateHUDGrenades();
    }
    void UCombatComponent::UpdateHUDGrenades()
    {
        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            Controller->SetHUDGrenades(Grenades);
        }
    }
    ```

- 죽었는데 또 죽는 현상 방지
    - BlasterCharacer.cpp
        ``` C++
        void ABlasterCharacter::ReceiveDamage(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatorController, AActor* DamageCauser)
        {
            if (bElimned) return;
            ...
        }
        ```