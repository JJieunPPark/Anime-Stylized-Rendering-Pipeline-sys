Research Log - 2026.07.21
목표

URP의 PBR 라이팅 구조를 기반으로 자체 Stylized Lighting Shader의 첫 번째 프로토타입을 제작한다. 기존 UniversalFragmentPBR()의 호출 구조를 이해하고, 이를 직접 구현한 UniversalFragmentStylized()로 대체하여 실제 캐릭터 모델에서 셀 셰이딩이 정상적으로 동작하는지 검증하는 것을 목표로 하였다.

진행 내용
1. Stylized Shader 프로젝트 구조 설계

기존에는 URP 내부의 Lighting.hlsl를 직접 수정하는 방식을 고려하였으나, 유지보수성과 향후 Unity 업데이트를 고려하면 원본 파일을 변경하는 것은 적절하지 않다고 판단하였다. 따라서 URP의 구조를 그대로 유지하면서 필요한 부분만 교체하는 구조를 설계하였다.

프로젝트는 다음과 같은 세 개의 파일로 구성하였다.

StylizedLit.shader
        │
        ▼
StylizedLitPass.hlsl
        │
        ▼
StylizedLighting.hlsl

각 파일의 역할은 다음과 같이 분리하였다.

StylizedLit.shader
ShaderLab 작성
Pass 구성
HLSL 연결
StylizedLitPass.hlsl
Vertex Shader
Fragment Shader
Stylized Lighting 호출
StylizedLighting.hlsl
Toon Lighting 계산
향후 Angel Ring, Face Shadow, PVC Highlight 등 Stylized 효과 구현 예정

이 구조는 Unity URP 내부의 Lit Shader 구조와 거의 동일한 형태이며, 이후 Deferred Rendering으로 이식하기에도 적합하다.

2. UniversalFragmentPBR 호출 구조 변경

기존 Lit Shader는 Fragment Shader 내부에서 최종적으로

UniversalFragmentPBR()

를 호출하여 모든 라이팅 계산을 수행한다.

이번 프로토타입에서는 이를 직접 구현한

UniversalFragmentStylized()

로 변경하였다.

그러나 처음에는 Lighting.hlsl 전체를 복사하여 사용하면서 include guard와 함수 이름 충돌 문제를 겪었다. 또한 함수 이름에 오타가 존재하여 UniversalFragmentPBR()가 정상적으로 호출되지 않는 오류도 발생하였다.

오류를 해결하면서 다음과 같은 사실을 이해하였다.

URP의 Lighting 함수는 다양한 Shader에서 공통적으로 사용된다.
함수 이름 하나만 변경되어도 Shader Graph를 포함한 모든 셰이더에서 컴파일 오류가 발생한다.
include guard는 중복 include를 막기 때문에 원본과 동일한 guard를 사용하면 이후 추가한 코드가 전혀 컴파일되지 않는다.
원본 URP 파일은 수정하지 않고, 별도의 Stylized 파일을 통해 확장하는 것이 유지보수에 훨씬 적합하다.

이 과정에서 HLSL의 include 구조와 Unity Shader 컴파일 과정을 실제로 경험하며 이해할 수 있었다.

3. Toon Lighting Prototype 구현

이번 프로토타입에서는 가장 단순한 Toon Lighting만 구현하였다.

라이팅 계산은 다음과 같은 흐름으로 구성하였다.

Main Light

↓

Light Direction

↓

Normal

↓

dot(N,L)

↓

step()

↓

Shadow / Light 선택

↓

Final Color

기존 PBR의 BRDF 계산은 모두 제거하고 가장 단순한 셀 셰이딩을 구현하였다.

그림자 색상은

Shadow = Albedo × 0.5

를 사용하였으며,

step(0.5, NdotL)

을 이용하여 두 단계의 명암만 출력하도록 구성하였다.

현재는 Threshold 값이 고정되어 있으며 Inspector에서 조절하는 기능은 아직 구현하지 않았다.

4. 실제 캐릭터 모델 적용

Sphere에서 정상적으로 동작하는 것을 확인한 후 실제 3D 캐릭터 모델에 셰이더를 적용하였다.

초기에는 Sphere 하나만 테스트하였지만 이후 캐릭터의 모든 머티리얼을 새로운 Stylized Shader로 교체하여 테스트를 진행하였다.

