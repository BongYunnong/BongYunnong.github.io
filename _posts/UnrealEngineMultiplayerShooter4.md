# Weapon Aim Mechanism
- Blaster > PlayerController 폴더에 PlayerController 상속받는 BlasterPlayerController 클래스 생성
- Blaster > HUD 폴더에 HUD 상속받는 BlasterHUD 클래스 생성
- Weapon.h
    ``` C++
    public:
        // Textures for the weapon Crosshairs
        UPROPERTY(EditAnywhere, Category = Crosshairs)
            class UTexture2D* CrosshairsCenter;
        UPROPERTY(EditAnywhere, Category = Crosshairs)
            UTexture2D* CrosshairsLeft;
        UPROPERTY(EditAnywhere, Category = Crosshairs)
            UTexture2D* CrosshairsRight;
        UPROPERTY(EditAnywhere, Category = Crosshairs)
            UTexture2D* CrosshairsTop;
        UPROPERTY(EditAnywhere, Category = Crosshairs)
            UTexture2D* CrosshairsBottom;
    ```
    - BP_AssaultRifle에 각 속성에 알맞은 Texture 넣자
- BlasterHUD.h
    ``` C++
    USTRUCT(BlueprintType)
    struct FHUDPackage
    {
        GENERATED_BODY()
    public:
        class UTexture2D* CrosshairsCenter;
        UTexture2D* CrosshairsLeft;
        UTexture2D* CrosshairsRight;
        UTexture2D* CrosshairsTop;
        UTexture2D* CrosshairsBottom;
	    float CrosshairSpread;
    };
    ...
    public:
        virtual void DrawHUD() override;
    private:
        FHUDPackage HUDPackage;
	    void DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter, FVector2D Spread);
	    UPROPERTY(EditAnywhere)
		float CrosshairSpreadMax = 16.0f;
    public:
        FORCEINLINE void SetHUDPackage(const FHUDPackage& Package) { HUDPackage = Package; };
    ```
- BlasterHUD.cpp
    ``` C++
    #include "BlasterHUD.h"

    void ABlasterHUD::DrawHUD()
    {
        Super::DrawHUD();
        
        FVector2D ViewportSize;
        if (GEngine)
        {
            GEngine->GameViewport->GetViewportSize(ViewportSize);
            const FVector2D ViewportCenter(ViewportSize.X / 2.f, ViewportSize.Y / 2.f);
            
            float SpreadScaled = CrosshairSpreadMax * HUDPackage.CrosshairSpread;
            if (HUDPackage.CrosshairsCenter)
            {
                FVector2D Spread(0.f, 0.f);
                DrawCrosshair(HUDPackage.CrosshairsCenter, ViewportCenter, Spread);
            }
            if (HUDPackage.CrosshairsLeft)
            {
                FVector2D Spread(-SpreadScaled, 0.f);
                DrawCrosshair(HUDPackage.CrosshairsLeft, ViewportCenter, Spread);
            }
            if (HUDPackage.CrosshairsRight)
            {
                FVector2D Spread(SpreadScaled, 0.f);
                DrawCrosshair(HUDPackage.CrosshairsRight, ViewportCenter, Spread);
            }
            if (HUDPackage.CrosshairsTop)
            {
                FVector2D Spread(0.f, SpreadScaled);
                DrawCrosshair(HUDPackage.CrosshairsTop, ViewportCenter, Spread);
            }
            if (HUDPackage.CrosshairsBottom)
            {
                FVector2D Spread(0.f, -SpreadScaled);
                DrawCrosshair(HUDPackage.CrosshairsBottom, ViewportCenter, Spread);
            }
        }
    }
    void ABlasterHUD::DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter, FVector2D Spread)
    {
        const float TextureWidth = Texture->GetSizeX();
        const float TextureHeight = Texture->GetSizeY();
        const FVector2D TextureDrawPoint(
            ViewportCenter.X - (TextureWidth / 2.0f),
            ViewportCenter.Y - (TextureHeight / 2.0f)
        );
        DrawTexture(
            Texture,
            TextureDrawPoint.X,
            TextureDrawPoint.Y,
            TextureWidth,
            TextureHeight,
            0.f,
            0.f,
            1.f,
            1.f,
            FLinearColor::White
        );
    }
    ```
