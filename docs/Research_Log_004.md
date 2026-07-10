# Research Log
## Date : 2026-07-07

# URP Deferred Rendering Pipeline 분석 (UniversalRenderer.cs / DeferredLights.cs)

오늘부터 URP 자체를 수정하기 위한 구조 분석을 시작하였다. 목표는 기존 URP를 그대로 사용하는 것이 아니라, Character 전용 Rendering Pipeline을 추가하여 Live2D와 Anime 스타일의 Toon Rendering을 구현할 수 있는 위치를 찾는 것이다.
처음 UniversalRenderer.cs를 열었을 때 약 2000줄이 넘는 코드량 때문에 전체를 읽는 것은 비효율적이라고 판단하였다. 따라서 Renderer 전체를 읽기보다 실제 렌더링 순서를 결정하는 부분만 선택적으로 분석하기로 하였다.
RenderPass 생성부를 먼저 확인하였다.

생성자에서는

- GBufferPass
- DeferredPass
- ForwardOnlyPass
- CopyDepthPass
- ShadowPass

등의 Pass를 생성하고 있었다.
처음에는 생성되는 순서가 렌더링 순서라고 생각하였지만 실제로는 Setup() 함수 내부에서 EnqueuePass()를 호출하여 실행 순서를 결정한다는 사실을 확인하였다. 
즉 RenderPass는

RenderPass → Setup() → EnqueuePass() → Execute()

순으로 동작한다.
이를 통해 UniversalRenderer는 실제 렌더링을 수행하는 클래스라기보다 RenderPass를 관리하는 Scheduler에 가깝다는 것을 이해하였다.

---

이후 Deferred Rendering의 전체 구조를 다시 정리하였다.
기존에는
Deferred Rendering

↓

Lighting

↓

출력

정도로 단순하게 생각했지만
실제 구조는

ShadowPass → GBufferPass → CopyDepthPass → DeferredPass → ForwardOnlyPass → PostProcess → Screen

이라는 사실을 확인하였다.

특히 Deferred Rendering이
"GBuffer 생성" 과 "Lighting"

으로 완전히 분리되어 있다는 점이 가장 큰 수확이었다.
즉 DeferredPass는 Scene을 다시 그리는 것이 아니라GBuffer를 읽어 Lighting만 계산하는 역할이라는 점을 이해하였다.

---

ForwardOnlyPass에 대한 이해도 수정하였다.
처음에는 단순한 Forward Rendering이라고 생각하였으나
실제로는 Hair, Skin, Unlit등

GBuffer에 저장하기 어려운 Material만 처리하는 전용 Pass였다.
Shader 내부의
LightMode = UniversalForwardOnly
Tag를 이용하여
ForwardOnlyPass에서 자동으로 렌더링된다는 사실도 확인하였다.

이를 통해

새로운 LightMode = CharacterToon 을 정의하면 Character 전용 RenderPass를 추가할 수 있을 것으로 판단하였다.
즉
Shader Tag → RenderPass 선택
이라는 구조를 이해하게 되었다.

---

UniversalRenderer 분석 중
TODO 주석도 발견하였다.

특히
CreateGbufferResources() 를 너무 늦게 호출하면 GBuffer Attachment의 크기가 잘못 생성될 수 있다는 개발자의 메모를 확인하였다.

이를 통해 GBuffer 생성 시점 자체가 매우 중요한 구조라는 사실을 알게 되었다.
Unity 개발자도 호출 순서를 다시 조사해야 한다는 TODO를 남겨놓은 것으로 보아 렌더링 파이프라인 내부에서도 매우 민감한 부분이라는 점을 확인하였다.

---

ShadowMap도 다시 이해하였다.
기존에는 그림자가 저장된 Texture라고만 생각하였다.
하지만 실제로 빛 시점에서 생성된 Depth Buffer라는 점을 이해하였다.

즉

빛 입장에서 Scene을 먼저 렌더링하고 Depth만 저장한다. Lighting 단계에서는 현재 Pixel의 Depth와 ShadowMap의 Depth를 비교하여 빛을 받는지 판단한다.

이를 통해

그림자를 계산한 후 Mask를 씌우는 것이 아니라 ShadowMap 자체를 하나의 입력으로 사용하는 구조가 더 자연스럽다는 생각으로 발전하였다.

기존 아이디어 Shadow 계산 → Mask 삭제 → 출력

에서
새로운 아이디어는
ShadowMap → Face Mask → Hair Mask → Material Mask → Style Function → Shadow 수정 → Lighting

으로 변경되었다.

즉 ShadowMap은 결과가 아니라
Style Function의 입력 데이터가 된다.

---

이후 DeferredLights.cs 분석을 시작하였다. UniversalRenderer보다 코드가 훨씬 적었지만 실제로 Deferred Rendering의 핵심 구조가 대부분 이곳에 존재하였다.

먼저 GBufferNames를 확인하였다.

_GBuffer0

_GBuffer1

_GBuffer2

...

등이 실제 RTHandle 이름이라는 것을 확인하였다.

이후

GBufferAlbedoIndex

