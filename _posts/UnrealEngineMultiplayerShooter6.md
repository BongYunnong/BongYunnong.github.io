## WeaponAmmo
- ammo는 server에서 관리하는 것이 좋음
- WBP_CharacterOverlay에 WeaponAmmoText, WeaponAmmoAmount 추가
- tip) AActor에는 Owner가 있고, 이것은 Replicated되도록 설정 되어있음
- Weapon.h
    ``` C++
    public:
    	virtual void OnRep_Owner() override;
	    void SetHUDAmmo();
    private:
        UPROPERTY(EditAnywhere, ReplicatedUsing = OnRep_Ammo)
        int32 Ammo;
        UFUNCTION()
        void OnRep_Ammo();
	    void SpendRound();
        UPROPERTY(EditAnywhere)
        int32 MagCapacity;
        UPROPERTY()
        class ABlasterCharacter* BlasterOwnerCharacter;
	    UPROPERTY()
		class ABlasterPlayerController* BlasterOwnerController;
    ```
- Weapon.cpp
    ``` C++
    #include "Blaster/PlayerController/BlasterPlayerController.h"

    void AWeapon::Fire(const FVector& HitTarget)
    {
        ...
        SpendRound();
    }
    void AWeapon::SetHUDAmmo()
    {
        BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
        if (BlasterOwnerCharacter)
        {
            BlasterOwnerController = BlasterOwnerController == nullptr ? Cast<ABlasterPlayerController>(BlasterOwnerCharacter->GetController()) : BlasterOwnerController;
            if (BlasterOwnerController)
            {
                BlasterOwnerController->SetHUDWeaponAmmo(Ammo);
            }
        }
    }
    void AWeapon::SpendRound()
    {
	    Ammo = FMath::Clamp(Ammo - 1, 0, MagCapacity);
	    SetHUDAmmo();
    }
    void AWeapon::OnRep_Ammo()
    {
        BlasterOwnerCharacter = BlasterOwnerCharacter == nullptr ? Cast<ABlasterCharacter>(GetOwner()) : BlasterOwnerCharacter;
        SetHUDAmmo();
    }
    void AWeapon::OnRep_Owner()
    {
        Super::OnRep_Owner();
        SetHUDAmmo();
    }
    ```
- CharacterOverlay.h
    ``` C++
    public:
        UPROPERTY(meta = (BindWidget))
        UTextBlock* WeaponAmmoAmount;
    ```
- BlasterPlayerController.h
    ``` C++
    public:
        void SetHUDWeaponAmmo(int32 Ammo);
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::SetHUDWeaponAmmo(int32 Ammo)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->WeaponAmmoAmount;
        if (bHUDValid)
        {
            FString AmmoText = FString::Printf(TEXT("%d"), Ammo);
            BlasterHUD->CharacterOverlay->WeaponAmmoAmount->SetText(FText::FromString(AmmoText));
        }
    }
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        ...
        EquippedWeapon->SetOwner(Character);
        EquippedWeapon->SetHUDAmmo():
        ...
    }
    ```
    - 이거 안 해도 Owner가 바뀌니까 될 줄 알았는데.. 그건 또 아니네..
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::MulticastElim_Implementation()
    {
        if (BlasterPlayerController)
        {
            BlasterPlayerController->SetHUDWeaponAmmo(0);
        }
    }
    ```
    
- 어떤 무기를 먹으면 기존에 있던 무기 떨어뜨리기
- CombatComponent.Cpp
    ``` C++
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        ...
        if (EquippedWeapon)
        {
            EquippedWeapon->Dropped();
        }
        ...
    }
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::OnRep_Owner()
    {
        Super::OnRep_Owner();
        if (Owner == nullptr)
        {
            BlasterOwnerCharacter = nullptr;
            BlasterOwnerController = nullptr;
        }
        else
        {
            SetHUDAmmo();
        }
    }
    void AWeapon::Dropped()
    {
        ...
        BlasterOwnerCharacter = nullptr;
        BlasterOwnerController = nullptr;
    }
    ```
## CanFire
- Weapon.h
    ``` C++
    public:
    	bool IsEmpty();
    ```
- Weapon.cpp
    ``` C++
    bool AWeapon::IsEmpty()
    {
        return Ammo <= 0;
    }
    ```
- CombatComponent.h
    ``` C++
    private:
	    bool CanFire();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::Fire()
    {
        if (CanFire())
        {
            ...
        }
    }
    bool UCombatComponent::CanFire()
    {
        if (EquippedWeapon == nullptr) return false;
        return !EquippedWeapon->IsEmpty() && bCanFire;
    }
    ```

## Carroed Ammo
- WBP_CharacterOverlay에 CarriedAmmoAmount 추가
- CharacterOverlay.h
    ``` C++
    public:
        UPROPERTY(meta = (BindWidget))
        UTextBlock* CarriedAmmoAmount;
    ```
- BlasterPlayerController.h
    ``` C++
    public:
        void SetHUDCarriedAmmo(int32 Ammo);
    ```
- BlasterPlayerController.cpp
    ``` C++
    void ABlasterPlayerController::SetHUDCarriedAmmo(int32 Ammo)
    {
        BlasterHUD = BlasterHUD == nullptr ? Cast<ABlasterHUD>(GetHUD()) : BlasterHUD;
        bool bHUDValid = BlasterHUD &&
            BlasterHUD->CharacterOverlay &&
            BlasterHUD->CharacterOverlay->CarriedAmmoAmount;
        if (bHUDValid)
        {
            FString AmmoText = FString::Printf(TEXT("%d"), Ammo);
            BlasterHUD->CharacterOverlay->CarriedAmmoAmount->SetText(FText::FromString(AmmoText));
        }
    }
    ```
- Source > Blaster > Weapon 폴더에 WeaponTypes.h라는 헤더파일 생성
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponType : uint8
    {
        EWT_AssaultRifle UMETA(DisplayName = "AssaultRifle"),
        EWT_MAX UMETA(DisplayName = "DefaultMax")
    };
    ```
    - class가 붙은 것은, 이게 안 붙으면 다른 enum끼리 namespace충돌이 일어나기 때문. : uint8이거는 enum의 type