- tip) 애셋 여러개 선택 후 우클릭 > Asset Actions > Bulk Edit via Property Matrix 하면 여러 애셋을 동시에 수정 가능
- Blueprints > HUD에 Blaster HUD를 상속받는 BP_BlasterHUD 생성
- Blueprints > PlayerController에 Blaster PlayerController를 상속받는 BP_BlasterPlayerController 생성
- BP_BlasterGameMode에서 Default HUD를 BP_BlasterHUD로 설정
- BP_BlasterGameMode에서 Default PlayerController를 BP_BlasterPlayerController로 설정

- CombatComponent.h
    ``` C++
    protected:
    	void SetHUDCrosshairs(float DeltaTime);
    private:
    	class ABlasterPlayerController* Controller;
	    class ABlasterHUD* HUD;
            
        // HUD and Crosshairs
        float CrosshairVelocityFactor;
	    float CrosshairInAirFactor;
    ```
- CombatComponent.cpp
    ``` C++
    #include "Blaster/PlayerController/BlasterPlayerController.h"
    #include "Blaster/HUD/BlasterHUD.h"

    void UCombatComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
        SetHUDCrosshairs(DeltaTime);
    }
    void UCombatComponent::SetHUDCrosshairs(float DeltaTime)
    {
        if (Character == nullptr && Character->Controller == nullptr) return;

        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            HUD = HUD == nullptr ? Cast<ABlasterHUD>(Controller->GetHUD()) : HUD;
            if (HUD)
            {
                FHUDPackage HUDPackage;
                if (EquippedWeapon)
                {
                    HUDPackage.CrosshairsCenter = EquippedWeapon->CrosshairsCenter;
                    HUDPackage.CrosshairsLeft = EquippedWeapon->CrosshairsLeft;
                    HUDPackage.CrosshairsRight = EquippedWeapon->CrosshairsRight;
                    HUDPackage.CrosshairsTop = EquippedWeapon->CrosshairsTop;
                    HUDPackage.CrosshairsBottom = EquippedWeapon->CrosshairsBottom;
                }
                else
                {
                    FHUDPackage HUDPackage;
                    HUDPackage.CrosshairsCenter = nullptr;
                    HUDPackage.CrosshairsLeft = nullptr;
                    HUDPackage.CrosshairsRight = nullptr;
                    HUDPackage.CrosshairsTop = nullptr;
                    HUDPackage.CrosshairsBottom = nullptr;
                }
                // Calculate Crrosshair Spread

                // [0, 600] -> [0, 1]
                FVector2D WalkSpeedRange(0.0f, Character->GetCharacterMovement()->MaxWalkSpeed);
                FVector2D VelocityMultiplierRange(0.0f,1.0f);
                FVector Velocity = Character->GetVelocity();
                Velocity.Z = 0.f;
                CrosshairVelocityFactor = FMath::GetMappedRangeValueClamped(WalkSpeedRange, VelocityMultiplierRange, Velocity.Size());
                if (Character->GetCharacterMovement()->IsFalling())
                {
                    CrosshairInAirFactor = FMath::FInterpTo(CrosshairInAirFactor, 2.25f, DeltaTime, 2.25f);
                }
                else
                {
                    CrosshairInAirFactor = FMath::FInterpTo(CrosshairInAirFactor, 0.f, DeltaTime, 30.f);
                }

                HUDPackage.CrosshairSpread = CrosshairVelocityFactor + CrosshairInAirFactor;
                HUD->SetHUDPackage(HUDPackage);
            }
        }
    }
    ```

## Correcting the weapon direction
- CombatComponent.h
    ``` C++
    private:
    	FVector HitTarget;
    ```
