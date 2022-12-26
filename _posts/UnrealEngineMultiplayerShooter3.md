# Weapon
- ProjectileWeapon
    - Spawn a Projectile
    - Has a Velocity
    - May/may not have gravity
    - Hit Event
    - Tracer particles
- HitScan Weawpons
    - Perform a line trace
    - Instant hit
    - Beam Particles

## ProjectileWeapon
- Weapon을 상속받는 ProjectileWeapon 클래스 생성
- ProjectileWeapon.h
    ``` C++
    private:
        UPROPERTY(EditAnywhere)
        class UBoxComponent* CollisionBox;
    ```
- ProjectileWeapon.cpp
    ``` C++
    #include "Components/BoxComponent.h"
    AProjectile::AProjectile()
    {
        PrimaryActorTick.bCanEverTick = true;
        CollisionBox = CreateDefaultSubobject<UBoxComponent>(TEXT("CollisionBox"));
        SetRootComponent(CollisionBox);
        CollisionBox->SetCollisionObjectType(ECollisionChannel::ECC_WorldDynamic);
        CollisionBox->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
        CollisionBox->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
        CollisionBox->SetCollisionResponseToChannel(ECollisionChannel::ECC_Visibility, ECollisionResponse::ECR_Block);
        CollisionBox->SetCollisionResponseToChannel(ECollisionChannel::ECC_WorldStatic, ECollisionResponse::ECR_Block);
    }
    ```
- Actor를 상속받는 Projectile클래스 생성


- Project Settings > Input > Action Mappings
    - Fire 이름의 Left Mouse Button binding

- BlasterCharacter.h
    ``` C++
    public:
    	void PlayFireMontage(bool bAiming);
    protected:
        void FireButtonPressed();
        void FireButtonReleased();
    private:
        UPROPERTY(EditAnywhere, Category = Combat)
        class UAnimMontage* FireWeaponMontage;
    ```
    - BP_BlasterChararcter에서 알맞은 FireWeaponMontage 설
- BlasterCharacter.cpp
    ``` C++
    #include "BlasterAnimInstance.h"

    void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        ...
        PlayerInputComponent->BindAction("Fire", IE_Pressed, this, &ABlasterCharacter::FireButtonPressed);
        PlayerInputComponent->BindAction("Fire", IE_Released, this, &ABlasterCharacter::FireButtonReleased);
    }
    void ABlasterCharacter::FireButtonPressed()
    {
        if (Combat)
        {
            Combat->FireButtonPressed(true);
        }
    }
    void ABlasterCharacter::FireButtonReleased()
    {
        if (Combat)
        {
            Combat->FireButtonPressed(false);
        }
    }
    void ABlasterCharacter::PlayFireMontage(bool bAiming)
    {
        if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && FireWeaponMontage)
        {
            AnimInstance->Montage_Play(FireWeaponMontage);
            FName SectionName;
            SectionName = bAiming ? FName("RifleAim") : FName("RifleHip");
            AnimInstance->Montage_JumpToSection(SectionName);
        }
    }
    ```
- CombatComponent.h
    ``` C++
    protected:
	    void FireButtonPressed(bool bPressed);
    private:
    	bool bFireButtonPressed;
    ```
- CombatComponent.cpp   
    ``` C++
    void UCombatComponent::FireButtonPressed(bool bPressed)
    {
        bFireButtonPressed = bPressed;
        if (Character && bFireButtonPressed)
        {
            Character->PlayFireMontage(bAiming);
        }
    }
    ```

- Fire 애니메이션은 Additive여야함
    - Fire_Rifle : Mesh Space, Standard Animation Frame, Zero_Pose
    - Fire_Rifle_Ironsights : Mesh Space, Standard Animation Frame, Aim_Zero_Pose

- Fire_Rifle_Hip기반으로 FireWeapon Anim Montage 생성
    - RifleHip Section 만들고 Default Secion 지우기
    - Group을 WeaponSlot으로 설정
    - Fire_Rifle_Ironsights 추가하고 RifleAim section 생성하기
    - MontageSection에서 Clear
    - blend In, Out의 blentime은 0

- Equipped cache를 사용해서, Slot 'WeaponSlot'에 연결 후 cache에 저장(WeaponSlot)

## Fire Weapon Effects
- Weapon.h
    ``` C++
    public:
    	void Fire();
    private:
    	UPROPERTY(EditAnywhere, Category = "Weapon Properties")
		class UAnimationAsset* FireAnimation;
    ```