- CombatComponent.h
    ``` C++
    #include "Blaster/Weapon/WeaponTypes.h"
    ...
    private:
	    // Carried ammo for the currently equipped weapon
        UPROPERTY(ReplicatedUsing = OnRep_CarriedAmmo)
        int32 CarriedAmmo;
        UFUNCTION()
        void OnRep_CarriedAmmo();
	    TMap<EWeaponType, int32> CarriedAmmoMap;
        UPROPERTY(EditDefaultsOnly)
        int32 StartingAmmo = 30;
        void InitializeCarriedAmmo();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::BeginPlay()
    {
        Super::BeginPlay();
        if (Character)
        {
            ...
            if (Character->HasAuthority())
            {
                InitializeCarriedAmmo();
            }
        }
    }
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        ...
        if (CarriedAmmoMap.Contains(EquippedWeapon->GetWeaponType()))
        {
            CarriedAmmo = CarriedAmmoMap[EquippedWeapon->GetWeaponType()];
        }
        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            Controller->SetHUDCarriedAmmo(CarriedAmmo);
        }
        ...
    }
    void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(UCombatComponent, EquippedWeapon);
        DOREPLIFETIME(UCombatComponent, bAiming);
        DOREPLIFETIME_CONDITION(UCombatComponent, CarriedAmmo, COND_OwnerOnly);
    }
    void UCombatComponent::OnRep_CarriedAmmo()
    {
        Controller = Controller == nullptr ? Cast<ABlasterPlayerController>(Character->Controller) : Controller;
        if (Controller)
        {
            Controller->SetHUDCarriedAmmo(CarriedAmmo);
        }
    }
    void UCombatComponent::InitializeCarriedAmmo()
    {
        CarriedAmmoMap.Emplace(EWeaponType::EWT_AssaultRifle, StartingAmmo);
    }

    ```
- Weapon.h
    ``` C++
    #include "WeaponTypes.h"
    ...
    private:
        UPROPERTY(EditAnywhere)
        EWeaponType WeaponType;
    public:
	    FORCEINLINE EWeaponType GetWeaponType() const { return WeaponType; }
    ```
- 주의! TMap은 Replicated되지 않음
    - hash알고리즘을 사용하기 때문

## Reload
- ProjectSettings > ActionMappings 에 Reload Action을 R키로 추가

- Source > Blaster > BlasterTypes 폴더에 CombatState.h 추가
    ``` C++
    ENUM(BlueprintType)
    enum class ECombatState : uint8
    {
        ECS_Unoccupied UMETA(DisplayName = "Unoccupied"),
        ECS_Reloading UMETA(DisplayName = "Reloading"),
        ECS_MAX UMETA(DisplayName = "DefaultMAX")
    };
    ```