처음에는 일부 흰색 장식이 기존 셰이더를 사용하고 있어 일부 부위만 다른 라이팅이 적용되는 현상이 발생하였다.

이를 조사하면서 다음과 같은 사실을 확인하였다.

하나의 모델은 여러 개의 Material을 사용한다.
Texture 이름이 아닌 Material 단위로 Shader가 적용된다.
일부 부위만 다른 셰이더가 적용되어 있으면 해당 부분만 다른 라이팅이 출력된다.
모든 Material의 Shader를 동일하게 변경해야 전체 모델이 같은 라이팅을 사용한다.

모든 머티리얼을 StylizedLit으로 변경한 후 캐릭터 전체가 동일한 셀 셰이딩으로 출력되는 것을 확인하였다.

5. 구현 결과

최종적으로 다음 기능들이 정상적으로 동작하였다.

Custom Stylized Shader 생성
StylizedLit.shader 작성
StylizedLitPass.hlsl 작성
StylizedLighting.hlsl 작성
UniversalFragmentStylized() 구현
Directional Light 연동
Sphere 테스트 성공
실제 캐릭터 모델 적용 성공
모든 Material에서 동일한 Toon Lighting 출력 확인

이는 기존 URP 구조를 유지하면서 첫 번째 Stylized Lighting Prototype을 성공적으로 구현한 것이다.

이번 구현을 통해 새롭게 이해한 점

이번 구현을 진행하면서 가장 크게 느낀 점은 URP의 구조가 생각보다 매우 모듈화되어 있다는 것이다. 처음에는 Lighting.hlsl 하나만 수정하면 모든 것이 바뀔 것이라 생각했지만 실제로는 ShaderLab, Pass, Fragment Shader, Lighting 함수가 서로 역할을 분리하여 동작하고 있었다.

특히 Fragment Shader는 직접 라이팅 계산을 수행하는 것이 아니라 Lighting 함수에 필요한 데이터를 전달하고 최종 색상을 받아오는 역할을 수행한다는 점을 직접 확인하였다. 따라서 앞으로 새로운 Stylized Lighting을 구현하더라도 Fragment Shader 전체를 수정하는 것이 아니라 Lighting 함수만 교체하면 대부분의 구조를 그대로 유지할 수 있다는 사실을 이해하게 되었다.

또한 하나의 캐릭터 모델이 여러 개의 Material로 구성된다는 점을 실제 테스트를 통해 확인하였다. 셰이더는 Mesh가 아니라 Material 단위로 적용되기 때문에 캐릭터 전체에 새로운 라이팅을 적용하려면 모든 Material을 동일한 Shader로 변경해야 한다는 점도 이번 구현 과정에서 자연스럽게 이해할 수 있었다.

향후 개발 계획

이번 프로토타입은 셀 셰이딩의 가장 기본적인 구조만 구현한 단계이다. 앞으로는 현재의 구조를 유지하면서 하나씩 기능을 추가할 예정이다.

예정된 구현 순서는 다음과 같다.

Shadow Threshold

↓

Shadow Color

↓

Shadow Softness

↓

Global Illumination

↓

Additional Light

↓

Hair Angel Ring

↓

Face Shadow

↓

PVC Highlight

↓

Material Mask

↓

Deferred Rendering 이식

현재 구현은 단순한 step() 기반의 2단 Toon Lighting이지만, 이 구조를 기반으로 연구에서 목표로 하는 Live2D 스타일의 인터랙티브 셰이더까지 점진적으로 확장할 계획이다.

오늘의 연구 성과

오늘은 단순히 URP 소스 코드를 분석한 것이 아니라, 분석한 구조를 기반으로 실제 동작하는 커스텀 셰이더를 구현하였다. UniversalFragmentPBR()를 UniversalFragmentStylized()로 대체하여 자체 라이팅 파이프라인을 구축하였으며, 이를 Sphere뿐 아니라 실제 캐릭터 모델에 적용하여 정상적으로 렌더링되는 것을 확인하였다. 이는 연구의 방향이 단순한 코드 분석 단계에서 실제 구현 단계로 전환되었음을 의미하는 첫 번째 마일스톤이며, 이후 개발될 모든 Stylized Rendering 기능의 기반이 되는 프로토타입이라 할 수 있다.
