---
title: "VisualStudio 다운로드 속도 향상시키기"
permalink: /posts/git/upgradevsdownloadspeed
excerpt: "How to Upgrade Visual Studio Installer download speed"
last_modified_at: 2022-12-11T22:38:00-04:00
toc: true
---

# VisualStudio download 향상시키기
- 이건 좀 아니지 않나 싶을정도로 느릴 때가 있다.
    - ![image](https://user-images.githubusercontent.com/11372675/206906931-6f8e7118-17d9-47c9-9489-ced897f4c54b.png)
- 메모장을 권리자 권한으로 실행하기
    - ![image](https://user-images.githubusercontent.com/11372675/206906833-6cd39f31-568e-4238-b4e7-43b743215802.png)
- 메모장에서 C:\Windows\System32\drivers\etc 폴더의 hosts 파일 열기
    - tip) 안 보인다면 txt확장자 말고 모든 파일로 탐색
- host 하단에 해외 ip를 적어주기
    - ![image](https://user-images.githubusercontent.com/11372675/206907031-8e57a4e5-0e5a-4702-9c38-909fee68beb9.png)
    - 어떤 ip를 적어야 할 지 모르겠으면 아래에 (참고)를 보자.
- 야호! 이제 좀 정상적으로 작동함
    - ![image](https://user-images.githubusercontent.com/11372675/206907095-af3e12de-b95b-4abc-8726-3cd891d711ea.png)

- 참고
    - https://stackoverflow.com/questions/44563536/visual-studio-2017-installer-extremely-slow
    - 위 링크로 들어가서 아래에 보면 예시 ip가 있음
