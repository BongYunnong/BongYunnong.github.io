- 이동을 처리하는 방밥
    1. 클라가 서버에게 이동 요청을 보내면 서버가 이동시키고, 해당 클라와 다른 클라들에게 broadcast
    2. 클라에서 미리 움직이고 서버에게 알리면 클라들에게 알려줌(만약 서버에서 계산한 것을 다시 받았는데 클라의 위치와 다르면 jitter, snap back이 생김)
    3. 보간하기
    4. server-side rewind

## High Ping Warning
- WBP_CharacterOverlay에 HighPingImage 추가
    - HighPingAnimation 애니메이션 생성
        - RenderOpacity Track추가
- CharacterOverlay.h
    ``` C++
    UPROPERTY(meta = (BindWidget))
    class UImage* HighPingImage;
	UPROPERTY(meta = (BindWidgetAnim), Transient)
	UWidgetAnimation* HighPingAnimation;
    ```
- BlasterPlayerController.h
    ``` C++
    protected:
        void HighPingWarning();
        void StopHighPingWarning();
	    void CheckPing(float DeltaTime);
    private:
        float HighPingRunningTime = 0.f;
        UPROPERTY(EditAnywhere)
        float HighPingDuration = 5.f;
	    float PingAnimationRunningTime = 0.f;
	    UPROPERTY(EditAnywhere)
        float CheckPingFrequency = 20.f;
        UPROPERTY(EditAnywhere)
        float HighPingThreshold = 50.f;
    ```
- BlassterPlayerController.cpp
    ``` C++
    #include "Components/Image.h"
    void ABlasterPlayerController::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        SetHUDTime();
        CheckTimeSync(DeltaTime);
        PollInit();
        CheckPing(DeltaTime);
    }
    void ABlasterPlayerController::CheckPing(float DeltaTime)
    {
        HighPingRunningTime += DeltaTime;
        if (HighPingRunningTime > CheckPingFrequency)
        {
            PlayerState = PlayerState == nullptr ? GetPlayerState<APlayerState>() : PlayerState;
            if (PlayerState)
            {
                if (PlayerState->GetPing() * 4 > HighPingThreshold)
                {
                    HighPingWarning();
                    PingAnimationRunningTime = 0.f;
                }
            }
            HighPingRunningTime = 0.0f;
        }
        bool bHighPingAnimationPlaying = BlasterHUD
            && BlasterHUD->CharacterOverlay
            && BlasterHUD->CharacterOverlay->HighPingAnimation
            && BlasterHUD->CharacterOverlay->IsAnimationPlaying(BlasterHUD->CharacterOverlay->HighPingAnimation);
        if (bHighPingAnimationPlaying)
        {
            PingAnimationRunningTime += DeltaTime;
            if (PingAnimationRunningTime > HighPingDuration)
            {
                StopHighPingWarning();
            }
        }
    }
    void ABlasterPlayerController::HighPingWarning()
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->HighPingImage &&
            BlasterHUD->CharacterOverlay->HighPingAnimation;
        if (bHUDValid)
        {
            BlasterHUD->CharacterOverlay->HighPingImage->SetOpacity(1.f);
            BlasterHUD->CharacterOverlay->PlayAnimation(BlasterHUD->CharacterOverlay->HighPingAnimation);
        }
    }
    void ABlasterPlayerController::StopHighPingWarning()
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->HighPingImage &&
            BlasterHUD->CharacterOverlay->HighPingAnimation;
        if (bHUDValid)
        {
            BlasterHUD->CharacterOverlay->HighPingImage->SetOpacity(0.f);
            if (BlasterHUD->CharacterOverlay->IsAnimationPlaying(BlasterHUD->CharacterOverlay->HighPingAnimation))
            {
                BlasterHUD->CharacterOverlay->StopAnimation(BlasterHUD->CharacterOverlay->HighPingAnimation);
            }
        }
    }
    ```
- Ping 테스트하고싶으면 DefaultEngine.ini파일에서
    ``` ini
    [PacketSimulationSettings]
    PktLag = 100
    ```
    - 이렇게 설정


## Local Fire Effect
- Weapon.cpp
    ``` C++
    void AWeapon::Fire(const FVector& HitTarget)
    {
        ...
        if (HasAuthority())
        {
            SpendRound();
        }
    }
    ```
