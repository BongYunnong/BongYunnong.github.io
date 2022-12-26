# Create Project
## 프로젝트 만들기
- "Blaster"라는 이름으로 Blank, C++ 프로젝트 생성
- 다운받은 Plugins폴더를 Blaster 프로젝트 폴더에 넣기
- Plugin에서 OnlineSubsystemSteam 활성화
- 위 문서에서 DefaultEngine.ini 복사해서 붙여넣기
    - https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Online/Steam/
    ``` ini
    [/Script/Engine.GameEngine]
    +NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemSteam.SteamNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")

    [OnlineSubsystem]
    DefaultPlatformService=Steam

    [OnlineSubsystemSteam]
    bEnabled=true
    SteamDevAppId=480

    bInitServerOnClient=true

    [/Script/OnlineSubsystemSteam.SteamNetDriver]
    NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
    ```
- DefaultGame.ini
    ``` ini
    [/Script/Engine.GameSession]
    MaxPlayers=100
    ```
- Binaries, Intermediate, Saved 지우고 GenerateProjectFiles
- ContentBrowser에서 Show Plugin Content 활성화

- 기존 맵은 GameStartupMap으로 지정하고 Lobby맵 만들기
- LevelBlueprint에서 WBP Menu Create하고, MenuSetup 실행
    - Path 알맞게 지정하기
- MultiplayerSessionSubsystem.cpp
    ``` C++
    LastSessionSettings->bUseLobbiesIfAvailable = true; 
    ```
- ProjectSetting > PACKAGING > List of maps to include in a packaged build
    - /Game/Maps/Lobby 추가
    - /Game/Maps/GameStartupMap 추가
- 패키징 해보기


## OnlineSession 테스트
- 마켓플레이스의 Military Weapons Silver 프로젝트에 추가
    - UE4 프로젝트에 추가하고, 이걸 UE5로 migrate하는 작업..
- Unreal Learning Kit 추가
- AnimationStarterpack 추가
- mixamo에서 애니메이션 가져오기
    - Mixamo의 애니메이션을 UE4 ThirdPerson 마네킹으로 리타겟팅 시킬것임
    - 리타겟팅 매니저에서 적절한 본 연결
    - Pos 맞추기(Modify pose > Use current Pose)
    - 리타겟팅이 뭔가 이상할 때
        1. Recursively set Translation Retargeting Skeleton
        2. pelvis를 Animation Scaled로 설정
        3. Root를 Animation으로 설정

- UE5부터는 IK Mannequin 애셋에서 IK Retargeting을 할 수 있음
    - 여기에 있는 Chain Name에 따라서 리타겟팅 가능
    1. IK Rig 만들기(적용이 되게 하고싶은 캐릭터 스켈레탈 메시 선택)
    2. Pelvis를 Retarget Root로 설정
    3. Chain 생성(Root, Spine, Head, Clavicle, Arm, Index, Middle, Pinky, Ring, Thumb, Leg(Thigh~ball))
        - bone 개수가 더 적어도 Start~End Bone을 설정하여 리타겟 가능
    4. IK Retargeter를 생성(Source IK Rig 선택)
        - TargetIKRig에 원하는 IK Rig Asset 설정
    5. 뭔가 잘 안 맞는다 싶으면 newpose, editpose로 pose를 바꿔보자

## Chararcter
- Character를 상속받는 BlasterCharacter를 생성
    - Location은 Character라는 폴더를 만들도록 하자
- Blueprints / Character폴더를 만들고, BlasterCharacter를 상속받는 BP_BlasterChracter를 생성
    - 위치, 회전 설정

- BlasterCharacter.h
    ``` C++
    protected:
        void MoveForward(float Value);
        void MoveRight(float Value);
        void Turn(float Value);
        void LookUp(float Value);
    private:
        UPROPERTY(VisibleAnywhere, Category = Camera)
            class USpringArmComponent* CameraBoom;

        UPROPERTY(VisibleAnywhere, Category = Camera)
            class UCameraComponent* FollowCamera;
    ```
- BlasterCharacter.h
    ``` C++
    #include "GameFramework/SpringArmComponent.h"
    #include "Camera/CameraComponent.h"
    #include "GameFramework/CharacterMovementComponent.h"
    
    ABlasterCharacter::ABlasterCharacter()
    {
        PrimaryActorTick.bCanEverTick = true;

        CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));
        CameraBoom->SetupAttachment(GetMesh());
        CameraBoom->TargetArmLength = 600.0f;
        CameraBoom->bUsePawnControlRotation = true;

        FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));
        FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
        FollowCamera->bUsePawnControlRotation = false;

	    bUseControllerRotationYaw = false;
	    GetCharacterMovement()->bOrientRotationToMovement = true;
    }
    
    void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        Super::SetupPlayerInputComponent(PlayerInputComponent);
        PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ABlasterCharacter::Jump);
        PlayerInputComponent->BindAxis("MoveForward", this, &ABlasterCharacter::MoveForward);
        PlayerInputComponent->BindAxis("MoveRight", this, &ABlasterCharacter::MoveRight);
        PlayerInputComponent->BindAxis("Turn", this, &ABlasterCharacter::Turn);
        PlayerInputComponent->BindAxis("LookUp", this, &ABlasterCharacter::LookUp);
    }

    void ABlasterCharacter::MoveForward(float Value)
    {
        if (Controller != nullptr && Value != 0.f)
        {
            const FRotator YawRotation(0.f, Controller->GetControlRotation().Yaw, 0.f);
            const FVector Direction(FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X));
            AddMovementInput(Direction, Value);
        }
    }

    void ABlasterCharacter::MoveRight(float Value)
    {
        if (Controller != nullptr && Value != 0.f)
        {
            const FRotator YawRotation(0.f, Controller->GetControlRotation().Yaw, 0.f);
            const FVector Direction(FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y));
            AddMovementInput(Direction, Value);
        }
    }

    void ABlasterCharacter::Turn(float Value)
    {
        AddControllerYawInput(Value);
    }

    void ABlasterCharacter::LookUp(float Value)
    {
        AddControllerPitchInput(Value);
    }
    ```