- CombatComponent.cpp   
    ``` C++
    void UCombatComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
        SetHUDCrosshairs(DeltaTime);

        if (Character && Character->IsLocallyControlled())
        {
            FHitResult HitResult;
            TraceUnderCrosshairs(HitResult);
            HitTarget = HitResult.ImpactPoint;
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	FVector GetHitTarget() const;
    ```
- BlasterCharacter.cpp
    ``` C++
    FVector ABlasterCharacter::GetHitTarget() const
    {
        if (Combat == nullptr) return FVector();
        return Combat->HitTarget;
    }
    ```
- BlasterAnimInstance.h
    ``` C++
    private:
    	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		FRotator RightHandRotation;

	    UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
	    bool bLocallyControlled;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        if (bWeaponEquipped && EquippedWeapon && EquippedWeapon->GetWeaponMesh() && BlasterCharacter->GetMesh())
        {
            ...
            if (BlasterCharacter->IsLocallyControlled()) 
            {
                bLocallyControlled = true;
                FTransform RightHandTransform = BlasterCharacter->GetMesh()->GetSocketTransform(FName("hand_r"), ERelativeTransformSpace::RTS_World);
                RightHandRotation = UKismetMathLibrary::FindLookAtRotation(RightHandTransform.GetLocation(), RightHandTransform.GetLocation() +BlasterCharacter->GetHitTarget());
            }

            FTransform MuzzleTipTransform = EquippedWeapon->GetWeaponMesh()->GetSocketTransform(FName("MuzzleFlash"), ERelativeTransformSpace::RTS_World);
            FVector MuzzleX(FRotationMatrix(MuzzleTipTransform.GetRotation().Rotator()).GetUnitAxis(EAxis::X));
            DrawDebugLine(GetWorld(), MuzzleTipTransform.GetLocation(), MuzzleTipTransform.GetLocation() + MuzzleX * 1000.0f, FColor::Red);
        }
    }
    ```

- BlasterAnimBP에서, Rotate Root Bone 이후에 Transform (Modify) Bone 노드 연결
    - Bone to Modify는 hand_r
    - Right Hand Rotation 값 연결
    - Replace Existing, Rotation에서 Replace Exisiting, World Space로 설정

- FABRIK이 aimoffset을 받는 것으로 되어있어서 transform modify bone을 사용하면 덮어씌워질 수 있음
    - fabrik cache대신 aimoffset을 사용하고, transform modfiy bone잏후에 fabrik을 적용하자
    
- Rotate Body 부분을 새로만든 FullBodyStateMachine의 State에 잘라서 붙여넣기
    - Full Body StateMachine도 Cache해서 Local To Component에 넣기

- Right hand, Left hand 부분도 새로만든 TransformHans StateMachine의 State에 잘라서 붙여넣기
    - 이것도 Cache

## Zoom While Aiming
- Weapon.h
    ``` C++
    public:
    	// Zoom FOV While Aiming
	    UPROPERTY(EditAnywhere)
		float ZoomedFOV = 30.f;
    	UPROPERTY(EditAnywhere)
		float ZoomInterpSpeed = 20.f;
    public:
    	FORCEINLINE float GetZoomedFOV() const { return ZoomedFOV; }
	    FORCEINLINE float GetZoomInterpSpeed() const { return ZoomInterpSpeed; }
    ```
- CombatComponent.h
    ``` C++
	// Aiming and FOV
	
	// Field of view when not aiming
	float DefaultFOV;
	float CurrentFOV;

	UPROPERTY(EditAnywhere, Category = Combat)
		float ZoomedFOV = 30.f;
	UPROPERTY(EditAnywhere, Category = Combat)
		float ZoomInterpSpeed = 20.f;
	void InterpFOV(float DeltaTime);
    ```