- CombatComponent.h
    ``` C++
    protected:
        void LocalFire(const FVector_NetQuantize& TraceHitTarget);
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::Fire()
    {
        if (CanFire())
        {
            bCanFire = false;
            LocalFire(HitTarget);
            ServerFire(HitTarget);
            if (EquippedWeapon)
            {
                CrosshairShootingFactor = 0.75f;
            }
            StartFireTimer();
        }
    }
    void UCombatComponent::LocalFire(const FVector_NetQuantize& TraceHitTarget)
    {
        if (EquippedWeapon == nullptr) return;
        if (Character && CombatState == ECombatState::ECS_Reloading && EquippedWeapon->GetWeaponType() == EWeaponType::EWT_Shotgun)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire(TraceHitTarget);
            CombatState = ECombatState::ECS_Unoccupied;
            return;
        }
        if (Character && CombatState == ECombatState::ECS_Unoccupied)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire(TraceHitTarget);
        }
    }
    void UCombatComponent::MulticastFire_Implementation(const FVector_NetQuantize& TraceHitResult)
    {
        if (Character && Character->IsLocallyControlled() && !Character->HasAuthority()) return;
        LocalFire(TraceHitResult);
    }
    ```

## Show Weapon WIdget locally
- 이미 Equip하는 것은 서버side에서 처리하기 때문에, Overlay이 되는 것은 클라에서 해줘도 됨
- Weapon.cpp
    ``` C++
    void AWeapon::BeginPlay()
    {
        Super::BeginPlay();
        
        AreaSphere->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
        AreaSphere->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);
        AreaSphere->OnComponentBeginOverlap.AddDynamic(this, &AWeapon::OnSphereOverlap);
        AreaSphere->OnComponentEndOverlap.AddDynamic(this, &AWeapon::OnSphereEndOverlap);
        if (PickupWidget)
        {
            PickupWidget->SetVisibility(false);
        }
    }
    ```
- E키를 눌렀을 때 equip되는 것은 딜레이가 있을 수 있지만, 이건 감수해야함

## Replicate Scatter
- HitScan 무기를 사용해보면 사실 서버와 클라를 비교했을 때 탄착지점이 다른 것을 확인 가능
- Weapon.h
    ``` C++
    UENUM(BlueprintType)
    enum class EFireType :uint8
    {
        EFT_HitScan UMETA(Displayname = "HitScanWeapon"),
        EFT_Projectile UMETA(Displayname = "ProjectileWeapon"),
        EFT_Shotgun UMETA(Displayname = "ShotgunWeapon"),
        EWS_MAX UMETA(Displayname = "DefaultMAX"),
    };
    ...
    public:
	    FVector TraceEndWithScatter(const FVector& HitTarget);
        UPROPERTY(EditAnywhere)
        EFireType FireType;
        UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        bool bUseScatter = false;
    protected:
        // Trace End With Scatter
        UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        float DistanceToSphere = 800.f;
        UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        float SphereRadius = 75.f;
    ```
