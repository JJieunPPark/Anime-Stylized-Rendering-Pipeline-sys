# Research Log - StencilDeferred.hlsl 분석 및 Stylized Deferred Lighting 구조 설계

## 연구 목표

이번 분석의 목표는 URP Deferred Rendering에서 실제 픽셀 단위의 라이팅이 어떤 과정을 거쳐 수행되는지를 이해하는 것이다. 이전까지는 UniversalRenderer와 DeferredLights.cs를 분석하여 C# 단계에서 Render Pass가 어떻게 생성되고 Shader가 호출되는지를 확인하였다. 그러나 실제 조명 계산은 GPU에서 수행되므로, 이번에는 StencilDeferred.hlsl을 중심으로 Deferred Lighting의 실제 데이터 흐름과 계산 구조를 분석하였다. 또한 기존의 물리 기반 라이팅(BRDF)을 Live2D 스타일의 Stylized Rendering으로 변경하기 위해 어떤 위치를 수정해야 하는지 조사하고, 기존 Light 시스템을 Style System으로 재해석할 수 있는 가능성을 함께 검토하였다.

---

## DeferredShading() 함수의 역할

StencilDeferred.hlsl에서 가장 중요한 함수는 DeferredShading()이었다. 처음에는 이 함수 내부에서 복잡한 BRDF 계산이 모두 이루어질 것이라고 예상하였으나, 실제 분석 결과 DeferredShading()은 계산을 직접 수행하는 함수가 아니라 Lighting 계산을 위한 데이터를 준비하고 적절한 함수로 전달하는 역할을 수행하고 있었다. 전체 구조는 World Position을 복원한 뒤 Light를 생성하고, Material 정보를 준비한 후 LightingPhysicallyBased()를 호출하여 최종 Color를 반환하는 구조였다.

```
Depth

↓

World Position 복원

↓

Light 생성

↓

Material 정보 준비

↓

LightingPhysicallyBased()

↓

Color Buffer 출력
```

즉 DeferredShading()은 실제 BRDF를 구현하는 함수라기보다 Deferred Lighting 전체를 제어하는 진입점이라고 이해할 수 있었다. 따라서 향후 Stylized Rendering을 구현할 경우 LightingPhysicallyBased()를 직접 수정하기보다 DeferredShading() 단계에서 새로운 LightingStylized()를 호출하는 구조가 가장 자연스러울 것으로 판단하였다.

---

## InputData의 역할

DeferredShading()에서는 가장 먼저 InputData를 생성한다. InputData에는 PositionWS, NormalWS, ViewDirectionWS가 저장되며 이후 모든 라이팅 계산의 입력으로 사용된다. 기존에는 단순히 BRDF 계산을 위한 데이터라고 생각하였으나 분석을 진행하면서 Stylized Rendering에서도 동일한 데이터가 매우 중요하다는 점을 확인하였다.

PositionWS는 Style Volume이나 Bounding Box 내부에 현재 픽셀이 포함되는지를 판정하는 기준으로 사용할 수 있으며, NormalWS는 Stroke 방향, Rim Light, Highlight, Edge Detection 등의 기준 벡터가 된다. 또한 ViewDirectionWS는 Angel Ring이나 Fresnel 기반 Highlight, 카메라 의존형 스타일 효과를 구현하는 데 그대로 사용할 수 있다.

즉 InputData는 BRDF 전용 데이터가 아니라 Stylized Rendering에서도 거의 그대로 재사용 가능한 공통 입력 구조라고 판단하였다.

---

## GetStencilLight() 분석

DeferredShading() 다음으로 중요한 함수는 GetStencilLight()였다. 이 함수는 Directional Light, Point Light, Spot Light를 각각 처리하지만 최종적으로는 모두 동일한 Light 구조체를 생성하여 반환한다. 즉 광원의 종류는 내부적으로 다르지만 이후 라이팅 계산에서는 모두 동일한 인터페이스를 사용하도록 설계되어 있었다.

```
Directional

↓

Light

Point

↓

Light

Spot

↓

Light
```

이는 LightingPhysicallyBased()가 광원의 종류를 구분하지 않고 동일한 코드로 처리할 수 있도록 하기 위한 구조이며, GPU에서 코드 중복을 줄이기 위한 설계라고 이해하였다.