GBufferSpecularMetallicIndex

GBufferNormalSmoothnessIndex

GBufferLightingIndex

등이 각각 어떤 역할을 담당하는지 분석하였다.

---

GBufferSliceCount() 도 확인하였다. 처음에는 GBuffer 개수가 고정이라고 생각했지만 실제로는 ShadowMask, RenderingLayer, FramebufferFetch등 옵션에 따라 동적으로 증가한다.

즉 필요한 Buffer만 생성하는 구조라는 것을 이해하였다. 이 구조를 보면서 향후 Face Buffer, Hair Buffer, Cloth Buffer 등을 추가할 수 있을 가능성을 생각하였다.
물론 단순히 Buffer 개수만 늘리는 것으로 끝나는 것이 아니라 DeferredPass와 Shader도 함께 수정해야 한다는 점도 확인하였다.

---

GetGBufferFormat()도 분석하였다. 각 GBuffer가 RGBA8, Normal Format, Depth Format 등 어떤 GraphicsFormat으로 생성되는지를 결정한다. 여기서 Buffer는 단순히 저장 공간이며 실제 계산은 하지 않는다는 점을 이해하였다.

---

CreateGbufferResources()는 오늘 가장 중요한 함수 중 하나였다. 여기서 RTHandle을 실제로 생성한다.

즉
Buffer 개수 결정 → GraphicsFormat 결정 → RTHandle 생성
이라는 순서를 확인하였다.

이를 통해 Character Buffer를 추가하려면 가장 먼저 수정해야 하는 위치가 CreateGbufferResources() 라는 점을 메모하였다.

---

UpdateDeferredInputAttachments()도 분석하였다.

이 함수는DeferredPass가 어떤 GBuffer를 읽을 것인지를 연결한다.

즉
GBuffer 생성 → Deferred 입력 연결 → Lighting 이라는 흐름을 이해하였다.

---

Setup()에서는, ColorAttachment, DepthAttachment 등을 Deferred Rendering에 연결하는 구조를 확인하였다.

특히 GBufferLightingIndex는 새로운 Texture를 만드는 것이 아니라 실제 Camera Color Attachment와 연결된다는 사실을 확인하였다. 이를 보며 ColorBuffer를 여러 개로 분리하는 아이디어를 생각하게 되었다.

---

기존에는 Character도 World도 하나의 Color Buffer에 출력한다고 생각하였다.

그러나

World → ColorBuffer0
Character → ColorBuffer1
ColorBuffer0 + ColorBuffer1 → Composite → Final Output
이라는 구조가 더 적합하다는 생각이 들었다.

Character와 World의 Lighting Style을 독립적으로 관리할 수 있기 때문이다. 
다만 기존 GBufferLightingIndex를 수정하는 것은 Deferred 구조 전체를 변경해야 하므로
초기 연구에서는 새로운 RTHandle을 추가하고 Character 전용 Color Buffer를 생성하는 방식이 더 현실적이라고 판단하였다.

---

오늘 연구를 진행하면서 가장 크게 바뀐 점은 Rendering을 하나의 함수로 생각하지 않게 되었다는 것이다.

기존에는 Shader → Rendering 이라고 생각했지만

현재는

RenderPass → Buffer → Lighting → Composite

라는 독립적인 단계로 바라보게 되었다.

즉 Buffer는 데이터를 저장하고 RenderPass는 데이터를 생성하며 Lighting은 Buffer를 읽고 Composite는 결과를 합친다는 구조로 이해하게 되었다.

---

또한 기존에는 하나의 Material 안에서 모든 것을 해결하려고 하였지만

현재는 Face Buffer, Hair Buffer, Cloth Buffer, World Buffer 처럼 역할을 분리하는 방향으로 사고가 발전하였다.
각 Buffer는가장 표현하기 좋은 Parameter만 저장하고 Style Function은 필요한 Buffer만 읽어서 Anime 스타일을 계산하는 구조를 목표로 한다.

예를 들어
Face Buffer → {Face Direction, Face Shadow, Eye Mask, Nose Mask} | Hair Buffer → {Tangent, Angel Ring, Highlight Strength} | Cloth Buffer → {Pattern, Material ID, Fold Information}

등을 저장하는 방향을 생각하였다.

---

최종적으로 Style은 Buffer에 저장하는 것이 아니라 Style Function에서 결정하는 방향으로 연구 목표를 수정하였다.
즉 Buffer는 데이터를 저장할 뿐이며 Style마다 바뀌는 것은 수학식과 Mask 처리 방식이다.

최종 목표는
ShadowMap + Depth + Normal + Material Buffer + StylePreset → Style Function → Character Color Buffer → Composite → Final Image
라는 구조의 Character Rendering Pipeline을 구축하는 것이다.

현재는 RenderPass와 Buffer의 역할을 이해하는 단계이며 다음 연구에서는 DeferredLights의 ExecuteDeferredPass(), Lighting Shader, GBufferPass 를 분석하여

실제 Lighting 계산이 어디에서 수행되는지 확인하고
Style Function을 삽입할 수 있는 위치를 찾을 예정이다.