- Weapon.cpp
    ``` C++
    #include "Kismet/KismetMathLibrary.h"
    FVector AWeapon::TraceEndWithScatter(const FVector& HitTarget)
    {
        const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
        if (MuzzleFlashSocket == nullptr) return FVector();
        FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
        FVector TraceStart = SocketTransform.GetLocation();
        
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
- HitScanWeapon.h
    ``` C++
    protected:
	    //FVector TraceEndWithScatter(const FVector& TraceStart, const FVector& HitTarget);
    private:
        //UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        //bool bUseScatter = false;

        // Trace End With Scatter
        //UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        //float DistanceToSphere = 800.f;
        //UPROPERTY(EditAnywhere, Category = "Weapon Scatter")
        //float SphereRadius = 75.f;
    ```
- HitScanWeapon.cpp
    ``` C++
    // #include "Kismet/KismetMathLibrary.h"
    void AHitScanWeapon::WeaponTraceHit(const FVector& TraceStart, const FVector& HitTarget, FHitResult& OutHit)
    {
        UWorld* World = GetWorld();
        if (World)
        {
            //bUseScatter ? TraceEndWithScatter(TraceStart, HitTarget) :
            FVector End =  TraceStart + (HitTarget - TraceStart) * 1.25f;
            ...
        }
    }
    ```
- CombatComponent.h
    ``` C++
    protected:
        void FireProjectileWeapon();
        void FireHitScanWeapon();
        void FireShotgun();
	    void ShotgunLocalFire(const TArray <FVector_NetQuantize>& TraceHitTargets);
    protected:
    	UFUNCTION(Server, Reliable)
		void ServerShotgunFire(const TArray <FVector_NetQuantize>& TraceHitTargets);
	    UFUNCTION(NetMulticast, Reliable)
		void MulticastShotgunFire(const TArray <FVector_NetQuantize>& TraceHitTargets);

    ```
- CombatComponent.cpp
    ``` C++
    #include "Blaster/Weapon/Shotgun.h"
    void UCombatComponent::Fire()
    {
        if (CanFire())
        {
            bCanFire = false;
            if (EquippedWeapon)
            {
                CrosshairShootingFactor = 0.75f;
                switch (EquippedWeapon->FireType)
                {
                case EFireType::EFT_Projectile:
                    FireProjectileWeapon();
                    break;
                case EFireType::EFT_HitScan:
                    FireHitScanWeapon();
                    break;
                case EFireType::EFT_Shotgun:
                    FireShotgun();
                    break;
                }
            }
            StartFireTimer();
        }
    }
    void UCombatComponent::FireProjectileWeapon()
    {
        if (EquippedWeapon && Character)
        {
            HitTarget = EquippedWeapon->bUseScatter ? EquippedWeapon->TraceEndWithScatter(HitTarget) : HitTarget;
            if(!Character->HasAuthority()) LocalFire(HitTarget);
            ServerFire(HitTarget);
        }
    }
    void UCombatComponent::FireHitScanWeapon()
    {
        if (EquippedWeapon && Character)
        {
            HitTarget = EquippedWeapon->bUseScatter ? EquippedWeapon->TraceEndWithScatter(HitTarget) : HitTarget;
            if(!Character->HasAuthority()) LocalFire(HitTarget);
            ServerFire(HitTarget);
        }
    }
    void UCombatComponent::FireShotgun()
    {
        AShotgun* Shotgun = Cast<AShotgun>(EquippedWeapon);
        if (Shotgun && Character)
        {
            TArray<FVector_NetQuantize> HitTargets;
            Shotgun->ShotgunTraceEndWithScatter(HitTarget, HitTargets);
            if (!Character->HasAuthority()) ShotgunLocalFire(HitTargets);
            ServerShotgunFire(HitTargets);
        }
    }
    void UCombatComponent::LocalFire(const FVector_NetQuantize& TraceHitTarget)
    {
        if (EquippedWeapon == nullptr) return;
        if (Character && CombatState == ECombatState::ECS_Unoccupied)
        {
            Character->PlayFireMontage(bAiming);
            EquippedWeapon->Fire(TraceHitTarget);
        }
    }
    void UCombatComponent::ShotgunLocalFire(const TArray<FVector_NetQuantize>& TraceHitTargets)
    {
        AShotgun* Shotgun = Cast<AShotgun>(EquippedWeapon);
        if (Shotgun == nullptr || Character ==nullptr) return;
        if (CombatState == ECombatState::ECS_Reloading || CombatState == ECombatState::ECS_Unoccupied)
        {
            Character->PlayFireMontage(bAiming);
            Shotgun->FireShotgun(TraceHitTargets);
            CombatState = ECombatState::ECS_Unoccupied;
        }
    }
    void UCombatComponent::ServerShotgunFire_Implementation(const TArray<FVector_NetQuantize>& TraceHitTargets)
    {
        MulticastShotgunFire(TraceHitTargets);
    }

    void UCombatComponent::MulticastShotgunFire_Implementation(const TArray<FVector_NetQuantize>& TraceHitTargets)
    {
        if (Character && Character->IsLocallyControlled() && !Character->HasAuthority()) return;
        ShotgunLocalFire(TraceHitTargets);
    }

    ```
- Shotgun.h
    ``` C++
    public:
        // virtual void Fire(const FVector& HitTarget) override;
	    void ShotgunTraceEndWithScatter(const FVector& HitTarget, TArray<FVector_NetQuantize>& HitTargets);
	    virtual void FireShotgun(const TArray<FVector_NetQuantize>& HitTargets);
    ```
- Shotgun.cpp
    ``` C++
    #include "Kismet/KismetMathLibrary.h"
    void AShotgun::ShotgunTraceEndWithScatter(const FVector& HitTarget, TArray<FVector_NetQuantize>& HitTargets)
    {
        const USkeletalMeshSocket* MuzzleFlashSocket = GetWeaponMesh()->GetSocketByName("MuzzleFlash");
        if (MuzzleFlashSocket == nullptr) return;
        const FTransform SocketTransform = MuzzleFlashSocket->GetSocketTransform(GetWeaponMesh());
        const FVector TraceStart = SocketTransform.GetLocation();

        const FVector ToTargetNormalized = (HitTarget - TraceStart).GetSafeNormal();
        const FVector SphereCenter = TraceStart + ToTargetNormalized * DistanceToSphere;

        for (uint32 i = 0; i < NumberOfPellets; i++)
        {
            const FVector RandVec = UKismetMathLibrary::RandomUnitVector() * FMath::FRandRange(0.f, SphereRadius);
            const FVector EndLoc = SphereCenter + RandVec;
            FVector ToEndLoc = EndLoc - TraceStart;
            ToEndLoc = TraceStart + ToEndLoc * TRACE_LENGTH / ToEndLoc.Size();
            HitTargets.Add(ToEndLoc);
        }
    }
    ```

## Client-Side Prediction
- 클라에서 먼저 이동하고, 서버에게 전송하면 서버가 다시 클라에게 설정해주는 것
    - 클라가 계속 이동하고있으면 서버에서 전송받은 위치로 이동할 때 현재 위치와 맞지 않으면 jitter, Snap back이 생김

- 그럼 이걸 어케 써먹냐? Reconciliation 사용
    - client에서 서버에 보내는 정보를 저장해두고, 서버에서 rpc로 인증받은 것과 다른, 즉 클라이언트에서 서버에 더 보낸 정보가 있으면 이전 정보를 삭제함
    1. Client Moves
    2. Send RPC(Save it)
    3. Receives Processed Response from server
    4. Applies Correction
    5. Discard old movement(already processed by server)
    6. Applies all unprocessed movement
- 언리얼의 Charactermovement가 client-side prediction을 이미 사용하고있음


## Client-Side Predicting Ammo
- Weapon.cpp
    ``` C++
    void AWeapon::Fire(const FVector& HitTarget)
    {
        ...
        //if (HasAuthority())
        //{
            SpendRound();
        //}
    }
    ```
    - 이렇게 해보면 랙이 심할 때 HUD가 jitter되는 것을 확인할 수 있음
- 앞으로 할 것은 Ammo를 Replicate하지 않는 것

- Weapon.h
    ``` C++
    private:
    	//UPROPERTY(EditAnywhere, ReplicatedUsing = OnRep_Ammo)
    	UPROPERTY(EditAnywhere)
	    int32 Ammo;
        //UFUNCTION()
        //void OnRep_Ammo();
        UFUNCTION(Client, Reliable)
        void ClientUpdateAmmo(int32 ServerAmmo);
        UFUNCTION(Client, Reliable)
        void ClientAddAmmo(int32 AmmoToAdd);
        // the number of unprocessed server request for ammo
        // incremented in spendround, decremented in clientUpateAmmo
        int32 Sequence = 0;
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(AWeapon, WeaponState);
        //DOREPLIFETIME(AWeapon, Ammo);
    }
    //void AWeapon::OnRep_Ammo()
    //{
    //    ...
    //}

    void AWeapon::AddAmmo(int32 AmmoToAdd)
    {
        Ammo = FMath::Clamp(Ammo + AmmoToAdd, 0, MagCapacity);
        SetHUDAmmo();
    }
    void AWeapon::SpendRound()
    {
        Ammo = FMath::Clamp(Ammo - 1, 0, MagCapacity);
        SetHUDAmmo();
        if (HasAuthority())
        {
            ClientUpdateAmmo(Ammo);
        }
        else
        {
            ++Sequence;
        }
    }
    void AWeapon::ClientUpdateAmmo_Implementation(int32 ServerAmmo)
    {
        if (HasAuthority()) return;
        Ammo = ServerAmmo;
        --Sequence;
        Ammo -= Sequence;
        SetHUDAmmo();
    }
    void AWeapon::AddAmmo(int32 AmmoToAdd)
    {
        Ammo = FMath::Clamp(Ammo + AmmoToAdd, 0, MagCapacity);
        SetHUDAmmo();
        ClientAddAmmo(AmmoToAdd);
    }
    void AWeapon::ClientAddAmmo_Implementation(int32 AmmoToAdd)
    {
        if (HasAuthority()) return;
        Ammo = FMath::Clamp(Ammo + AmmoToAdd, 0, MagCapacity);
        BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
        if (BlasterOwnerCharacter && BlasterOwnerCharacter->GetCombat() && IsFull())
        {
            BlasterOwnerCharacter->GetCombat()->JumpToShotgunEnd();
        }
        SetHUDAmmo();
    }
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::UpdateAmmoValues()
    {
        ...
        //EquippedWeapon->AddAmmo(-ReloadAmount);
        EquippedWeapon->AddAmmo(ReloadAmount);
    }
    void UCombatComponent::UpdateShotgunAmmoValues()
    {
        ...
        //EquippedWeapon->AddAmmo(-1);
        EquippedWeapon->AddAmmo(1);
        ...
    }
    ```