---
title: "언리얼 디버깅"
permalink: /posts/Unreal/
excerpt: "언리얼 디버깅"
last_modified_at: 2022-12-01T15:37:00-04:00
toc: true
---

# 언리얼 디버깅
## UE_LOG
``` C++
UE_LOG(LogTEmp, Log, TEXT("원하는 메시지 입력"))
```
- ![image](https://user-images.githubusercontent.com/11372675/205002807-8486caf2-ee44-4a0d-a892-7daa358d1bda.png)
- 에디터의 OutputLog에서 확인 가능
    - ![image](https://user-images.githubusercontent.com/11372675/205002640-59308178-4173-4200-a3b5-36062a8cc380.png)

## GEngine->AddOnScreenebugMessage
- 원형
    ``` C++
    void AddOnScreenDebugMessage
    (
        uint64 key,
        float TimeToDisplay,
        FColor DisplayColor,
        const FString& DebugMessage,
        bool bNewerOnTop,
        const FVector2D& TextScale
    )
    ```
    - key : -1을 넘기면 기존 메시지를 덮어쓰지 않는 것, 0이면 덮어쓰는 것
    - TimeToDisplay : 얼마나 오래 메시지를 띄울 것인가
    - DisplayColor : 메시지 색상
    - DebugMessage : 출력할 메시지
- 사용
    ``` C++
    GEngine->AddOnScreenDebugMessage(-1, 1.0f, FColor::Blue, TEXT("원하는 메시지"));
    ```
- ![image](https://user-images.githubusercontent.com/11372675/205003333-38bee5be-6e13-45b3-833e-f468628705af.png)

- 게임을 플레이하면 좌측 상단에서 확인 가능
    - ![image](https://user-images.githubusercontent.com/11372675/205003146-1c0f2003-50b7-4187-88df-d5305d6f7daf.png)

- tip) GEngine에 마우스 커서를 가져다대면 아래와 같은 설명이 나온다.
    > *Global engine pointer. Can be 0 so don't use without checking
    - 그렇다고 말씀하시니 null 체크를 하도록 하자
    ``` C++
    if(GEngine) GEngine->AddOnScreenDebugMessage(-1, 1.0f, FColor::Blue, TEXT("원하는 메시지"));
    ```


## 중단점으로 디버깅 해보자
1. 중단점 설정하기
    - ![image](https://user-images.githubusercontent.com/11372675/204983821-97d9950b-292e-4f7c-a7e7-0ec29961a1f6.png)
    - 원하는 줄을 클릭하여 텍스트 커서를 이동시키고 "F9"를 눌러 중단점을 활성화/비활성화
        - 그냥 줄 맨 왼쪽 공간을 클릭해도 됨
        - 원하는 줄을 우클릭해서 삽입할수도 있음
            - ![image](https://user-images.githubusercontent.com/11372675/204984280-aa875bb7-9679-4a19-82ef-967e396917b4.png)
2. 프로세스 연결하기
    - 디버그 탭 > 프로세스에 연결
        - ![image](https://user-images.githubusercontent.com/11372675/204982957-9e5b90f9-094d-48d7-8998-42883fd8e5e1.png)
    - 실행중인 언리얼엔진 프로세스 선택
        - ![image](https://user-images.githubusercontent.com/11372675/204983108-81025bca-9294-4ef1-9f8e-0887b644e81c.png)
3. 디버그 모드로 진입
    - ![image](https://user-images.githubusercontent.com/11372675/204983293-3932c4b4-f832-439e-a584-ba9438ff37ea.png)
4. 언리얼 에디터에서 플레이
    - ![image](https://user-images.githubusercontent.com/11372675/204983441-a678859b-c4a6-4903-98ab-da67e5faa3f3.png)
    - 1에서 설정한 중단점에서 일시정지가 되고, 하단에서 필요한 데이터를 확인할 수 있다.

## 디버깅 단축키
- 중단점 삽입, 삭제 : F9
- 프로시저 단위 실행 : F10
    - 함수를 건너뜀
- 한 단계씩 코드 실행 : F11
    - 함수 안으로 들어감
- 커서까지 실행 : Ctrl + F10
- 프로시저 나가기 : Shift + F10