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
    };
    ...
    public:
        virtual void DrawHUD() override;
    private:
        FHUDPackage HUDPackage;
	    void DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter);
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
            if (HUDPackage.CrosshairsCenter)
            {
                DrawCrosshair(HUDPackage.CrosshairsCenter, ViewportCenter);
            }
            if (HUDPackage.CrosshairsLeft)
            {
                DrawCrosshair(HUDPackage.CrosshairsLeft, ViewportCenter);
            }
            if (HUDPackage.CrosshairsRight)
            {
                DrawCrosshair(HUDPackage.CrosshairsRight, ViewportCenter);
            }
            if (HUDPackage.CrosshairsTop)
            {
                DrawCrosshair(HUDPackage.CrosshairsTop, ViewportCenter);
            }
            if (HUDPackage.CrosshairsBottom)
            {
                DrawCrosshair(HUDPackage.CrosshairsBottom, ViewportCenter);
            }
        }
    }
    void ABlasterHUD::DrawCrosshair(UTexture2D* Texture, FVector2D ViewportCenter)
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
                HUD->SetHUDPackage(HUDPackage);
            }
        }
    }
    ```