처음에는 Light 구조 자체를 StyleField로 변경하는 방향을 생각하였으나 분석을 진행하면서 기존 Light 구조를 유지한 채 의미만 변경하는 것이 기존 URP와의 호환성 측면에서 더욱 유리하다는 결론을 내렸다.

즉 기존에는

```
Light

↓

빛 정보
```

였다면,

연구에서는

```
Light

↓

Style Mask Generator
```

로 역할 자체를 변경하는 방향을 고려하였다.

---

## Point Light와 Spot Light의 재해석

기존 URP에서 Point Light는 Sphere 형태의 영향을 가지며 Spot Light는 Cone 형태의 영향을 가진다. 처음에는 실제 Sphere Mesh나 Cone Mesh를 렌더링하는 것으로 이해하였으나 분석 결과 이들은 화면에 보이는 오브젝트가 아니라 "라이팅 계산이 수행되는 공간"을 정의하기 위한 영역이었다.

```
Point

↓

Sphere

↓

Lighting 계산 영역
```

```
Spot

↓

Cone

↓

Lighting 계산 영역
```

즉 Sphere나 Cone은 빛을 표현하는 것이 아니라 GPU가 어떤 픽셀에서 라이팅 계산을 수행할지를 결정하는 공간 정보였다.

이를 바탕으로 연구에서는 Point Light를 단순한 광원이 아니라 Style Volume으로 사용하는 방향을 구상하였다.

예를 들어

```
Point

↓

Hair Highlight Volume
```

```
Point

↓

Face Color Volume
```

```
Point

↓

Latex Reflection Volume
```

처럼 특정 스타일이 적용되는 공간 자체를 정의하는 용도로 사용할 수 있을 것으로 판단하였다.

Spot Light 역시 동일하게 Cone 형태의 Style Volume으로 사용하면 방향성이 있는 Pattern이나 Brush Stroke를 구현하는 데 활용할 수 있을 것으로 예상하였다.

---

## Distance Attenuation의 재해석

기존 URP에서는 Distance Attenuation을 빛이 거리와 함께 감소하는 계수로 사용한다.

그러나 본 연구에서는 이를 단순히 빛의 감쇠가 아니라 Style의 강도를 조절하는 Falloff로 재해석하는 방향을 고려하였다.

예를 들어

```
Distance Attenuation

↓

Stroke Strength
```

```
Distance Attenuation

↓

Pattern Density
```

```
Distance Attenuation

↓

Highlight Width
```

처럼 사용할 수 있다.

분석 과정에서는 단순히 기존 감쇠 공식을 사용하는 것이 아니라 별도의 Falloff Field를 생성하여 Material마다 서로 다른 감쇠 곡선을 적용하는 아이디어도 도출하였다.

예를 들어 Hair는 부드러운 Falloff를 사용하고 Latex는 급격하게 감소하는 Falloff를 사용하는 방식이다.

이는 기존 광원 계산을 그대로 활용하면서도 표현 방식만 크게 변경할 수 있다는 장점을 가진다.

---

## Cookie 분석과 Material 매핑

기존 Cookie는 Spot Light 등에 Texture를 투영하여 빛의 모양을 변경하는 기능이다.

처음에는 Stylized Rendering에서도 동일한 방식으로 사용하는 것을 고려하였다.

그러나 분석 과정에서 Cookie는 결국 빛 위에 셀로판지를 씌우는 것과 동일한 구조라는 점을 이해하였다.

Stylized Rendering에서는 빛의 색을 변경하기보다는 Material 자체에 Pattern을 적용하는 것이 더욱 자연스럽다고 판단하였다.

따라서 기존

```
Light

↓

Cookie

↓

Projection
```

구조 대신

```
Material

↓

Pattern Texture

↓

Material Mapping
```

구조를 사용하는 방향으로 연구 방향을 수정하였다.

---

## Bounding Box 기반 Style Mapping

초기에는 Pattern Texture를 UV 기반으로 매핑하는 방식을 고려하였다.

그러나 캐릭터마다 UV가 모두 다르며 UV Seam 문제도 발생할 수 있다는 점을 고려하여 Bounding Box 기반 Mapping을 사용하는 아이디어를 정리하였다.