- Weapon.cpp    
    ``` C++
    #include "Animation/AnimationAsset.h"
    #include "Components/SkeletalMeshComponent.h"
    
    void AWeapon::Fire()
    {
        if (FireAnimation)
        {
            WeaponMesh->PlayAnimation(FireAnimation, false);
        }
    }
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::FireButtonPressed(bool bPressed)
    {
        bFireButtonPressed = bPressed;
        if (EquippedWeapon == nullptr) return;
        if (Character && bFireButtonPressed)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire();
        }
    }
    ```
- ProjectileWeapon 상속받는 BP_AssaultRifle 생성
    - Mesh 설정, Area Sphere, Widget 설정
    - Fire Animation에 Fire_Rifle_W 애니메이션 설정

## Fire Effect in Multiplayer
- RPC를 사용하자

- CombatComponent.h
    ``` C++
    protected:
        UFUNCTION(Server,Reliable)
        void ServerFire();
        UFUNCTION(NetMulticast, Reliable)
		void MulticastFire();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::FireButtonPressed(bool bPressed)
    {
        bFireButtonPressed = bPressed;
        if (bFireButtonPressed)
        {
            ServerFire();
        }
    }

    void UCombatComponent::ServerFire_Implementation()
    {
        MulticastFire();
    }
    void UCombatComponent::MulticastFire_Implementation()
    {
        if (EquippedWeapon == nullptr) return;
        if (Character)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire();
        }
    }
    ```
- 최대한 네트워크간 넘기는 정보량을 줄이긴 해야함

## Hit Target
- CombatComponent.h
    ``` C++
    #define TRACE_LENGTH 80000
    ...
    protected:
    	void TraceUnderCrosshairs(FHitResult& TraceHitResult);
    private:
    	FVector HitTarget;
    ```
- CombatComponetn.cpp
    ``` C++
    #include "Kismet/GameplayStatics.h"
    #include "DrawDebugHelpers.h"

    void UCombatComponent::TraceUnderCrosshairs(FHitResult& TraceHitResult)
    {
        FVector2D ViewportSize;
        if (GEngine && GEngine->GameViewport)
        {
            GEngine->GameViewport->GetViewportSize(ViewportSize);
        }
        FVector2D CrosshairLocation(ViewportSize.X / 2.f, ViewportSize.Y / 2.f);
        FVector CrosshairWorldPosition;
        FVector CrosshairWorldDirection;
        bool bScreenToWorld = UGameplayStatics::DeprojectScreenToWorld(
            UGameplayStatics::GetPlayerController(this, 0),
            CrosshairLocation,
            CrosshairWorldPosition,
            CrosshairWorldDirection
        );
        if (bScreenToWorld)
        {
            FVector Start = CrosshairWorldPosition;
            FVector End = Start + CrosshairWorldDirection * TRACE_LENGTH;
            GetWorld()->LineTraceSingleByChannel(
                TraceHitResult,
                Start,
                End,
                ECollisionChannel::ECC_Visibility
            );
            if (!TraceHitResult.bBlockingHit)
            {
                TraceHitResult.ImpactPoint = End;
			    HitTarget = End;
            }
            else
            {
			    HitTarget = TraceHitResult.ImpactPoint;
                DrawDebugSphere(
                    GetWorld(),
                    TraceHitResult.ImpactPoint,
                    12.f,
                    12,
                    FColor::Red
                );
            }
        }
    }

    UCombatComponent::UCombatComponent()
    {
        PrimaryComponentTick.bCanEverTick = true;
        ...
    }
    void UCombatComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
        FHitResult HitResult;
        TraceUnderCrosshairs(HitResult);
    }
    ```
- Character의 CameraBoom 위치 조정 (350)(0,75,75)


## Spawn the Projectile
- ProjectileWeapon.h
    ``` C++
    private:
        UPROPERTY(EditAnywhere)
        TSubclassOf<class AProjectile> ProjectileClass;
    ```
- Blueprints > Weapon > Projectiles 폴더에 Projectile을 상속받는 BP_Projectile 생성
- BP_AssaultRifle에 ProjectileClass를 BP_Projectile로 설정

- Weapon.h
    ``` C++
    public:
    	virtual void Fire(const FVector& HitTarget);
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::Fire(const FVector& HitTarget)
    {
        if (FireAnimation)
        {
            WeaponMesh->PlayAnimation(FireAnimation, false);
        }
    }
    ```
- ProjectileWeapon.h
    ``` C++
    public:
        virtual void Fire(const FVector& HitTarget) override;
    ```