- ProjectSettings > Input
    - Action Mapping : Jump
    - Axis Mapping : Move Forward, MoveRight, Turn LookUp설정

## Anim Blueprint
- Blaster > Character 폴더에 AnimInstance를 상속받는 BlasterAnimInstance 클래스 생성
    - 헤더파일 경로 수정
- BlasterAnimInstance.h
    ``` C++
    public:
        virtual void NativeInitializeAnimation() override;
        virtual void NativeUpdateAnimation(float DeltaTime) override;
    private:
        UPROPERTY(BlueprintReadOnly, Category = Character, meta = (AllowPrivateAccess = "true"))
            class ABlasterCharacter* BlasterCharacter;
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
            float Speed;
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
            bool bIsInAir;
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
            bool bIsAccelerating;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    #include "BlasterAnimInstance.h"
    #include "BlasterCharacter.h"
    #include "GameFramework/CharacterMovementComponent.h"

    void UBlasterAnimInstance::NativeInitializeAnimation()
    {
        Super::NativeInitializeAnimation();

        BlasterCharacter = Cast<ABlasterCharacter>(TryGetPawnOwner());
    }

    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        Super::NativeUpdateAnimation(DeltaTime);
        if (BlasterCharacter == nullptr)
        {
            BlasterCharacter = Cast<ABlasterCharacter>(TryGetPawnOwner());
        }
        if (BlasterCharacter == nullptr) return;

        FVector Velocity = BlasterCharacter->GetVelocity();
        Velocity.Z = 0.f;
        Speed = Velocity.Size();

        bIsInAir = BlasterCharacter->GetCharacterMovement()->IsFalling();

        bIsAccelerating = BlasterCharacter->GetCharacterMovement()->GetCurrentAcceleration().Size() > 0.f ? true : false;
    }
    ```
- UnequippedIdleWalkRun라는 BlendSpace 1D 생성
    - Horizontal Axis : Name은 Speed, Maximum Axis Value는 350
    - Idle, Walk, Run 애니메이션 설정

- Blueprints > Character > Animation 폴더 만들고 BlasterAnimInstance 상속받는 BlasterAninBP 생성
    - 알맞은 State, Condition 넣기
    - tip) JumpStart -> Falling은 Automatic Rule Based
    - tip) JumpStop -> IdleWalkRun은 Time Remaining ratio가 0.1보다 작으면 true
    - tip) JumpEnd는 loop 체크해제

- BP_BlasterCharacter에 BlasterAnimBP 설정하기

## Seamless Travel & Lobby
- non-seamless travel
    - clients disconnects from server
    - client reconnects to the same server
    - 언제 일어남?
        - when loading a map for the first time
        - when connecting to a server for the first time
        - when ending a multiplayer game and starting a new one

- seamless travel
    - results in a smoother exerience
    - avoids any reconneciton issues
    - 게임모드에서 bUseSeamlessTravel = true;로 해줘야함
    - transition map 필요

- travel  in multiplayer
    - UWorld::ServerTravel
        - Server Only
        - umps the server to a new level
        - All connected clients will follow
        - Server calls APlayerController::ClientTravel
    - APlayerController::ClinetTravel
        - When called from a client : travel to new server
        - When called from a server : makes the plaer travle to a new map


- LearningKitGames_Showcase맵을 Maps폴더에 복붙
    - BlasterMap으로 이름 변경
    - GameMode를 상속받는 BP_BlasterGameMode를 생성하고, BlasterMap에 적용
    - Default Pawn Class를 BP_BlasterGameMode로 설정


- New Level > Empty Level 만들어서 TransitionMap으로 이름 설정
    - ProjectSetting > Maps & Modes > Transition map 을 TransitionMap으로 설정

- C++ Classes > Blaster > GameMode 폴더에 GameMode를 상속받는 LobbyGameMode 클래스 생성

- LobbyGameMode.h
    ``` C++
    public:
        virtual void PostLogin(APlayerController* NewPlayer) override;
    ```
- LobbyGameMode.cpp
    ``` C++
    #include "LobbyGameMode.h"
    #include "GameFramework/GameStateBase.h"

    void ALobbyGameMode::PostLogin(APlayerController* NewPlayer)
    {
        Super::PostLogin(NewPlayer);
        int32 NumberOfPlayers = GameState.Get()->PlayerArray.Num();
        if (NumberOfPlayers == 2)
        {
            UWorld* World = GetWorld();
            if (World)
            {
                bUseSeamlessTravel = true;
                World->ServerTravel(FString("/Gme/Maps/BlasterMap?listen"));
            }
        }
    }
    ```

- Content > Blueprints > GameModes 폴더 생성 후 LobbyGameMode를 상속받는 BP_LobbyGameMode생성
    - Default Pawn Class를 BP_BlasterCharacter로 설정

- Lobby맵의 WorldSetting에서 GameMode Override를 BP_LobbyGameMode로 설정

## Network Role
- 3가지의 상태
    - Server
    - Client Controlling the Pawn
    - Client Controlling the Pawn
- ENetRole
    - ROLE_Authority : 서버에있으면 Authority
    - ROLE_SimulatedProxy : 다른 사람이 조종하고있는 폰
    - ROLE_AutonomousProxy : 내가 조종하는 폰
    - ROLE_None : 정의되지 않음

- Blaster > HUD 폴더에 UserWidget을 상속받는 OverheadWidget라는 이름의 C++ class 생성
    - Blueprints > HUD 폴더에 OverheadWidget을 상속받는 WBP_OverheadWidget을 생성
        - Canvas Panel 제거하고 Text만 추가 (이름 - DisplayText)
- OverheadWidget.h
    ``` C++
    public:
        UPROPERTY(meta = (BindWidget))
        class UTextBlock* DisplayText;
        
        void SetDisplayText(FString TextToDisplay);

        UFUNCTION(BlueprintCallable)
        void ShowPlayerNetRole(APawn* InPawn);
    protected:
        virtual void OnLevelRemovedFromWorld(ULevel* InLevel, UWorld* InWorld) override;
    ```
    - tip) BindWidget을 하면 이것을 상속받는 Blueprint에서 변수 이름과 똑같은 항목을 찾게 됨