- CombatComponent.cpp
    ``` C++
    #include "Camera/CameraComponent.h"
    void UCombatComponent::BeginPlay()
    {
        Super::BeginPlay();
        if (Character)
        {
            Character->GetCharacterMovement()->MaxWalkSpeed = BaseWalkSpeed;
            if (Character->GetFollowCamera())
            {
                DefaultFOV = Character->GetFollowCamera()->FieldOfView;
			    CurrentFOV = DefaultFOV;
            }
        }
    }

    void UCombatComponent::InterpFOV(float DeltaTime)
    {
        if (EquippedWeapon == nullptr) return;
        if (bAiming)
        {
            CurrentFOV = FMath::FInterpTo(CurrentFOV, EquippedWeapon->GetZoomedFOV(), DeltaTime, EquippedWeapon->GetZoomInterpSpeed());
        }
        else
        {
            CurrentFOV = FMath::FInterpTo(CurrentFOV, DefaultFOV, DeltaTime, ZoomInterpSpeed);
        }
        if (Character && Character->GetFollowCamera())
        {
            Character->GetFollowCamera()->SetFieldOfView(CurrentFOV);
        }
    }
    void UCombatComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
    {
        ...
        if (Character && Character->IsLocallyControlled())
        {
            ...
	        SetHUDCrosshairs(DeltaTime);
            InterpFOV(DeltaTime);
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
    public:
    	FORCEINLINE UCameraComponent* GetFollowCamera() const { return FollowCamera; };
    ```
- 줌을 너무 땡겼을 때 흐리게 보이는게 싫으면 BP_BlasterCharacter의 FollowCamera의 Detail에서, Aperture(F-stop)을 최대로 끌어올리자


## Shirink Crosshairs when aiming
- CombatComponent.h
    ``` C++
    #include "Blaster/HUD/BlasterHUD.h"
    ...
    private:
    	float CrosshairAimFactor;
	    float CrosshairShootingFactor;
    ```
- CombatComponent.cpp
    ``` C++
    //#include "Blaster/HUD/BlasterHUD.h"
    void UCombatComponent::FireButtonPressed(bool bPressed)
    {
        ...
        if (bFireButtonPressed)
        {
            ...
            if (EquippedWeapon)
            {
                CrosshairShootingFactor = 0.75f;
            }
        }
    }
    void UCombatComponent::SetHUDCrosshairs(float DeltaTime)
    {
        ...
        if (Controller)
        {
            ...
            if (HUD)
            {
                ...
                if (bAiming)
                {
                    CrosshairAimFactor = FMath::FInterpTo(CrosshairAimFactor, 0.58f, DeltaTime, 30.0f);
                }
                else
                {
                    CrosshairAimFactor = FMath::FInterpTo(CrosshairAimFactor, 0.0f, DeltaTime, 30.0f);
                }
			    CrosshairShootingFactor = FMath::FInterpTo(CrosshairAimFactor, 0.0f, DeltaTime, 40.0f);
                HUDPackage.CrosshairSpread = 0.5f + CrosshairVelocityFactor + CrosshairInAirFactor - CrosshairAimFactor + CrosshairShootingFactor;
                HUD->SetHUDPackage(HUDPackage);
            }
        }
    }
    ```

- Source > Blaster > Interfaces 폴더에 Unreal Interface를 상속받는 InteractWithCrosshairsInterface를 생성

- BlasterCharacter.h
    ``` C++
    #include "Blaster/Interfaces/InteractWithCrosshairsInterface.h"
    ...
    class BLASTER_API ABlasterCharacter : public ACharacter, public IInteractWithCrosshairsInterface
    ...
    private:
    	void HideCameraIfCharacterClose();
        UPROPERTY(EditAnywhere)
        float CameraThreshold = 200.f;
    ```