- ProjectileWeapon.cpp
    ``` C++
    #include "ProjectileWeapon.h"
    #include "Engine/SkeletalMeshSocket.h"
    #include "Projectile.h"

    void AProjectileWeapon::Fire(const FVector& HitTarget)
    {
        Super::Fire(HitTarget);
        APawn* InstigatorPawn = Cast<APawn>(GetOwner());
        const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName(FName("MuzzleFlash"));
        if (MuzzleFlashSocket)
        {
            FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
            // From Muzzle flash socket to hit location from TraceUnderCrosshiars
            FVector ToTarget = HitTarget - SocketTransform.GetLocation();
            FRotator TargetRotation = ToTarget.Rotation();
            if (ProjectileClass && InstigatorPawn)
            {
                FActorSpawnParameters SpawnParams;
                SpawnParams.Owner = GetOwner();
                SpawnParams.Instigator = InstigatorPawn;
                UWorld* World = GetWorld();
                if (World)
                {
                    World->SpawnActor<AProjectile>(ProjectileClass, SocketTransform.GetLocation(), TargetRotation, SpawnParams);
                }
            }
        }
    }
    ```
- CombatComponent.h
    ``` C++
    void UCombatComponent::MulticastFire_Implementation()
    {
        if (EquippedWeapon == nullptr) return;
        if (Character)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire(HitTarget);
        }
    }
    ```
## Projectile Movement Component
- Projectile.h
    ``` C++
    private:
        UPROPERTY(VisibleAnywhere)
		class UProjectileMovementComponent* ProjectileMovementComponent;
    ```
- Projectile.cpp
    ``` C++
    #include "GameFramework/ProjectileMovementComponent.h"
    AProjectile::AProjectile()
    {
        ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovementComponent"));
	    ProjectileMovementComponent->bRotationFollowsVelocity = true;
        ProjectileMovementComponent->InitialSpeed = 15000;
        ProjectileMovementComponent->MaxSpeed = 15000;
    }
    ```

## Projectile Tracer
- Projectile.h
    ``` C++
	UPROPERTY(EditAnywhere)
		class UParticleSystem* Tracer;

	class UParticleSystemComponent* TracerComponent;
    ```
- Projectile.cpp
    ``` C++
    #include "Kismet/GameplayStatics.h"
    #include "Particles/ParticleSystemComponent.h"
    #include "Particles/ParticleSystem.h"

    AProjectile::AProjectile()
    {
        PrimaryActorTick.bCanEverTick = true;
        bReplicates = true;
        ...
    }
    void AProjectile::BeginPlay()
    {
        Super::BeginPlay();
        if (Tracer)
        {
            TracerComponent = UGameplayStatics::SpawnEmitterAttached(
                Tracer,
                CollisionBox,
                FName(),
                GetActorLocation(),
                GetActorRotation(),
                EAttachLocation::KeepWorldPosition
            );
        }
    }
    ```
- BP_Projectile에서 Tracer 속성에 알맞은 Particle 설정

- ProjectileWeapon.cpp  
    ``` C++
    void AProjectileWeapon::Fire(const FVector& HitTarget)
    {
        Super::Fire(HitTarget);
        if (!HasAuthority()) return;
        ...
    }
    ```

## Replicating Hit Target
- CombatComponent.h
    ``` C++
    protected:
    	UFUNCTION(Server, Reliable)
		void ServerFire(const FVector_NetQuantize& TraceHitResult);
	    UFUNCTION(NetMulticast, Reliable)
		void MulticastFire(const FVector_NetQuantize& TraceHitResult);
    private:
        //FVector HitTarget;
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::FireButtonPressed(bool bPressed)
    {
        bFireButtonPressed = bPressed;
        if (bFireButtonPressed)
        {
            FHitResult HitResult;
            TraceUnderCrosshairs(HitResult);
            ServerFire(HitResult.ImpactPoint);
        }
    }
    void UCombatComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    }
    void UCombatComponent::ServerFire_Implementation(const FVector_NetQuantize& TraceHitResult)
    {
        MulticastFire(TraceHitResult);
    }
    void UCombatComponent::MulticastFire_Implementation(const FVector_NetQuantize& TraceHitResult)
    {
        if (EquippedWeapon == nullptr) return;
        if (Character)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire(TraceHitResult);
        }
    }
    void UCombatComponent::TraceUnderCrosshairs(FHitResult& TraceHitResult)
    {
        ...
        // HitTarget관련 부분을 없앰
        if (bScreenToWorld)
        {
            FVector Start = CrosshairWorldPosition;
            FVector End = Start + CrosshairWorldDirection * TRACE_LENGTH;
            GetWorld()->LineTraceSingleByChannel(
                TraceHitResult,
                Start,
                End,
                ECollisionChannel::ECC_Visibility
            );
        }
    }
    ```