- OverheadWidget.cpp
    ``` C++
    #include "Components/TextBlock.h"

    void UOverheadWidget::SetDisplayText(FString TextToDisplay)
    {
        if(DisplayText)
        {
            DisplayText->SetText(FText::FromString(TextToDisplay));
        }    
    }
    void UOverheadWidget::ShowPlayerNetRole(APawn* InPawn)
    {
        ENetRole LocalRole = InPawn->GetLocalRole();
        FString Role;
        switch(LocalRole)
        {
            case ENetRole::ROLE_Authority:
                Role = FString("Authority");
                break;
            case ENetRole::ROLE_AutonomousProxy:
                Role = FString("Autonomous Proxy");
                break;
            case ENetRole::ROLE_SimulatedProxy:
                Role = FString("Simulated Proxy");
                break;
            case ENetRole::ROLE_None:
                Role = FString("None");
                break;
        }        
        FString LocalRoleString = FString::Printf(TEXT("Local Role : %s"), *Role);
        SetDisplayText(LocalRoleString);
    }
    void UOverheadWidget::OnLevelRemovedFromWorld(ULevel* InLevel, UWorld* InWorld)
    {
        RemoveFromParent();
        Super::OnLevelRemovedFromWorld(InLevel, InWorld);
    }
    ```
- BlasterCharacter.h
    ``` C++
    private:
        UPROPERTY(EditAnywhere, BlueprintReadOnly, meta = (AllowPrivateAccess))
        class UWidgetComponent* OverheadWidget;
    ```
    -  meta = (AllowPrivateAccess) 요거를 해줘야 private임에도 BlueprintReaOnly나 BlueprintReadWrite를 쓸 수 있음
- BlasterCharacter.cpp
    ``` C++
    #include "Components/WidgetComponent.h"
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
        OverheadWidget = CreateDefaultSubobject<UWidgetComponent>(TEXT("OverheadWidget"));
        OverheadWidget->SetupAttachment(RootComponent);
    }
    ```
- BP_BlasterCharacter에서 WidgetClass를 WBP_OverheadWidget으로 설정
    - Widget 위치 변경
    - DrawAtDesiredSize true로 설정
    - BeginPlay에서 OverheadWidget의 Get User Widget Object > Cast to WBP_OverheadWidget을 실행하고, ShowPlayerNetRole 실행(In Pawn은 self)

- tip) RemoteRole을 확인해보면, Client에서는 다 Authority로 보임
    - listen Server에서 자신이 컨트롤하는 폰은 Autonomous Proxy, 다른 폰은 Simulated Proxy로 보임
- tip) PlayerState의 GetPlayerName()을 사용하면 플레이어의 이름을 가져올 수 있음

# Weapon
- Blaster > Weapon 폴더에 Actor를 상속받는 Weapon 클래스 추가

- tip) ctrl + k , o : header파일 열기

- weapon.h
    ``` C++
    UENUM(BlueprintType)
    enum class EWeaponState :uint8
    {
        EWS_Initial UMETA(Displayname = "Initial State"),
        EWS_Equipped UMETA(Displayname = "Equipped"),
        EWS_Dropped UMETA(Displayname = "Dropped"),
        EWS_MAX UMETA(Displayname = "DefaultMAX"),
    };
    ...
    public:	
        AWeapon();
        virtual void Tick(float DeltaTime) override;
    protected:
        virtual void BeginPlay() override;
    private:
        UPROPERTY(VisibleAnywhere, Category = "Weapon Properties")
        USkeletalMeshComponent* WeaponMesh;
        UPROPERTY(VisibleAnywhere, Category = "Weapon Properties")
        class USphereComponent* AreaSphere;

        UPROPERTY(VisibleAnywhere, Category = "Weapon Properties")
        EWeaponState WeaponState;
    ```
- Weapon.cpp
    ``` C++
    #include "Weapon.h"
    #include "Components/SphereComponent.h"
    AWeapon::AWeapon()
    {
        PrimaryActorTick.bCanEverTick = false;
        bReplicates = true;

        WeaponMesh = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("WeaponMesh"));
        SetRootComponent(WeaponMesh);

        WeaponMesh->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Block);
        WeaponMesh->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Ignore);
        WeaponMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);

        AreaSphere = CreateDefaultSubobject<USphereComponent>(TEXT("AreaSphere"));
        AreaSphere->SetupAttachment(RootComponent);
        AreaSphere->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
        AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }

    void AWeapon::BeginPlay()
    {
        Super::BeginPlay();
        
        if (HasAuthority())
        {
            AreaSphere->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
            AreaSphere->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);
        }
    }
    ```
    - AreaSphere는 일단 다 collision을 끄고, 서버에서만 실행이 되도록 만들 예정

- Blueprints > Weapon폴더 만들어서 Weapon 상속받는 BP_Weapon생성
    - SkeletalMesh에 Assualt Rifle 적용
    - Area Sphere 조정

## Pickup Weapon
- UserWidget 상속받은 WBP_PickupWidget 생성
    - Text 추가하고 이름은 "PickupText", Text는 "E - Pick Up"로 설정
- Weapon.h
    ``` C++
    protected:
        UFUNCTION()
        virtual void OnSphereOverlap(class UPrimitiveComponent* OverlappedComp, class AActor* OtherActor, class UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
    private:
        UPROPERTY(VisibleAnywhere, Category = "Weapon Properties")
        class UWidgetComponent* PickupWidget;
    ```
