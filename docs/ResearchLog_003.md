# Research Log
## 2026-07-03

### Goal
기존 상용 Toon Shader의 구조를 분석하여 Live2D 스타일 3D Shader 설계에 필요한 핵심 기능과 구현 방식을 조사한다.

---

## Progress

### 1. 연구 환경 구축

- Unity 프로젝트 생성
- lilToon 설치
- 연구용 캐릭터(Ren) Import
- PMX → Blender → FBX 과정에서 Material 정보가 손실되는 문제 확인
- 이후 연구용 모델은 Unity Asset Store 에셋을 우선 사용하기로 결정

---

### 2. lilToon 구조 분석

lilToon의 전체 구조를 조사하였다.

확인한 구성

- Shader
- Includes
- Pass
- Common Functions

분석 결과 실제 렌더링 계산은 대부분 `Includes` 내부에서 이루어지며, Shader 파일은 Pass와 기능을 연결하는 역할을 수행한다는 것을 확인하였다.

---

### 3. Shadow 구현 분석

`lilGetShading()`을 중심으로 그림자 계산 과정을 분석하였다.

확인한 주요 연산

- dot(N,L)
- saturate()
- lerp()
- lilTooningScale()

기본 Toon Shadow는

```
Normal
→ dot(N,L)
→ Toon Threshold
→ Shadow Color
→ Final Color
```

과 같은 흐름으로 계산됨을 확인하였다.

---

### 4. lerp 함수 분석

GPU에서 가장 자주 사용되는 함수 중 하나인 `lerp()`의 역할을 분석하였다.

주요 활용

- 색상 보간
- Normal 보간
- 그림자 보간

수학적으로는

```
A(1-t)+Bt
```

를 계산하는 선형 보간 함수임을 확인하였다.

---

### 5. Shadow Mask 분석

`shadowStrengthMask`를 분석하였다.

GPU에서는 Texture를 단순한 이미지가 아니라 데이터 저장소처럼 활용한다는 점을 확인하였다.

RGBA 채널은 각각 독립적인 데이터를 저장할 수 있으며, 하나의 Texture 안에 여러 종류의 정보를 Packing하는 방식을 사용한다.

---

### 6. Face Shadow 분석

`_ShadowMaskType == 2`에서 얼굴 전용 Shadow 계산을 확인하였다.

기존 Toon Shadow 대신

- 얼굴 방향 계산
- 좌우 광원 판별
- SDF 기반 얼굴 그림자
- 일반 Shadow와 Blend

과정을 수행하는 구조임을 확인하였다.

---

### 7. Material Inspector와 Shader 대응

Unity Material Inspector의 파라미터가 Shader 변수와 직접 연결되어 있다는 것을 확인하였다.

예시

- Shadow Border
- Shadow Blur
- Shadow Strength

이를 통해 앞으로는

```
Inspector 변경
↓

Shader 코드 확인
↓

화면 변화 분석
```

방식으로 연구를 진행하기로 하였다.

---

### 8. MatCap / Rim Light 조사

MatCap과 Rim Light의 역할 차이를 정리하였다.

- Rim Light : View Direction 기반 가장자리 조명
- MatCap : Normal 기반 재질 표현

특히 MatCap은 PVC 재질과 같은 스타일 표현에 활용 가능성이 높다고 판단하였다.

---

### 9. 연구 아이디어

현재 대부분의 Deferred Rendering은 PBR 중심의 GBuffer를 사용한다.
애니 스타일 렌더링에서는

- Face Shadow
- Hair Shadow
- Rim
- Stylized Highlight

등의 정보가 더 중요할 가능성이 있다.

향후

"애니 스타일 렌더링에 최적화된 Rendering Pipeline"

가능성을 조사할 예정이다.

---

## Next

- Shadow 구현 분석 계속
- MatCap 구현 분석
- Rim Light 구현 분석
- Unity HDRP GBuffer 조사
- Unreal GBuffer 조사
- 기존 NPR Shader(Pencil+4, Advanced Cel Shader)와 비교 분석