- BlasterCharacter.cpp
    ``` C++
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
        GetMesh()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Visibility, ECollisionResponse::ECR_Block);
    }
    void ABlasterCharacter::HideCameraIfCharacterClose()
    {
        if (!IsLocallyControlled()) return;
        if ((FollowCamera->GetComponentLocation() - GetActorLocation()).Size() < CameraThreshold)
        {
            GetMesh()->SetVisibility(false);
            if (Combat && Combat->EquippedWeapon && Combat->EquippedWeapon->GetWeaponMesh())
            {
                Combat->EquippedWeapon->GetWeaponMesh()->bOwnerNoSee = true;
            }
        }
        else
        {
            GetMesh()->SetVisibility(true);
            if (Combat && Combat->EquippedWeapon && Combat->EquippedWeapon->GetWeaponMesh())
            {
                Combat->EquippedWeapon->GetWeaponMesh()->bOwnerNoSee = false;
            }
        }
    }
    void ABlasterCharacter::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        AimOffset(DeltaTime);
        HideCameraIfCharacterClose();
    }
    ```
- CobatComponent.h
    ``` C++
    private:
    	FHUDPackage HUDPackage;
    ```
- CobatComponent.cpp
    ``` C++
    void UCombatComponent::TraceUnderCrosshairs(FHitResult& TraceHitResult)
    {
        ...
        if (bScreenToWorld)
        {
            FVector Start = CrosshairWorldPosition;

            if (Character)
            {
                float DistanceToCharacter = (Character->GetActorLocation() - Start).Size();
                Start += CrosshairWorldDirection * (DistanceToCharacter + 100.0f);
            }
            ...
            if (TraceHitResult.GetActor() && TraceHitResult.GetActor()->Implements<UInteractWithCrosshairsInterface>())
            {
                HUDPackage.CrosshairsColor = FLinearColor::Red;
            }
            else
            {
                HUDPackage.CrosshairsColor = FLinearColor::White;
            }
        }
    }

    void UCombatComponent::SetHUDCrosshairs(float DeltaTime)
    {
        ...
        if (Controller)
        {
            ...
            if (HUD)
            {
                //FHUDPackage HUDPackage;
                ...
            }
        }
    }
    ```
- BlasterHUD.h
    ``` C++
    USTRUCT(BlueprintType)
    struct FHUDPackage
    {
        GENERATED_BODY()
    public:
        ...
        FLinearColor CrosshairsColor;
    };

    UCLASS()
    class BLASTER_API ABlasterHUD : public AHUD
    {
        ...
    private:
    	void DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter, FVector2D Spread, FLinearColor CrosshairColor);
        ...
    }
    ```
- BlasterHUD.cpp
    ``` C++
	// DrawCrosshair(HUDPackage.CrosshairsCenter, ViewportCenter, Spread, HUDPackage.CrosshairsColor);
    // 위와 같은 부분 모두 HUDPackage.CrosshairsColor추가하도록 업데이트
    void ABlasterHUD::DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter, FVector2D Spread, FLinearColor CrosshairColor)
    {
        ...
        DrawTexture(
            ...
            CrosshairColor
        );
    }
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        if (bWeaponEquipped && EquippedWeapon && EquippedWeapon->GetWeaponMesh() && BlasterCharacter->GetMesh())
        {
            ...
            if (BlasterCharacter->IsLocallyControlled()) 
            {
                bLocallyControlled = true;
                FTransform RightHandTransform = BlasterCharacter->GetMesh()->GetSocketTransform(FName("hand_r"), ERelativeTransformSpace::RTS_World);
                FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(RightHandTransform.GetLocation(), RightHandTransform.GetLocation() + BlasterCharacter->GetHitTarget());
                RightHandRotation = FMath::RInterpTo(RightHandRotation, LookAtRotation, DeltaTime, 30.0f);
            }
        }
    }
    ```

## Hit Reaction
- HitReaction 애니메이션을 zero pose에서부터 additive한 animation으로 변경
- Blueprint > Character > Animation 폴더에 AnimMontage 생성
    - Hit_React_add 애니메이션들 추가
    - Preview Base Pose를 Zero_Pose로 설정
    - FromLeft / FromRight / FromBack / FromFront 생성
    - 그룹의 Slot을 HitReactSlot으로 변경
    - Blend Time은 0으로 초기화