- Weapon.cpp
    ``` C++
    #include "Components/WidgetComponent.h"
    #include "Blaster/Character/BlasterCharacter.h"
    AWeapon::AWeapon()
    {
        ...
	    PickupWidget = CreateDefaultSubobject<UWidgetComponent>(TEXT("PickupWidget"));
	    PickupWidget->SetupAttachment(RootComponent);
    }
    void AWeapon::BeginPlay()
    {
        Super::BeginPlay();
        if (HasAuthority())
        {
            ...
		    AreaSphere->OnComponentBeginOverlap.AddDynamic(this, &AWeapon:: OnSphereOverlap);
        }
        if (PickupWidget)
        {
            PickupWidget->SetVisibility(false);
        }
    }
    void AWeapon::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            PickupWidget->SetVisibility(true);
        }
    }
    ```
- BP_Weapon에서 PickupWidget을 Screen Space, WBP_PickupWidget, DrwaDesiredSize로 설정

- BlasterCharacter.h
    ``` C++
    public:
	    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimePros) const override;
    private:
        UPROPERTY(Replicated)
        class AWeapon* OverlappingWeapon;
    public:
        FORCEINLINE void SetOverlappingWeapon(AWeapon* Weapon) { OverlappingWeapon = Weapon; }
    ```
    - Server에서 Overlap된 Weapon 정보만 Client에게 Replicate되어서 안전함
- BlasterCharacter.cpp
    ``` C++
    #include "Net/UnrealNetwork.h"
        
    void ABlasterCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(ABlasterCharacter, OverlappingWeapon);
    }
    ```
- Weapon.h
    ``` C++
    public:
    	void ShowPickupWidget(bool bShowWidget);
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::OnSphereOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
    {
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter && PickupWidget)
        {
            BlasterCharacter->SetOverlappingWeapon(this);
        }
    }
    void AWeapon::ShowPickupWidget(bool bShowWidget)
    {
        if (PickupWidget)
        {
            PickupWidget->SetVisibility(bShowWidget);
        }
    }
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/Weapon/Weapon.h"
    void ABlasterCharacter::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        if (OverlappingWeapon)
        {
            OverlappingWeapon->ShowPickupWidget(true);
        }
    }
    ```
- 와.. 왜 Binding이 안 됐나 보니까 header에서 class 빼먹은것 땜에였음
- DOREPLIFETIME에 Condition주기
    ``` C++
	DOREPLIFETIME_CONDITION(ABlasterCharacter, OverlappingWeapon, COND_OwnerOnly);
    ```

- Tick은 임시였고, 제대로 만들어보자
- BlasterCharacter.h
    ``` C++
    private:
        UPROPERTY(ReplicatedUsing = OnRep_OverlappingWeapon)
        class AWeapon* OverlappingWeapon;

        UFUNCTION()
        void OnRep_OverlappingWeapon();
    ```
- BlsaterCharacter.cpp
    ``` C++
    void ABlasterCharacter::OnRep_OverlappingWeapon()
    {
        if(OverlappingWeapon)
            OverlappingWeapon->ShowPickupWidget(true);
    }

    void ABlasterCharacter::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
    }
    ```
    - OnRep는 Server에서는 실행되지 않음(서버에서 클라에게 replication될 때만 불리기 때문)
- listen server에서도 실행되게 하자
- BlasterCharacter.h
    ``` C++
    public:
        void SetOverlappingWeapon(AWeapon* Weapon);
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::SetOverlappingWeapon(AWeapon* Weapon)
    {
        OverlappingWeapon = Weapon;
        if (IsLocallyControlled())
        {
            if (OverlappingWeapon)
                OverlappingWeapon->ShowPickupWidget(true);
        }
    }
    ```

### on overlap end
- Weapon.h
    ``` C++
    UFUNCTION()
	virtual void OnSphereEndOverlap(class UPrimitiveComponent* OverlappedComp, class AActor* OtherActor, class UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);
    ```
- Weapon.cpp
    ``` C++
    void AWeapon::OnSphereEndOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
    {
        ABlasterCharacter* BlasterCharacter = Cast<ABlasterCharacter>(OtherActor);
        if (BlasterCharacter)
        {
            BlasterCharacter->SetOverlappingWeapon(nullptr);
        }
    }
    ```
- BlasterCharacter.h
    ``` C++
	UFUNCTION()
	void OnRep_OverlappingWeapon(AWeapon* LastWeapon);
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::OnRep_OverlappingWeapon(AWeapon* LastWeapon)
    {
        if(OverlappingWeapon)
            OverlappingWeapon->ShowPickupWidget(true);
        if (LastWeapon)
            LastWeapon->ShowPickupWidget(false);
    }
    void ABlasterCharacter::SetOverlappingWeapon(AWeapon* Weapon)
    {
        if (OverlappingWeapon)
            OverlappingWeapon->ShowPickupWidget(false);
        OverlappingWeapon = Weapon;
        if (IsLocallyControlled())
        {
            if (OverlappingWeapon)
                OverlappingWeapon->ShowPickupWidget(true);
        }
    }
    ```

## ActorComponent
- Blaster > BlasterComponetns 폴더에 ActorComponent를 상속받은 CombatCoponent생성
- CombatComponent.h
    ``` C++
    class AWeapon;
    ...
    public:
    	friend class ABlasterCharacter;
	    void EquipWeapon(AWeapon* WeaponToEquip);
    private:	
        class ABlasterCharacter* Character;
        class AWeapon* EquippedWeapon;
    ```
- CombatComponent.cpp
    ``` C++
    #include "Blaster/Weapon/Weapon.h"
    #include "Blaster/Character/BlasterCharacter.h"
    #include "Engine/SkeletalMeshSocket.h"

    UCombatComponent::UCombatComponent()
    {
        PrimaryComponentTick.bCanEverTick = false;
    }
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        if (Character == nullptr || WeaponToEquip == nullptr) return;

        EquippedWeapon = WeaponToEquip;
        EquippedWeapon->SetWeaponSate(EWeaponState::EWS_Equipped);
        const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(FName("RightHandSocket"));
        if (HandSocket)
        {
            HandSocket->AttachActor(EquippedWeapon, Character->GetMesh());
        }
        EquippedWeapon->SetOwner(Character);
        EquippedWeapon->ShowPickupWidget(false);
    }
    ```