- BlasterCharacter.h
    ``` C++
    #include "Blaster/BlasterTypes/CombatState.h"
    ...
    public:
    	void PlayReloadMontage();
    protected:
    	void ReloadButtonPressed();
    private:
	    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, meta = (AllowPrivateAccess = "true"))
        class UCombatComponent* Combat;
    	UPROPERTY(EditAnywhere, Category = Combat)
		UAnimMontage* ReloadMontage;
    public:
    	ECombatState GetCombatState() const;
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/Weapon/WeaponTypes.h"

    void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        ...
        PlayerInputComponent->BindAction("Reload", IE_Pressed, this, &ABlasterCharacter::ReloadButtonPressed);
    }
    void ABlasterCharacter::ReloadButtonPressed()
    {
        if (Combat)
        {
            Combat->Reload();
        }
    }
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
            }
            AnimInstance->Montage_JumpToSection(SectionName);
        }
    }
    ECombatState ABlasterCharacter::GetCombatState() const
    {
        if (Combat == nullptr) return ECombatState::ECS_MAX;
        return Combat->CombatState;
    }
    ```
- CombatComponent.h
    ``` C++
    #include "Blaster/BlasterTypes/CombatState.h"
    public:
    	void Reload();
	    UFUNCTION(BlueprintCallable)
	    void FinishReloading();
    protected:
    	UFUNCTION(Server, Reliable)
		void ServerReload();
	    void HandleReload();
    private:
        UPROPERTY(ReplicatedUsing = OnRep_CombatState)
		ECombatState CombatState = ECombatState::ECS_Unoccupied;
        UFUNCTION()
        void OnRep_CombatState();
    ```
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::Reload()
    {
        if (CarriedAmmo > 0 && CombatState != ECombatState::ECS_Reloading)
        {
            ServerReload();
        }
    }
    void UCombatComponent::ServerReload_Implementation()
    {
        if (Character == nullptr) return;
        CombatState = ECombatState::ECS_Reloading;
        HandleReload();
    }
    void UCombatComponent::FinishReloading()
    {
        if (Character == nullptr) return;
        if (Character->HasAuthority())
        {
            CombatState = ECombatState::ECS_Unoccupied;
        }
    }
    void UCombatComponent::HandleReload()
    {
        Character->PlayReloadMontage();
    }
    void UCombatComponent::OnRep_CombatState()
    {
        switch (CombatState)
        {
        case ECombatState::ECS_Reloading:
            HandleReload();
            break;
        }
    }
    void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        ...
        DOREPLIFETIME(UCombatComponent, CombatState);
    }
    ```
- Blueprints > Character > Animation 폴더에 Reload라는 AnimMontage만들고 ReloadRifle 애니메이션 추가
    - Reload 섹션 만들고 Default는 삭제
    - Slot은 WeaponSlot으로 변경
    - BP_BlasterCharacter에 ReloadMontage 설정하기
- BlasterAnimInstance.h
    ``` C++
    private:
    	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		bool bUseFABRIK;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    #include "Blaster/BlasterTypes/CombatState.h"
    ...
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bUseFABRIK = BlasterCharacter->GetCombatState() != ECombatState::ECS_Reloading;
    }
    ```
- BlasterAnimBP에서 Transform Hands를 Transform Right Hand, Trasnform Left Hand로 분리하고, Blend Poses by bool(UseFABRIK)으로 분기 설정
- Reload 애니메이션 몽타주 마지막쯤에 ReloadFinished라는 Notify 생성
    - BlasterAnimBP에서 AnimNotify_ReloadFinished 이벤트로부터 시작해서 BlasterCharacter, BlsterCharacter->Combat을 Valid 체크하고, BlasterCharacter->Combat->FinishReloading 실행
- BP_BlasterCharacter에 Reload 몽타주 설정하기

## Bug Fixing
- CombatComponent.cpp
    ``` C++
    void UCombatComponent::FinishReloading()
    {
        ...
        if (bFireButtonPressed)
        {
            Fire();
        }
    }
    bool UCombatComponent::CanFire()
    {
        if (EquippedWeapon == nullptr) return false;
        return !EquippedWeapon->IsEmpty() && bCanFire && CombatState == ECombatState::ECS_Unoccupied;
    }
    void UCombatComponent::MulticastFire_Implementation(const FVector_NetQuantize& TraceHitResult)
    {
        ...
        if (Character && CombatState == ECombatState::ECS_Unoccupied)
        ...
    }
    void UCombatComponent::OnRep_CombatState()
    {
        switch (CombatState)
        {
        ...
        case ECombatState::ECS_Unoccupied:
            if (bFireButtonPressed)
            {
                Fire();
            }
            break;
        }
    }
    ```