- BlasterCharacter.h
    ``` C++
    public:
        UFUNCTION(NetMulticast, Unreliable)
            void MulticastHit();
    protected:
    	void PlayHitReactMontage();
    private:
        UPROPERTY(EditAnywhere, Category = Combat)
        UAnimMontage* HitReactMontage;
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/Character/BlasterCharacter.h"
    ...
    void ABlasterCharacter::PlayHitReactMontage()
    {
        if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

        UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
        if (AnimInstance && HitReactMontage)
        {
            AnimInstance->Montage_Play(HitReactMontage);
            FName SectionName("FromFront");
            AnimInstance->Montage_JumpToSection(SectionName);
        }
    }
    void ABlasterCharacter::MulticastHit_Implementation()
    {
        PlayHitReactMontage();
    }
    ```
- Projectile.cpp
    ``` C++
    void AProjectile::OnHit(UPrimitiveComponent* HitComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
    {
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
		    BlasterCharacter->MulticastHit();
        }
        Destroy();
    }
    ```
- BP_BlasterCharacter에 HitReactMontage넣기

- Projectile.cpp
    ``` C++
    AProjectile::AProjectile()
    {
        ...
        CollisionBox->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Block);
    }
    ```
- BlasterAnimBP에서 WeaponSlot 이후에 HitReactSlot 연결

- 그런데 이렇게 하면 사실 메시를 때리는 것이 아니라 capsule을 때리는 것..
- ObjectType으로 SkeletalMesh 생성(Default는 Block)
    - 이렇게 하면 C++로 작업할 때 ECC_GameTraceChannel1로 봐야해서 가시성이 좋지 않음
    - Blaster.h
        ``` C++
        #define ECC_SkeletalMesh ECollisionChannel::ECC_GameTraceChannel1
        ```
- Projectile.h
    ``` C++
    #include "Blaster/Blaster.h"
    AProjectile::AProjectile()
    {
        ...
        //CollisionBox->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Block);
        CollisionBox->SetCollisionResponseToChannel(ECC_SkeletalMesh, ECollisionResponse::ECR_Block);
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/Blaster.h"
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
        GetMesh()->SetCollisionObjectType(ECC_SkeletalMesh);
    }
    ```
    - 혹시 모르니 BP_BlasterCharacter에섯 SkeletalMesh의 ObjectType을 체크하자

## SmoothRotation and Proxy
- 사실 BlasterAnimBP에 있던 rotate root bone은 멀티플레이 환경에서 좋지 않음
    - 
- BlasterCharacter.h
    ```C++
    protected:
    	void SimProxiesTurn();
    private:
	    bool bRotateRootBone;
        float TurnThreshold = 0.5f;
        FRotator ProxyRotationLastFrame;
        FRotator ProxyRotation;
	    float ProxyYaw;
	    float TimeSinceLastMovementReplication;
	    float CalculateSpeed();
    public:
	    virtual void OnRep_ReplicatedMovement() override;
	    FORCEINLINE bool ShouldRotateRootBone() const { return bRotateRootBone; };
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
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
        HideCameraIfCharacterClose();
    }
    float ABlasterCharacter::CalculateSpeed()
    {
        FVector Velocity = GetVelocity();
        Velocity.Z = 0.f;
	    return Velocity.Size();
    }
    void ABlasterCharacter::AimOffset(float DeltaTime)
    {
        ...
	    float Speed = CalculateSpeed();
        ...
        if (Speed == 0.f && !bIsInAir)	// standing still, not jumping
        {
            bRotateRootBone = true;
            ...
        }
        if (Speed > 0.f || bIsInAir)	// running or jumping
        {
            bRotateRootBone = false;
            ...
        }
        CalculateAO_Pitch();
        ...
    }
    void ABlasterCharacter::SimProxiesTurn()
    {
        if (Combat == nullptr || Combat->EquippedWeapon == nullptr) return;

        bRotateRootBone = false;
        float Speed = CalculateSpeed();
        if (Speed > 0.f)
        {
            TurningInPlace = ETurningInPlace::ETIP_NotTurning;
            return;
        }
        ProxyRotationLastFrame = ProxyRotation;
        ProxyRotation = GetActorRotation();
        ProxyYaw = UKismetMathLibrary::NormalizedDeltaRotator(ProxyRotation, ProxyRotationLastFrame).Yaw;

        if (FMath::Abs(ProxyYaw) > TurnThreshold)
        {
            if (ProxyYaw > TurnThreshold)
            {
                TurningInPlace = ETurningInPlace::ETIP_Right;
            }
            else if (ProxyYaw < -TurnThreshold)
            {
                TurningInPlace = ETurningInPlace::ETIP_Left;
            }
            else
            {
                TurningInPlace = ETurningInPlace::ETIP_NotTurning;
            }
            return;
        }
        TurningInPlace = ETurningInPlace::ETIP_NotTurning;
    }
    void ABlasterCharacter::CalculateAO_Pitch()
    {
        AO_Pitch = GetBaseAimRotation().Pitch;
        if (AO_Pitch > 90.0f && !IsLocallyControlled())
        {
            // map pitch from [270, 360) to [-90, 0)
            FVector2D InRange(270.f, 360.f);
            FVector2D OutRange(-90.f, 0.f);
            AO_Pitch = FMath::GetMappedRangeValueClamped(InRange, OutRange, AO_Pitch);
        }
    }
    void ABlasterCharacter::OnRep_ReplicatedMovement()
    {
        Super::OnRep_ReplicatedMovement();
        SimProxiesTurn();
        TimeSinceLastMovementReplication = 0.f;
    }
    ```