- BlasterCharacter.h
    ``` c++
    public:
    	virtual void PostInitializeComponents() override;
    protected:
	    void EquipButtonPressed();
    public:
        UPROPERTY(VisibleAnywhere)
        class UCombatComponent* Combat;
    ```
- ProjectSettings > Input에 ActionMapping
    - E키를 Equip으로 매핑

- BlasterCharacter.cpp
    ``` C++
    #include "Blaster/BlasterComponents/CombatComponent.h"
    ABlasterCharacter::ABlasterCharacter()
    {
        Combat = CreateDefaultSubobject<UCombatComponent>(TEXT("CombatComponent"));
        Combat->SetIsReplicated(true);
    }
    void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        ...
        PlayerInputComponent->BindAction("Equip", IE_Pressed, this, &ABlasterCharacter::EquipButtonPressed);
    }
    void ABlasterCharacter::EquipButtonPressed()
    {
	    if (Combat && HasAuthority())
        {
            Combat->EquipWeapon(OverlappingWeapon);
        }
    }
    ```
- skeletalmesh의 hand_r에 socket달기
- Weapon.h
    ``` C++
    public:
        FORCEINLINE void SetWeaponSate(EWeaponState State) { WeaponState = State; }
    ```

## RPC
- BlasterCharacter.h
    ``` C++
	UFUNCTION(Server, reliable)
	void ServerEquipButtonPressed();
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::EquipButtonPressed()
    {
        if (Combat)
        {
            if (HasAuthority())
            {
                Combat->EquipWeapon(OverlappingWeapon);
            }
            else 
            {
                ServerEquipButtonPressed();
            }
        }
    }
    void ABlasterCharacter::ServerEquipButtonPressed_Implementation()
    {
        if (Combat)
        {
            Combat->EquipWeapon(OverlappingWeapon);
        }
    }
    ```
- Weapon.h
    ``` C++
    public:
    	virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    private:
        UPROPERTY(ReplicatedUsing = OnRep_WeaponState, VisibleAnywhere, Category = "Weapon Properties")
        EWeaponState WeaponState;
        UFUNCTION()
        void OnRep_WeaponState();
    public:
	    void SetWeaponSate(EWeaponState State);
    	FORCEINLINE USphereComponent* GetAreaSphere() const { return AreaSphere; }
    ```
- Weapon.cpp
    ``` C++
    #include "Net/UnrealNetwork.h"

    void AWeapon::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(AWeapon, WeaponState);

    }
    void AWeapon::OnRep_WeaponState()
    {
        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ShowPickupWidget(false);
            break;
        }
    }
    void AWeapon::SetWeaponSate(EWeaponState State)
    {
        WeaponState = State;

        switch (WeaponState)
        {
        case EWeaponState::EWS_Equipped:
            ShowPickupWidget(false);
            AreaSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
            break;
        }
    }
    ```
- CombatComponent.cpp
    ``` C++
    #include "Components/SphereComponent.h"
    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        if (Character == nullptr || WeaponToEquip == nullptr) return;

        EquippedWeapon = WeaponToEquip;
        EquippedWeapon->SetWeaponSate(EWeaponState::EWS_Equipped);
        const USkeletalMeshSocket* HandSocket = Character->GetMesh()->GetSocketByName(FName("RightHandSocket"));
        if (HandSocket)
        {
            HandSocket->AttachActor(EquippedWeapon, Character->GetMesh());
        }
        EquippedWeapon->SetOwner(Character);
    }
    ```
- tip) SetOwner는 Owner를 설정하는 함수인데, Owner는 On_Rep를 호출하도록 되어있고, On_Rep는 가상함수라서 overriding 가능

## Equip pose
- BlasterAnimInstance.h
    ``` C++
	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		bool bWeaponEquipped;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bWeaponEquipped = BlasterCharacter->IsWeaponEquipped();
    }

    ```
- BlasterCharacter.h
    ``` C++
	bool IsWeaponEquipped();
    ```
- BlasterCharacter.cpp
    ``` C++
    bool ABlasterCharacter::IsWeaponEquipped()
    {
        return (Combat && Combat->EquippedWeapon);
    }
    ```
- CombatComponent.h
    ``` C++
    public:
    	virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    private:
        UPROPERTY(Replicated)
        class AWeapon* EquippedWeapon;
    ```
- CombatComponent.cpp
    ``` C++
    #include "Net/UnrealNetwork.h"
    void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(UCombatComponent, EquippedWeapon);
    }
    ```

- BlasterAnimBP에서 WepaonEquipped boolean 값을 바탕으로 Blend Poses by bool

## Crouching
- Action Mapping
    - Crouch - Left Shift, Right Shift

- BlasterCharacter.h
    ``` C++
	void CrouchButtonPressed();
    ```
- BlasterCharacter.cpp
    ``` C++
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
       	GetCharacterMovement()->NavAgentProps.bCanCrouch = true;
    }
    void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        ...
        PlayerInputComponent->BindAction("Crouch", IE_Pressed, this, &ABlasterCharacter::CrouchButtonPressed);
    }
    void ABlasterCharacter::CrouchButtonPressed()
    {
        if (bIsCrouched)
        {
            UnCrouch();
        }
        else
        {
            Crouch();
        }
    }
    ```
- Character에 이미 Crouch라는 함수, bIsCrouched라는 변수가 있음
    - CharacterMovementComponent에서 CanMove를 true로 바꿔주자

- BlasterAnimInstance.h
    ``` C++
	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
	bool bIsCrouched;
    ```
- BlasterAnimInstance.cpp
    ``` C++
	bIsCrouched = BlasterCharacter->bIsCrouched;
    ```

## Aim
- Action Mapping
    - Aim - Right Mouse Button
- BlasterCharacter.h
    ``` C++
    protected:
        void AimButtonPressed();
        void AimButtonReleased();
    public:
    	bool IsAiming();
    ```