- FVector_NetQuantize : 벡터를 integer로 round함
    - 정보를 잘라내긴 하겠지만 더 효율적임

## Projectile Hit Events
- Projectile.h
    ``` C++
    public:
    	virtual void Destroyed() override;
    protected:    
        UFUNCTION()
        virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActo, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);

        UPROPERTY(EditAnywhere)
        class UParticleSystem* ImpactParticles;
        UPROPERTY(EditAnywhere)
        class USoundCue* ImpactSound;
    ```
- Projectile.cpp
    ``` C++
    #include "Sound/SoundCue.h"
    void AProjectile::BeginPlay()
    {
        ...
        if (HasAuthority())
        {
            CollisionBox->OnComponentHit.AddDynamic(this, &AProjectile::OnHit);
        }
    }
    void AProjectile::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActo, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        Destroy();
    }

    void AProjectile::Destroyed()
    {
        Super::Destroyed();
        if (ImpactParticles)
        {
            UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ImpactParticles, GetActorTransform());
        }
        if (ImpactSound)
        {
            UGameplayStatics::PlaySoundAtLocation(this, ImpactSound, GetActorLocation());
        }
    }
    ```
    - Destroyed는 Server와 Client 모두에서 불리기 때문에 이렇게 하는 것이 좋음
- BP_Projectile에서 Impact Particle, Impact Sound를 알맞게 설정

## Bullet Shell Effect
- Blaster > Weapon 폴더에 Actor를 상속받는 "Casing" 클래스 생성

- Casing.h
    ``` C++
    public:	
        ACasing();
    private:
        UPROPERTY(VisibleAnywhere)
        UStaticMeshComponent* CasingMesh;

        UPROPERTY(EditAnywhere)
        float ShellEjectionImpulse;

        UPROPERTY(EditAnywhere)
            class USoundCue* ShellSound;
    protected:
        virtual void BeginPlay() override;

        UFUNCTION()
        virtual void OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);
    ```
- Casing.cpp
    ``` C++
    #include "Casing.h"
    #include "Kismet/GameplayStatics.h"
    #include "Sound/SoundCue.h"

    ACasing::ACasing()
    {
        PrimaryActorTick.bCanEverTick = false;
        CasingMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("CasingMesh"));
        SetRootComponent(CasingMesh);
	    CasingMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
        CasingMesh->SetSimulatePhysics(true);
        CasingMesh->SetEnableGravity(true);
	    CasingMesh->SetNotifyRigidBodyCollision(true);
	    ShellEjectionImpulse = 10.f;
    }
    void ACasing::BeginPlay()
    {
        Super::BeginPlay();
	    CasingMesh->OnComponentHit.AddDynamic(this, &ACasing::OnHit);
	    CasingMesh->AddImpulse(GetActorForwardVector() * ShellEjectionImpulse);
    }
    void ACasing::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        if (ShellSound) 
        {
            UGameplayStatics::PlaySoundAtLocation(this, ShellSound, GetActorLocation());
        }
        Destroy();
    }
    ```
    - SetNotifyRigidBodyCollision이게 Blueprint에서는 Simulation Generates Hit Events
- Weapon.h
    ``` C++
    private:
	    UPROPERTY(EditAnywhere)
    	TSubclassOf<class ACasing> CasingClass;
    ```
- Weapon.cpp
    ``` C++
    #include "Casing.h"
    #include "Engine/SkeletalMeshSocket.h"

    void AWeapon::Fire(const FVector& HitTarget)
    {
        if (FireAnimation)
        {
            WeaponMesh->PlayAnimation(FireAnimation, false);
        }
        if (CasingClass)
        {
            const USkeletalMeshSocket* AmmoEjectSocket = WeaponMesh->GetSocketByName(FName("AmmoEject"));
            if (AmmoEjectSocket)
            {
                FTransform SocketTransform = AmmoEjectSocket->GetSocketTransform(WeaponMesh);
                UWorld* World = GetWorld();
                if (World)
                {
                    World->SpawnActor<ACasing>(CasingClass, SocketTransform.GetLocation(), SocketTransform.GetRotation().Rotator());
                }
            }
        }
    }
    ```

- Blueprints > Weapon > Casings 폴더에 Casing 클래스를 상속받는 BP_Casing 생성
    - static mesh 알맞게 설정
- BP_AssaultRifle에서 CasingClass를 BP_Casing으로 설정

- BP_Casing에 Shell Sound를 Shells_Cue로 설정
    - 여러개의 sound를 cue에서 Random으로 돌린 것
    - attenuation도 적용하자
    