예를 들어

```
Hair Bounding Box

↓

Hair Pattern
```

```
Face Bounding Box

↓

Skin Pattern
```

```
Accessory Bounding Box

↓

Metal Pattern
```

처럼 Material별 Bounding Box를 생성하고 해당 영역 내부에서만 Pattern을 계산하는 방식이다.

이를 통해 UV 의존성을 크게 줄일 수 있으며 Material별 Pattern 관리도 쉬워질 것으로 예상하였다.

추가적으로 OBB(Oriented Bounding Box)도 조사하였으나 현재 연구 단계에서는 구현이 단순하고 관리가 쉬운 Object Space 기준 Bounding Box가 더욱 적절하다고 판단하였다.

---

## Shadow와 Pattern의 결합

기존 URP에서는 Shadow Attenuation을 그림자의 강도로 사용한다.

그러나 Stylized Rendering에서는 그림자의 위치 자체는 그대로 유지하면서 내부 표현만 변경하는 것이 더욱 적절하다고 판단하였다.

예를 들어 Shadow 영역에서는

- Pencil Stroke
- Hatching
- Watercolor Pattern

등을 적용하고,

Highlight 영역에서는

- Angel Ring
- Latex Reflection
- Brush Highlight

를 적용하는 구조를 구상하였다.

즉 Shadow는 단순히 어두운 영역이 아니라 Pattern을 적용하는 Mask로 재해석하였다.

---

## BRDF 대신 Style Function

이번 분석에서 가장 큰 연구 방향 변화는 BRDF 자체를 수정하는 것이 아니라 새로운 Style Function을 만드는 것이 더 적절하다는 점이었다.

기존 URP는

```
LightingPhysicallyBased()

↓

BRDF
```

구조를 사용한다.

그러나 연구에서는

```
LightingStylized()

↓

Style Function
```

으로 변경하는 방향을 목표로 하였다.

Style Function에서는 물리 기반 Reflectance보다

- Brush Stroke
- Pencil Hatching
- Anime Shadow
- Latex Reflection
- Angel Ring
- Material Pattern

등을 우선적으로 계산하도록 설계할 예정이다.

즉 BRDF가 물리 기반 반사를 계산하는 함수라면 Style Function은 아티스트의 표현 방식을 계산하는 함수로 정의하였다.

---

## StyleData 구조 제안

기존 Deferred Lighting은

```
GBufferData

↓

BRDFData
```

로 변환된다.

그러나 연구에서는

```
GBufferData

↓

StyleData
```

구조를 사용하는 방향을 제안하였다.

StyleData에는 기존 Material 정보뿐 아니라

- Material ID
- Pattern ID
- Falloff Type
- Environment Receive
- Style Strength

등 Stylized Rendering에 필요한 데이터를 저장하도록 설계할 예정이다.

이를 통해 LightingStylized()는 BRDF 계산이 아니라 Material과 Pattern 중심의 계산을 수행하도록 변경할 수 있을 것으로 예상하였다.

---

## 연구 방향 정리

이번 StencilDeferred.hlsl 분석을 통해 기존 URP Deferred Lighting의 전체 데이터 흐름을 이해할 수 있었다. 또한 Point Light와 Spot Light를 Style Volume으로 재해석하고, Distance Attenuation을 Style Falloff로 변경하며, Cookie 대신 Material Mapping을 사용하는 방향으로 연구 아이디어를 구체화하였다. 무엇보다 기존 BRDF를 수정하는 것이 아니라 StyleData와 LightingStylized()를 새롭게 설계하는 것이 연구의 핵심이라는 점을 확인하였다.

현재까지 정리된 Stylized Rendering Pipeline은 다음과 같다.

```
GBufferData

↓

StyleData 생성

↓

Style Volume 생성

↓

Material Buffer 판정

↓

Pattern Texture 선택

↓

Falloff 계산

↓

Environment Color 계산

↓

LightingStylized()

↓

최종 Blend Rule

↓

Color Buffer 출력
```

기존 URP는 물리 기반 반사 계산을 목표로 하지만, 본 연구에서는 Material과 Pattern 중심의 Style Rendering을 수행하는 새로운 Deferred Rendering 구조를 설계하는 것을 최종 목표로 한다.