- BlasterCharacter.cpp
    ``` C++
    void ABlasterCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
    {
        ...
        PlayerInputComponent->BindAction("Aim", IE_Pressed, this, &ABlasterCharacter::AimButtonPressed);
        PlayerInputComponent->BindAction("Aim", IE_Released, this, &ABlasterCharacter::AimButtonReleased);
    }
    void ABlasterCharacter::AimButtonPressed()
    {
        if (Combat)
        {
            Combat->SetAiming(true);
        }
    }

    void ABlasterCharacter::AimButtonReleased()
    {
        if (Combat)
        {
            Combat->SetAiming(false);
        }
    }
    bool ABlasterCharacter::IsAiming()
    {
        return (Combat && Combat->bAiming);
    }
    ```
- CombatComponent.h
    ``` C++
    protected:
        void SetAiming(bool bIsAiming);
        UFUNCTION(Server, Reliable)
        void ServerSetAiming(bool bIsAiming);
    private:
	    UPROPERTY(Replicated)
    	bool bAiming;
            
        UPROPERTY(EditAnywhere)
        float BaseWalkSpeed;
        UPROPERTY(EditAnywhere)
        float AimWalkSpeed;
    ```
- CombatComponent.cpp
    ``` C++
    UCombatComponent::UCombatComponent()
    {
        PrimaryComponentTick.bCanEverTick = false;

        BaseWalkSpeed = 600.0f;
        AimWalkSpeed = 450.0f;
    }
    void UCombatComponent::BeginPlay()
    {
        Super::BeginPlay();
        if (Character)
        {
            Character->GetCharacterMovement()->MaxWalkSpeed = BaseWalkSpeed;
        }
    }
    void UCombatComponent::SetAiming(bool bIsAiming)
    {
        bAiming = bIsAiming;
        ServerSetAiming(bIsAiming);
        if (Character)
        {
            Character->GetCharacterMovement()->MaxWalkSpeed = bIsAiming ? AimWalkSpeed : BaseWalkSpeed;
        }
    }
    void UCombatComponent::ServerSetAiming_Implementation(bool bIsAiming)
    {
        bAiming = bIsAiming;
        if (Character)
        {
            Character->GetCharacterMovement()->MaxWalkSpeed = bIsAiming ? AimWalkSpeed : BaseWalkSpeed;
        }
    }
    void UCombatComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        ...
        DOREPLIFETIME(UCombatComponent, bAiming);
    }
    ```
- BlasterAnimInstance.h
    ``` C++
	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		bool bIsAiming
    ```
- BlasterAnimInstance.cpp
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        bIsAiming = BlasterCharacter->IsAiming();
    }
    ```

## Running
- "EquippedRun"이라는 이름으로 Blend space 생성
    - Horizontal Axis : YawOffset, -180~180
    - Vertical Axis : Lean, -180~180
- Jog Forward기반으로 Root를 회전시켜서 새로운 애니메이션 생성(Jog_Fwd_Lean_L, Jog_Fwd_Lean_R)
    - 이런식으로 Forward, Rt, Lt, Backward 다 Lean_L, Lean_R만들어서 EquippedRun Blendspace에 적용

- BlasterAnimInstance.h
    ``` C++
	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
	float YawOffset;
	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
	float Lean;

	FRotator CharacterRotationLastFrame;
	FRotator CharacterRotation;
	FRotator DeltaRotation;
    ```
- BlasterAnimInstance.cpp
    ``` C++
    #include "Kismet/KismetMathLibrary.h"
        
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        FRotator AimRotation = BlasterCharacter->GetBaseAimRotation();
        FRotator MovementRotation = UKismetMathLibrary::MakeRotFromX(BlasterCharacter->GetVelocity());
        FRotator DeltaRot = UKismetMathLibrary::NormalizedDeltaRotator(MovementRotation, AimRotation);
        DeltaRotation = FMath::RInterpTo(DeltaRotation, DeltaRot, DeltaTime, 6.f);
        YawOffset = DeltaRotation.Yaw;
        
        CharacterRotationLastFrame = CharacterRotation;
        CharacterRotation = BlasterCharacter->GetActorRotation();
        const FRotator Delta = UKismetMathLibrary::NormalizedDeltaRotator(CharacterRotation, CharacterRotationLastFrame);
        const float Target = Delta.Yaw / DeltaTime;
        const float Interp = FMath::FInterpTo(Lean, Target, DeltaTime, 6.f);
        Lean = FMath::Clamp(Interp, -90.f, 90.f);
    }
    ```
    - tip) Pawn에는 GetBaseAimRotation라는 함수가 있음
    - tip) AnimationBlendSpace에서 Smoothing을 주면 backward에서 글리칭 있음 -> 코드에서 아예 DeltaRotation이라는 FRotator 변수를 만들어서 lerp
- CombatComponent.h
    ``` C++
    protected:
    	UFUNCTION()
		void OnRep_EquippedWeapon();
    private:
        UPROPERTY(ReplicatedUsing = OnRep_EquippedWeapon)
        class AWeapon* EquippedWeapon;
    ```
- CombatComponent.cpp
    ``` C++
    #include "GameFramework/CharacterMovementComponent.h"

    void UCombatComponent::EquipWeapon(AWeapon* WeaponToEquip)
    {
        ...
        Character->GetCharacterMovement()->bOrientRotationToMovement = false;
        Character->bUseControllerRotationYaw = true;
    }
    void UCombatComponent::OnRep_EquippedWeapon()
    {
        if (EquippedWeapon && Character)
        {
            Character->GetCharacterMovement()->bOrientRotationToMovement = false;
            Character->bUseControllerRotationYaw = true;
        }
    }
    ```

## CrouchWalking & Aiming
- BlendSpace 1D로 "CrouchWalking" 생성
    - Horizontal Axis : YawOffset, -180~180
- BlendSpace 1D로 "AimWalk"생성
    - Horizontal Axis : YawOffset, -180~180
- BlendSpace 1D로 "CrouchAimWalk"생성
    - Horizontal Axis : YawOffset, -180~180

- Aim Offset
    - 각 aim에 맞는 1프레임 애니메이션 추출
    - additive animation으로 만들기 : Asset Details에 AdditiveAnimType을 mesh type, Base pose type을 SelectedAnimation frame으로 설정 후 원하는 base animation을 설정

- AimOffset 애셋을 생성해서 "HipAimOffset"이라 명명
    - Horizontal Axis : Yaw, -90~90
    - Vertical Axis : Pitch, -90~90
    - AdditiveSettings > Preview Base Pose : Zero_Pos
- AimOffset 애셋을 생성해서 "AimAimOffset"이라 명명
    - 반복

- Equipped StateMachine을 Cache하고, cache pose를 layeredblendperbone에 연결
    - Layer setup > index[0] > Branch Filters > index[0] > BoneName을 spine_01로 설정

- BlasterCharacter.h
    ``` C++
    protected:
    	void AimOffset(float DeltaTime);
    private:
        float AO_Yaw;
        float AO_Pitch;
    public:
    	FORCEINLINE float GetAO_Yaw() const { return AO_Yaw; };
	    FORCEINLINE float GetAO_Pitch() const { return AO_Pitch; };
    ```
- BlasterCharacter.cpp
    ``` C++
    #include "Kismet/KismetMathLibrary.h"

    void ABlasterCharacter::Tick(float DeltaTime)
    {
        Super::Tick(DeltaTime);
        AimOffset(DeltaTime);
    }

    void ABlasterCharacter::AimOffset(float DeltaTime)
    {
        if (Combat && Combat->EquippedWeapon == nullptr) return;
        
        FVector Velocity = GetVelocity();
        Velocity.Z = 0.f;
        float Speed = Velocity.Size();
        bool bIsInAir = GetCharacterMovement()->IsFalling();

        if (Speed == 0.f && !bIsInAir)	// standing still, not jumping
        {
            FRotator CurrentAimRotation = FRotator(0.f, GetBaseAimRotation().Yaw, 0.f);
            FRotator DeltaAimRotation = UKismetMathLibrary::NormalizedDeltaRotator(CurrentAimRotation,StartingAimRotation);
            AO_Yaw = DeltaAimRotation.Yaw;
            bUseControllerRotationYaw = false;
        }
        if (Speed > 0.f || bIsInAir)	// running or jumping
        {
            StartingAimRotation = FRotator(0.f, GetBaseAimRotation().Yaw, 0.f);
            AO_Yaw = 0.f;
            bUseControllerRotationYaw = true;
        }
	    AO_Pitch = GetBaseAimRotation().Pitch;
        if (AO_Pitch > 90.0f && !IsLocallyControlled())
        {
            // map pitch from [270, 360) to [-90, 0)
            FVector2D InRange(270.f, 360.f);
            FVector2D OutRange(-90.f, 0.f);
            AO_Pitch = FMath::GetMappedRangeValueClamped(InRange, OutRange, AO_Pitch);
        }
    }
    ```
- BlasterAnimInstance.h
    ``` C++
    private:
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
            float AO_Yaw;
        UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
            float AO_Pitch;
    ```
- BlasterAnimInstance.h
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        AO_Yaw = BlasterCharacter->GetAO_Yaw();
    }
    ```

