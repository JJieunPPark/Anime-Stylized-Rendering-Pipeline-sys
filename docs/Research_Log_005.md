###Research Log - 2026.07.10
##목표

URP Deferred Rendering Pipeline의 실제 렌더링 흐름을 분석하고, Live2D 스타일의 3D 셰이더를 적용하기 위해 C# 파이프라인과 HLSL 라이팅 구조를 파악한다.

###진행 내용
##1. UniversalRenderer.cs 분석 완료

오늘은 UniversalRenderer.cs를 중심으로 Deferred Rendering이 어떻게 실행되는지 추적하였다. 초반에는 Renderer가 매우 복잡할 것이라 예상했지만, 실제 역할은 렌더 패스를 생성하고 실행 순서를 관리하는 것이었다.
주요 흐름은 다음과 같다.
UniversalRenderer.Setup() → 필요한 Pass 생성 → EnqueuePass() → ScriptableRenderer 가 순서대로 실행
즉 UniversalRenderer는 실제 라이팅을 계산하는 클래스가 아니라 렌더링 순서를 구성하는 관리자 역할임을 확인하였다.
---
##2. Deferred Rendering 실행 순서 확인
Deferred Rendering의 실제 실행 순서를 확인하였다.
Shadow Pass → GBuffer Pass → Depth Copy → Deferred Pass → ForwardOnly Pass → Transparent → PostProcess
여기서 가장 중요했던 부분은

Deferred Pass 와 ForwardOnly Pass 의 관계였다.

처음에는 둘 중 하나만 사용하는 줄 알았으나, 실제로는 Deferred에서 처리 가능한 객체 + ForwardOnly로만 렌더링 가능한 객체를 모두 렌더링하여 하나의 Color Buffer를 완성한다는 것을 확인하였다.
---
##3. GBuffer의 역할 정리
GBuffer는 색을 저장하는 버퍼가 아니라
Normal, Albedo, Material 정보, Depth, Occlusion, Smoothness, Specular등의 재질 정보 저장소라는 점을 다시 정리하였다.
실제 라이팅 계산은 GBuffer → Deferred Lighting → Color Buffer 순서로 진행된다.
---
##4. DeferredLights.cs 분석

이후 DeferredLights.cs를 분석하였다. 생각보다 코드가 짧았으며, 실제 계산은 거의 하지 않고 
광원 정보 준비 → Material 설정 → Shader Pass 실행 만 담당하는 것을 확인하였다.

즉 C# -> GPU에게 필요한 데이터를 전달하는 역할이 대부분이었다.
---
##5. Setup() 함수 분석

다음 함수를 분석하였다.
Setup(...)

이 함수에서는 Depth Attachment, Color Attachment, Depth Copy Texture, Additional Shadow Pass, Normal Pass 여부 등을 Deferred Lighting 클래스 내부 변수에 저장한다.

여기서 GbufferAttachments[GBufferLightingIndex]에 Color Buffer가 연결되는 것도 확인하였다.
---
##6. Point / Spot Light 구조 분석

Point Light와 Spot Light는 생각보다 "빛" 자체를 계산하는 것이 아니라 빛이 영향을 줄 수 있는 공간을 GPU에게 알려주는 역할이었다.
Point Light → 구(Sphere) 형태의 계산 영역, Spot Light → 원뿔(Cone) 형태의 계산 영역 형태의 Mesh를 그린 뒤, 그 영역 내부 픽셀만 Lighting Shader를 실행하여 성능을 높인다.
즉 Sphere → Lighting 계산 영역이며, 실제로 화면에 구가 보이는 것은 아니다.
---
##7. SSAO / Fog 구조 확인