- BlasterAnimInstance.h
    ``` C++
    private:
	    UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
    	bool bRotateRootBone;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bRotateRootBone = BlasterCharacter->ShouldRotateRootBone();
        ...
    }
    ```
- ABP의 FullBody State에서 Blend Pose by bool로 Rotate RootBone 사용 분기

## Automatic Fire
- CombatComponnet.h
    ``` C++
    protected:
    	void Fire();
    private:
        // Automatic Fire
        FTimerHandle FireTimer;
        UPROPERTY(EditAnywhere, Category = Combat)
        float FireDelay = .15f;
        UPROPERTY(EditAnywhere, Category = Combat)
        bool bAutomatic = true;
	    bool bCanFire = true;

        void StartFireTimer();
        void FireTimerFinished();
    ```
- CombatComponent.Cpp
    ``` C++
    #include "TimerManager.h"

    void UCombatComponent::FireButtonPressed(bool bPressed)
    {
        bFireButtonPressed = bPressed;
        if (bFireButtonPressed)
        {
		    Fire();
        }
    }
    void UCombatComponent::Fire()
    {
        if (bCanFire)
        {
		    bCanFire = false;
            ServerFire(HitTarget);
            if (EquippedWeapon)
            {
                CrosshairShootingFactor = 0.75f;
            }
            StartFireTimer();
        }
    }
    void UCombatComponent::StartFireTimer()
    {
        if (EquippedWeapon == nullptr || Character == nullptr) return;
        Character->GetWorldTimerManager().SetTimer(
            FireTimer,
            this,
            &UCombatComponent::FireTimerFinished,
		    EquippedWeapon->FireDelay
        );
    }
    void UCombatComponent::FireTimerFinished()
    {
	    if (EquippedWeapon == nullptr) return;
	    bCanFire = true;
        if (bFireButtonPressed && EquippedWeapon->bAutomatic)
        {
            Fire();
        }
    }
    ```
- Weapon.h
    ``` C++
    public:
        // Automatic Fire
        UPROPERTY(EditAnywhere, Category = Combat)
            float FireDelay = .15f;
        UPROPERTY(EditAnywhere, Category = Combat)
            bool bAutomatic = true;
    ```
## Update Ammo
- CombatComponent.h
    ``` C++
    protected:
        int32 AmountToReload();
    private:
    	void UpdateAmmoValues();
    ```