- AimOffset에 AO_Yaw, AO_Pitch 값 넘기기

- 주의! : pitch가 locally controlled일때는 -90~90으로 나오는데 아닐때는 270~360,0~90으로 나올 수 있다.
    - CharacterMovement의 GetPackedAngles 함수를 보면 pitch값을 압축하기 위해 unsigned 값으로 넘겨주는 것 확인 가능
    
## FABRIK IK
- BlasterCharacter.h
    ``` C++
    public:
    	AWeapon* GetEquippedWeapon();
    ```
- BlasterCharacter.cpp
    ``` C++
    AWeapon* ABlasterCharacter::GetEquippedWeapon()
    {
        if (Combat == nullptr) return nullptr;
        return Combat->EquippedWeapon;
    }
    ```
- BlasterAnimInstance.h
    ``` C++
    private:
		class AWeapon* EquippedWeapon;

    	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		FTransform LeftHandTransform
    ```
- BlasterAnimInstance.cpp
    ``` C++
    #include "Blaster/Weapon/Weapon.h"
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
    	EquippedWeapon = BlasterCharacter->GetEquippedWeapon();
        ...
        if (bWeaponEquipped && EquippedWeapon && EquippedWeapon->GetWeaponMesh() && BlasterCharacter->GetMesh())
        {
            LeftHandTransform = EquippedWeapon->GetWeaponMesh()->GetSocketTransform(FName("LeftHandSocket"), ERelativeTransformSpace::RTS_World);
            FVector OutPosition;
            FRotator OutRotation;
            BlasterCharacter->GetMesh()->TransformToBoneSpace(FName("hand_r"), LeftHandTransform.GetLocation(), FRotator::ZeroRotator, OutPosition, OutRotation);
            LeftHandTransform.SetLocation(OutPosition);
            LeftHandTransform.SetRotation(FQuat(OutRotation));
        }
    }
    ```
- Weapon.h
    ``` C++
    public:
    	FORCEINLINE USkeletalMeshComponent* GetWeaponMesh() const { return WeaponMesh; }
    ```

- AimOffset Cache pose를 가져와서 FABRIK 노드에 연결
    - Effector Target : hand_r
    - Effector Transform Space : Bone Space
    - Effector Rotation Source : No Change
    - Tip Bone : hand_l
    - Root Bone : upperarm_l

## InPlace Turn
- Idle, Turn Right, Turn Left 상태를 enum으로 만들고싶음
    - 근데 header파일에서 만들면 다른 파일에서 접근을 할 때 해당 헤더 파일을 include해서 컴파일 시간이 늘어남
    - 헤더 파일을 직접 만들어서 거기에서 사용하자
        - Add File을 하는데, Location이 자주 엇나가니 계속 체크 필요
        - Source/Blaster/BlasterTypes폴더에 TurningInPlace.h 추가
        - RegenerateProject필요
