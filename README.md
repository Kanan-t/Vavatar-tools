# Vavatar_tools_KN

버추얼 아바타 제작을 위한 블렌더 애드온입니다. **셰이프키 생성**과 **텍스처 베이크**를 한 패널에서 처리합니다. (지원: 블렌더 4.2 이상)

---

## 이용 및 배포 조건

- 본 애드온은 **개인용 / 지인 배포용**으로, **1차 배포**를 전제로 제작되었습니다.
- **2차 배포가 적발될 경우 배포를 중단**합니다.
- 2차 배포를 원하는 경우 **제작자에게 문의**해 주세요.

---

## 설치

배포받은 **`vavatar_tools_kn-1.0.0.zip`** 파일로 설치합니다.

1. 블렌더 **Edit → Preferences → Get Extensions**
2. 오른쪽 위 **아래 화살표(⌄) 메뉴 → Install from Disk...**
3. 받은 **`vavatar_tools_kn-1.0.0.zip`** 을 선택합니다.

설치하면 3D 뷰포트 사이드바(**N**)에 **Vavatar_tools_KN** 탭이 생깁니다.

---

## 업데이트

새 버전이 나오면 애드온이 알려줍니다. (배포자가 새 버전을 올려두면 작동합니다.)

- 블렌더를 켜면 애드온이 **자동으로 새 버전을 확인**하고, 있으면 패널 맨 위에 **"New version 있음" 알림 배너**가 뜹니다.
- 배너(또는 패널의 **Update** 섹션)에서 **Update Now** 를 누르면 최신 버전이 설치됩니다. 적용하려면 **블렌더를 다시 시작**하세요.
- 직접 확인하려면 패널 **Update → Check for Updates** 를 누르면 됩니다.
- 자동 확인을 끄려면: **Preferences → Add-ons → 이 애드온 → "Check for updates on startup"** 체크 해제.

> 자동 확인이 되지 않으면 **Preferences → System → Network** 의 온라인 접근이 켜져 있는지 확인하세요.

---

## 사용법

패널: 3D 뷰포트 → **N** → **Vavatar_tools_KN**

### 셰이프키

셰이프키 생성 순서는 각 **공식 매뉴얼(문서)의 순서를 그대로** 따릅니다.

- **Shape Keys from List** : ARKit 셰이프키 세트를 **iPhone 페이스 트래킹 공식 문서(52 blendshapes)의 순서** 그대로 생성합니다.
- **MMD Shape Keys** : MMD 모프 세트를 **VRChat MMD Visemes 가이드의 순서** 그대로 생성합니다.
- **Split Shape Key L/R** : 좌우 대칭으로 만든 셰이프키를 왼쪽 / 오른쪽 두 개로 나눕니다.
  - 예: `mouthUpperUp` 셰이프키를 만들어 **활성 상태로 선택**한 뒤 버튼을 누르면 → `mouthUpperUpLeft` 와 `mouthUpperUpRight` 가 생성됩니다. 각각 **왼쪽 절반 / 오른쪽 절반**만 움직이도록 나뉩니다.
  - 기본적으로 **+X 방향을 Left**로 처리합니다.
  - 같은 이름의 Left / Right 가 이미 있으면 **그 자리에 값을 덮어씁니다**(맨 아래에 새로 생기지 않음).
- **ARKit / MMD Reference** : 참고 문서를 엽니다.

### 베이크

1. **Image** 에서 사용할 이미지를 선택합니다.
2. 오브젝트를 선택하고, **베이크할 면이 정면으로 보이도록 뷰를 맞춘 뒤** **Create Projected UV** 를 누릅니다.
   - Project From View는 **현재 화면에 보이는 각도 그대로** UV에 투영합니다. 예를 들어 얼굴 텍스처라면 **Front View(넘버패드 1)** 처럼 원하는 면을 정면으로 맞춰야 합니다.
3. UV Editing에서 UV를 정리합니다.
4. **UV to Bake** 에서 대상 UV를 고르고 **Bake** 를 누른 뒤 저장 위치를 지정합니다.
5. **4096×4096 투명 배경 PNG** 로 저장되고, 결과가 머티리얼(및 UV Editing · Texture Paint)에 반영됩니다.

---

## 참고 문서

- ARKit : https://hinzka.hatenablog.com/entry/2021/12/21/222635
- MMD : https://docs.google.com/document/d/1dqzjyvJPa845Atbq5oUaWyt19OK_ia0m325nvYddHy4/edit

---

## 라이선스

GNU General Public License v3.0 이상 (GPL-3.0-or-later)