- CombatComponent.cpp
    ``` C++
    int32 UCombatComponent::AmountToReload()
    {
        if (EquippedWeapon == nullptr) return 0;
        int32 RoomInMag = EquippedWeapon->GetMagCapacity() - EquippedWeapon->GetAmmo();
        if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
        {
            int32 AmountCarried = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
            int32 Least = FMath::Min(RoomInMag, AmountCarried);
            return FMath::Clamp(RoomInMag, 0, Least);
        }
        return 0;
    }
    void UCombatComponent::ServerReload_Implementation()
    {
        if (Character == nullptr && EquippedWeapon != nullptr) return;
        CombatState = ECombatState::ECS_Reloading;
        HandleReload();
    }
    void UCombatComponent::FinishReloading()
    {
        if (Character == nullptr) return;
        if (Character->HasAuthority())
        {
            CombatState = ECombatState::ECS_Unoccupied;
            UpdateAmmoValues();
        }
        if (bFireButtonPressed)
        {
            Fire();
        }
    }
    void UCombatComponent::UpdateAmmoValues()
    {
	    if (Character == nullptr && EquippedWeapon != nullptr) return;
        int32 ReloadAmount = AmountToReload();
        if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
        {
            CarriedAmmoMap[EquippedWeapon->GetWeaponType()] -= ReloadAmount;
            CarriedAmmo = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
        }
        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            Controller->SetHUDCarriedAmmo(CarriedAmmo);
        }
        EquippedWeapon->AddAmmo(-CarriedAmmo);
    }
    ```
- Weapon.h
    ``` C++
    public:
    	void AddAmmo(int32 AmmoToAdd);
    public:
        FORCEINLINE int32 GetAmmo() const { return Ammo; }
        FORCEINLINE int32 GetMagCapacity() const { return MagCapacity; }
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::AddAmmo(int32 AmmoToAdd)
    {
        Ammo = FMath::Clamp(Ammo - AmmoToAdd, 0, MagCapacity);
        SetHUDAmmo();
    }
    ```
## Reload Effect
- Reload 몽타주에 AnimNotify-PlaySound로 Rifle_LOwer_cue와 Rifle_ReloadInsert_Cue 추가
- Weapon.h
    ``` C++
    public:
        UPROPERTY(EditAnywhere)
        class USoundCue* EquipSound;
    ```
- CombatComponent.cpp
    ``` C++
    #include "Sound/SoundCue.h"
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        ...
        if (EquippedWeapon->EquipSound)
        {
            UGameplayStatics::PlaySoundAtLocation(
                this,
                EquippedWeapon->EquipSound,
                Character->GetActorLocation()
            );
        }
        ...
    }

    void UCombatComponent::OnRep_EquippedWeapon()
    {
        if (EquippedWeapon && Character)
        {
            ...
            if (EquippedWeapon->EquipSound)
            {
                UGameplayStatics::PlaySoundAtLocation(
                    this,
                    EquippedWeapon->EquipSound,
                    Character->GetActorLocation()
                );
            }
        }
    }
    ```
- BP_AssaultRifle에 Rifle_Raise_Cue 넣기

- Reload중에 AimOffset끄기
- BlasterAnimInstance.h
    ``` C++
    private:
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		bool bUseAimOffsets;
	    UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		bool bTransformRightHand;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bUseAimOffsets = BlasterCharacter->GetCombatState() != ECombatState::ECS_Reloading;
	    bTransformRightHand = BlasterCharacter->GetCombatState() != ECombatState::ECS_Reloading;
    }
    ```
- AimOffset StateMachine에서 bUseAimOffset을 조건으로 분기 만들기
- Transform Right Hand에서  bTransformRIghtHand도 조건으로 넣기

- CombatComponent.cpp
    ``` C++
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        ...
        if (EquippedWeapon->IsEmpty())
        {
            Reload();
        }
        ...
    }
    void UCombatComponent::FireTimerFinished()
    {
        ...
        if (EquippedWeapon->IsEmpty())
        {
            Reload();
        }
    }
    ```