- TurningInPlace.h
    ``` C++
    #pragma once
    UENUM(BlueprintType)
    enum class ETurningInPlace :uint8
    {
        ETIP_Left UMETA(Displayname = "Turning Left"),
        ETIP_Right UMETA(Displayname = "Turning Right"),
        ETIP_NotTurning UMETA(Displayname = "Not Turning"),

        ETIP_MAX UMETA(Displayname = "DefaultMax")
    };
    ```
- BlasterAnimInstance.h
    ``` C++
    #include "Blaster/BlasterTypes/TurningInPlace.h"
    private:
    	UPROPERTY(BlueprintReadOnly, Category = Movement, meta = (AllowPrivateAccess = "true"))
		ETurningInPlace TurningInPlace;


    ```
- BlasterAnimInstance.cpp   
    ``` C++
    void UBlasterAnimInstance::NativeUpdateAnimation(float DeltaTime)
    {
        ...
        TurningInPlace = BlasterCharacter->GetTurningInPlace();
        ...
    }
    ```
- BlasterCharacter.h
    ``` C++
    #include "Blaster/BlasterTypes/TurningInPlace.h"
    private:
	    float InterpAO_Yaw;
    	ETurningInPlace TurningInPlace;
        void TurnInPlace(float DeltaTime);
    public:
	    FORCEINLINE ETurningInPlace GetTurningInPlace() const { return TurningInPlace; };
    ```
- BlasterCharacter.cpp
    ``` C++
    ABlasterCharacter::ABlasterCharacter()
    {
        ...
    	TurningInPlace = ETurningInPlace::ETIP_NotTurning;
    }
    void ABlasterCharacter::AimOffset(float DeltaTime)
    {
        ...
        if (Speed == 0.f && !bIsInAir)	// standing still, not jumping
        {
            ...
            if (TurningInPlace == ETurningInPlace::ETIP_NotTurning)
            {
                InterpAO_Yaw = AO_Yaw;
            }
            bUseControllerRotationYaw = faalse;
            TurnInPlace(DeltaTime);
        }
        if (Speed > 0.f || bIsInAir)	// running or jumping
        {
            ...
            TurningInPlace = ETurningInPlace::ETIP_NotTurning;
        }
        ...
    }
    void ABlasterCharacter::TurnInPlace(float DeltaTime)
    {
        if (AO_Yaw > 90.0f)
        {
            TurningInPlace = ETurningInPlace::ETIP_Right;
        }
        else if (AO_Yaw < -90.0f)
        {
            TurningInPlace = ETurningInPlace::ETIP_Left;
        }
        if (TurningInPlace != ETurningInPlace::ETIP_NotTurning)
        {
            InterpAO_Yaw = FMath::FInterpTo(InterpAO_Yaw, 0.f, DeltaTime, 10.f);
            AO_Yaw = InterpAO_Yaw;
            if (FMath::Abs(AO_Yaw) < 15.f)
            {
                TurningInPlace = ETurningInPlace::ETIP_NotTurning;
			    StartingAimRotation = FRotator(0.f, GetBaseAimRotation().Yaw, 0.f);
            }
        }
    }
    ```
    
- TurnLeft State에, Idle 로직을 복붙하고 layeredblendperbone에 연결
    - Layer setup > index[0] > Branch Filters > index[0] > BoneName을 spine_01로 설정
- Rotate Root Bone노드에 AO Yaw 값 넣기
    - tip) 더 자연스럽게 하기 위해 details > yaw scale bias clamp에 interp result를 true로 할 수 있음


## ETC
- 카메라가 다른 클라이언트 캐릭터에 의해 자꾸 줌 인 되는 것 막기
    - BlasterCharacter.cpp
        ``` C++
        GetCapsuleComponent()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
        GetMesh()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
        ```

- tip) character의 detail패널에 보면 Net Update Frequency가 있음
    - faster pace shooter게임에서는 update frequency를 66, min update frequency를 33으로 함
    - BlasterCharacter.cpp
        ``` C++
        ABlasterCharacter::ABlasterCharacter()
        {
            ...
            NetUpdateFrequency = 66.f;
            MinNetUpdateFrequency = 33.f;
        }
        ```
- DefaultEngine.ini에서 Net Rate를 지정할수도 있음
    ``` ini
    [/Script/OnlineSubsystemUtils.IpNetDriver]
    NetServerMaxTickRate=60
    ```

- CharacterMovement에서 Rotation Rate를 설정할수도 있다.
    - BasicChararcter.cpp
        ``` C++
	    GetCharacterMovement()->RotationRate = FRotator(0.f, 0.f, 850.f);
        ```

- Jump를 override할 수 있다.
    - BlasterChararcter.h
        ``` C++
	    virtual void Jump() override;
        ```
    - BlasterCharacter.cpp
        ``` C++
        void ABlasterCharacter::Jump()
        {
            if (bIsCrouched)
            {
                UnCrouch();
            }
            Super::Jump();
        }
        ```
- 왼쪽 앞, 오른쪽 뒤로 움직일 때 애니메이션이 매우 이상
    - Jog_Fwd_45_Left(Root를 오른쪽으로 45, spine을 쪽으로 45), Jog_Bwd_45_Right(Root를 왼쪽으로 45, spine을 오른쪽으로 45) 생성
    - EquippedRun에서 Grid Divisions를 8로 변경
    - Jog_lt와 Jog_Fwd 사이, Jog_rt와 Jog_Bwd 사이에 각 애니메이션 삽입(Lean은 필요없음)

- sound는 기본적으로 replication이 됨
    - animation에서 플레이되기 때문

- sound attenuation : 플레이어가 사운드에서 멀어져감에 따라 그 볼륨을 줄여주는 기능

- sync marker : 길이가 달라도 연관성이 있는 애니메이션간의 동기화 상태를 유지시켜줌
    - 걷기와 달리기 애니메이션의 길이가 매우 다르다면..? -> 주요 애니메이션 노드 하나를 Leader로 삼고 다른 모든 애니메이션은 단순히 길이에 스케일을 적용하여 일치시킴