Deferred Lighting 이후에는 SSAO와 Fog가 Fullscreen Pass 형태로 적용된다. 두 기능 모두 Depth Buffer → Screen Space 계산 → Color Buffer 수정 이라는 구조를 가진다.
이는 추후 스타일 후처리에도 참고할 수 있는 구조라고 판단하였다.
---
##8. Shader Pass 연결 구조 확인
InitStencilDeferredMaterial()을 분석하여 FindPass()가 실제 Shader의 Pass 이름을 찾아 번호를 저장한다는 것을 확인하였다.
즉 DeferredLights.cs → Material.FindPass() → StencilDeferred.shader → 실제 Shader Pass로 연결된다.
---
##9. StencilDeferred.shader 발견

Unity6에서는 찾는 데 시간이 걸렸지만, 최종적으로 Shaders/Utils/StencilDeferred.shader를 발견하였다.

이 파일은 Deferred Directional, Point, Spot, Fog, SSAO, Pass를 정의하는 파일이었다.
그러나 계산식은 거의 존재하지 않았다.
---
##10. 실제 계산 위치 확인
처음에는 StencilDeferred.shader 안에서 라이팅 계산을 할 것으로 예상하였다. 그러나 분석 결과 #pragma fragment DeferredShading 으로 Fragment Shader만 지정하고 있었으며, 실제 구현은 StencilDeferred.hlsl 에 존재하였다.
즉 구조는
StencilDeferred.shader → StencilDeferred.hlsl → Lighting.hlsl → BRDF.hlsl → 실제 수학 계산으로 연결된다.
오늘 가장 큰 성과 중 하나였다.
---

###연구 아이디어
##1. Point / Spot Light의 재해석
Point(기존: 빛) → Point(연구: Style Volume) 으로 사용하는 방안을 생각하였다.
즉 빛이 아니라 스타일이 적용되는 공간으로 사용하는 것이다.
예를 들어 Hair Highlight, Face Color, Latex Reflection, Angel Ring등을 각각 별도의 Style Volume으로 관리할 수 있다.
---
##2. Directional / Point / Spot을 스타일 필드로 재정의
기존: Directional → 평행광 →
연구: Directional → Directional Style Field, Point → 구형 스타일 영역 | Spot → 원뿔형 스타일 영역 → 기존 Light를 Style Mask Generator로 재해석

로 개념을 변경한다.
---
##3. Material Buffer 활용

기존에는 Directional Light 하나로 캐릭터 전체가 동일한 계산을 수행한다. 연구에서는 Material Buffer 안에 Hair, Face, Cloth, Latex, Eye, Accessory 등의 Material ID를 저장한 뒤,
Directional / Point / Spot Style Field와 조합하여 부위별 스타일 계산을 수행하는 구조를 구상하였다.
---
##4. BRDF 대신 Style Function

현재 URP: LightingPhysicallyBased() → BRDF 
연구 방향: LightingStylized() → Style Function → 물리 기반 반사보다 스트로크, 해칭, 패턴, 애니풍 그림자, 특수 재질을 우선하는 계산 함수로 변경이 목적이다.

현재 이해한 Deferred 구조
UniversalRenderer
→ DeferredLights
→ StencilDeferred.shader
→ StencilDeferred.hlsl
→ Lighting.hlsl
→ BRDF.hlsl
→ 최종 Color Buffer
---
###다음 목표

다음부터는 C# 분석을 종료하고 HLSL 분석을 시작한다.

우선순위는 다음과 같다.

StencilDeferred.hlsl의 DeferredShading() 함수 전체 분석
GetStencilLight()를 분석하여 Light 구조 생성 과정 이해
LightingPhysicallyBased() 내부 추적
BRDF.hlsl에서 Diffuse, Specular, Fresnel, GGX 등 실제 수학식 분석
기존 BRDF를 대체할 LightingStylized() 구조 설계
Material Buffer와 Style Field를 결합할 위치 검토
Character Pass와 World Pass를 분리하는 구조 설계

##오늘 한 줄 요약

"URP Deferred의 C# 파이프라인 분석을 사실상 마무리하고, 실제 픽셀 계산이 이루어지는 HLSL(StencilDeferred.hlsl)까지 진입하였다. 또한 Directional/Point/Spot Light를 조명이 아닌 Style Field로 재해석하는 연구 방향을 구체화하였다."
