# Research Log - 2026.07.16

## **Lighting.hlsl 분석 (후반부) 및 GBufferCommon.hlsl 분석**

### 목표

오늘의 목표는 URP Deferred Rendering에서 실제 라이팅이 어떻게 최종 색으로 합쳐지는지 이해하고, 이후 제작할 Live2D 스타일 3D Shader의 구조를 어떤 방식으로 설계할지 방향을 정리하는 것이었다. 특히 `Lighting.hlsl`의 후반부를 분석하여 PBR, Blinn-Phong, BakedLit의 전체 흐름을 파악하고, `GBufferCommon.hlsl`을 통해 Deferred Rendering에서 GBuffer가 어떤 규칙으로 구성되는지 조사하였다.

---

# Lighting.hlsl 분석

Lighting.hlsl을 분석하면서 가장 먼저 확인한 것은 Unity가 라이팅을 계산하는 방식이 생각보다 매우 모듈화되어 있다는 점이었다. 이전에는 Lighting.hlsl 하나에서 모든 계산을 수행하는 것으로 생각했지만 실제로는 각 기능별 함수가 분리되어 있었으며, Lighting.hlsl은 이 함수들을 호출하여 하나의 결과를 조립하는 관리자 역할에 가까웠다.

특히 `LightingData` 구조체는 하나의 픽셀에 대한 라이팅 결과를 종류별로 임시 저장하는 컨테이너 역할을 수행한다. 여기에는 GI(Global Illumination), Main Light, Additional Light, Vertex Lighting, Emission이 각각 따로 저장된다. 이 구조체는 RenderTexture나 실제 GBuffer가 아니라 단순히 Deferred Lighting 계산 중 사용하는 임시 데이터 구조체라는 점을 이해하였다.

처음에는 왜 굳이 라이팅을 각각 나누어 저장하는지 의문이 들었지만, 디버깅, 기능 On/Off, 조명 종류별 관리, 최종 합성 순서 제어를 위해 이러한 구조를 사용한다는 점을 확인하였다. 이러한 방식은 내가 제작하려는 Stylized Shader에도 그대로 적용할 수 있으며, 기존 LightingData 대신 StyleLightingData를 만들어 Shadow, Angel Ring, Rim Light, PVC Highlight, Environment Color 등을 각각 따로 저장한 뒤 마지막에 원하는 규칙으로 합치는 구조가 적합하다고 판단하였다.

---

CalculateLightingColor() 함수에서는 LightingData 안에 저장된 여러 라이팅 결과를 하나씩 합쳐 최종 조명색을 생성한다. 기본적인 구조는 GI → Main Light → Additional Lights → Vertex Lighting을 순서대로 더한 뒤 Albedo를 곱하고 마지막에 Emission을 추가하는 형태이다.

이 과정에서 가장 중요한 점은 PBR 경로에서는 Albedo가 이미 BRDF 계산 안에서 적용되기 때문에 마지막 CalculateFinalColor()에서는 Albedo 대신 1을 전달한다는 점이다. 반대로 BakedLit 경로에서는 GI와 조명만 계산한 후 마지막 단계에서 Albedo를 곱한다. 같은 함수라도 호출 경로에 따라 역할이 달라진다는 점을 이해하였다.

또한 Emission은 조명의 영향을 받지 않는 자체 발광이므로 가장 마지막에 더한다. 이 부분을 보며 내가 구현하려는 엔젤링은 Emission처럼 항상 더하는 방식보다는, 빛의 방향에 따라 반응하는 반사광이므로 별도의 Style Layer로 계산하는 편이 적합하다는 결론을 내렸다.

---

Blinn-Phong 부분에서는 Lambert Diffuse와 Specular를 조합하는 방식을 분석하였다. CalculateBlinnPhong()은 하나의 광원에 대해 거리 감쇠, 그림자 감쇠를 적용한 뒤 Lambert Diffuse를 계산하고, Half Vector와 NdotH를 이용하여 Blinn-Phong Specular를 계산한 후 두 결과를 합쳐 반환한다.

여기서 특히 Smoothness가 단순히 0~1 범위의 값이 아니라 exp2()를 이용하여 약 2~2048 범위의 지수로 변환된다는 점이 인상적이었다. 이를 통해 같은 Smoothness 값이라도 매우 날카로운 하이라이트를 표현할 수 있다는 것을 이해하였다.

하지만 내 연구에서는 현실적인 PBR 하이라이트보다는 애니메이션 스타일의 하이라이트가 중요하기 때문에 Blinn-Phong 자체를 그대로 사용하는 것보다 Angel Ring, PVC Highlight 등의 별도 함수로 대체하는 것이 더 적합하다고 판단하였다.

---

UniversalFragmentPBR()은 실제 PBR 계산 함수라기보다는 PBR 파이프라인 전체를 조립하는 관리자 함수라는 점을 이해하였다.

전체 흐름은 다음과 같다.

SurfaceData → BRDFData 생성 → Shadow/AO 준비 → Main Light 획득 → Global Illumination 계산 → Main Light PBR 계산 → Additional Light 반복 → Vertex Lighting 추가 → CalculateFinalColor() 호출

즉 UniversalFragmentPBR() 내부에서 직접 라이팅 공식을 계산하는 것이 아니라 LightingPhysicallyBased()를 반복 호출하는 구조였다.

이를 분석하면서 내 셰이더 역시 UniversalFragmentStylized()와 같은 구조를 만들 수 있을 것이라는 아이디어를 얻었다.

SurfaceData 대신 StyleData를 생성하고, LightingPhysicallyBased() 대신 LightingStylized()를 호출하며, Main Light와 Additional Light 각각에 대해 Hair, Face, PVC, Angel Ring 등의 Style Function을 실행하는 방식으로 변경하면 기존 URP Deferred 구조를 유지하면서도 Stylized Rendering을 구현할 수 있을 것으로 생각하였다.

---

UniversalFragmentBlinnPhong() 역시 PBR과 거의 동일한 구조를 사용하지만, LightingPhysicallyBased() 대신 CalculateBlinnPhong()을 호출한다는 점만 다르다는 것도 확인하였다.

즉 Unity는 라이팅 방식(PBR, Blinn-Phong)이 달라져도 전체 파이프라인은 거의 동일하게 유지하며, 실제 재질 계산 함수만 교체하는 구조를 사용하고 있었다.

이 부분은 내 연구에서도 매우 중요한 참고가 된다. 결국 나 역시 UniversalFragmentStylized() 내부에서 LightingStylized()만 교체하면 전체 Deferred Pipeline은 거의 그대로 사용할 수 있을 가능성이 높다는 것을 확인하였다.

---

BakedLit 역시 분석하였다.

처음에는 BakedLit이 자체적으로 조명을 계산한다고 생각했지만 실제로는 이미 계산되어 저장된 Baked GI를 읽어 AO를 곱하고 Albedo를 적용한 뒤 Emission을 더하는 매우 단순한 구조였다.

즉 BakedLit은 조명을 계산하는 셰이더가 아니라 "미리 계산된 조명을 사용하는 셰이더"라는 점을 이해하였다.

이를 보며 내 셰이더에서도 얼굴 그림자나 기본 명암처럼 변하지 않는 요소는 미리 제작한 Style Texture를 이용하고, Angel Ring이나 Rim Light처럼 움직이는 효과만 실시간으로 계산하는 Hybrid 방식도 가능하겠다는 아이디어를 얻었다.

---

Vertex Shader와 Fragment Shader의 역할도 다시 정리하였다.

처음에는 Fragment Shader에서 너무 많은 계산을 수행하는 것이 아닌가 하는 의문이 있었지만, 실제 GPU 파이프라인을 다시 확인한 결과 Vertex Shader는 데이터를 준비하는 역할이고 Fragment Shader는 최종 픽셀 색을 계산하는 역할이라는 점을 이해하였다.

Vertex Shader에서는 Position, Normal, Tangent, UV, Distance, Material ID 등을 준비하여 Fragment Shader로 전달하고, Fragment Shader에서는 전달받은 데이터를 이용하여 Lighting, Angel Ring, Toon Shadow, Rim Light 등을 계산하는 구조가 가장 적절하다는 결론을 내렸다.

또한 Vertex Shader에서 Toon Shadow를 미리 계산하는 것은 GPU 보간(interpolation) 때문에 정확한 경계가 무너진다는 것도 확인하였다.

GPU는 Vertex Shader에서 계산된 값을 삼각형 내부에서 자동으로 선형보간하기 때문에 Toon 경계나 Angel Ring처럼 날카로운 효과는 반드시 Fragment Shader에서 계산해야 한다는 점을 이해하였다.

---

Clear Coat에 대해서도 조사하였다.

처음에는 단순히 하이라이트를 하나 더 추가하는 기능이라고 생각했지만 실제로는 기본 재질 위에 얇은 투명 코팅층을 하나 더 올려 별도의 반사를 계산하는 구조였다.

이 개념을 보며 Angel Ring 역시 기존 Hair Lighting과는 독립된 두 번째 반사층처럼 구현하면 자연스럽게 빛 방향에 따라 움직이는 효과를 만들 수 있을 것이라는 아이디어를 얻었다.

특히 Angel Ring은 캐릭터가 회전해도 머리카락의 형태만 따라가고 빛 방향에는 계속 반응해야 하므로 기존 Specular를 수정하는 것보다 별도 Layer를 만드는 것이 적합하다고 판단하였다.

---

Additional Light를 처리하는 과정도 분석하였다.

Unity는 Additional Lights를 반복문으로 순회하며 LightingPhysicallyBased()의 결과를 모두 더하는 방식으로 구현되어 있었다.

처음에는 여기서 lerp()를 사용하여 스타일을 보간하면 되지 않을까 생각했지만, 루프 안에서 계속 lerp()를 수행하면 광원의 처리 순서에 따라 결과가 달라지는 문제가 발생할 수 있다는 점을 알게 되었다.

따라서 여러 Style Light를 사용할 경우에는 각 광원의 결과를 먼저 누적하거나 가장 강한 값을 선택한 뒤 마지막 단계에서 한 번만 결합하는 구조가 더 적절하다는 결론을 내렸다.

---

# GBufferCommon.hlsl 분석

Lighting.hlsl 분석 이후에는 GBufferCommon.hlsl을 분석하였다.

처음에는 코드가 100줄 남짓이라 단순한 파일이라고 생각했지만 실제로는 Deferred Rendering 전체에서 사용하는 GBuffer의 저장 규칙을 정의하는 핵심 헤더 파일이라는 것을 이해하였다.

이 파일에는 계산 코드가 거의 없으며, Material Flags 정의, GBuffer 슬롯 번호 정의, GBufferData 구조체 정의, Material Flag Pack/Unpack, Normal Pack/Unpack만 존재한다.

즉 GBufferCommon은 Deferred Rendering에서 GBuffer를 어떤 규칙으로 저장하고 읽을 것인지 정의하는 계약서 역할을 수행한다.

---

Material Flags는 ReceiveShadowsOff, SpecularHighlightsOff, SubtractiveMixedLighting, SpecularSetup 네 가지 플래그를 비트 단위로 저장하는 구조였다.

1, 2, 4, 8처럼 각각 다른 비트를 사용하여 하나의 uint 변수 안에 여러 옵션을 동시에 저장하는 방식을 사용한다는 점을 확인하였다.

이 구조를 보며 나 역시 Hair, Face, PVC, Angel Ring 등의 Style Option을 Style Flag 형태로 저장할 수 있겠다는 아이디어를 얻었다.

다만 Material Category처럼 서로 배타적인 속성은 Material ID로 관리하고, Angel Ring 사용 여부나 Rim Light 사용 여부처럼 동시에 여러 개가 활성화될 수 있는 옵션만 Style Flag로 관리하는 것이 더 적절하다고 판단하였다.

---

GBuffer의 슬롯 구성도 분석하였다.

기본 GBuffer는 총 세 개의 Render Target을 사용하며,

GBuffer0은 BaseColor와 Material Flags,

GBuffer1은 Specular 또는 Metallic과 Occlusion,

GBuffer2는 Normal과 Smoothness를 저장한다.

이후 Depth, ShadowMask, RenderingLayers는 활성화된 기능에 따라 동적으로 추가된다.

이를 분석하면서 현재 GBuffer에는 남는 공간이 거의 없다는 점을 확인하였다.

따라서 내 연구에서는 Metallic, Specular, Smoothness처럼 Stylized Shader에서 사용하지 않는 PBR 채널을 Style Data 저장 공간으로 재활용하거나, 최종적으로는 Style GBuffer를 별도로 추가하는 방향을 고려하게 되었다.

---

GBufferData 구조체 역시 분석하였다.

이 구조체는 GBuffer 텍스처에 압축되어 저장된 데이터를 Deferred Lighting이 사용하기 쉽게 풀어놓은 구조체이다.

BaseColor, Smoothness, SpecularColor, Occlusion, Normal, Material Flags, Depth, ShadowMask, RenderingLayers 등이 모두 포함되어 있었다.

특히 구조체에 필드를 추가한다고 실제 GBuffer가 확장되는 것은 아니라는 점도 확인하였다.

새로운 데이터를 저장하려면 GBufferOutput, GBufferInput, GBufferCommon, Deferred Pass를 모두 수정해야 한다는 것을 이해하였다.

---

Material Flags는 GBuffer의 A 채널에 저장되기 때문에 PackGBufferMaterialFlags()에서 0~255 범위의 정수를 0~1 범위의 실수로 압축하고, Deferred Lighting에서는 다시 UnpackGBufferMaterialFlags()를 이용하여 원래의 uint 값으로 복원한다는 것도 확인하였다.

또한 Normal은 Octahedral Encoding을 이용하여 압축 저장할 수 있다는 것도 조사하였다.

Octahedral Encoding은 원래 3차원 법선 벡터를 2차원 좌표로 변환하여 저장하는 방식이며, 같은 저장 공간에서 방향 오차를 줄일 수 있기 때문에 Deferred Renderer에서 널리 사용되는 방식이라는 것을 이해하였다.

다만 내 연구에서는 Normal 자체를 수정하는 것보다 Normal을 이용해 Angel Ring, Rim Light, Toon Shadow를 계산하는 것이 훨씬 중요하므로, 이 부분은 원리 정도만 이해하고 넘어가기로 결정하였다.

---

# 새롭게 이해한 점

Lighting.hlsl은 실제 라이팅 계산을 수행하는 파일이라기보다는 여러 라이팅 함수를 조립하여 최종 결과를 생성하는 관리자 역할이라는 점을 이해하였다. PBR과 Blinn-Phong 역시 전체 구조는 거의 동일하며 실제 차이는 LightingPhysicallyBased()와 CalculateBlinnPhong() 같은 재질 계산 함수뿐이라는 것도 확인하였다.

또한 GBuffer는 단순히 여러 텍스처가 아니라 Deferred Rendering 전체에서 사용하는 표준 데이터 형식이며, GBufferCommon.hlsl은 그 저장 규칙 자체를 정의하는 파일이라는 점도 새롭게 이해하였다.

Vertex Shader는 계산을 수행하는 곳이 아니라 Fragment Shader가 사용할 데이터를 준비하는 단계이며, Toon Shadow와 Angel Ring처럼 정확한 경계가 필요한 효과는 반드시 Fragment Shader에서 계산해야 한다는 것도 다시 정리하였다.

---

# 연구에 적용할 방향

앞으로 제작할 Stylized Deferred Renderer는 기존 URP Deferred Pipeline을 유지한 상태에서 LightingPhysicallyBased()를 LightingStylized()로 교체하는 방향으로 설계할 예정이다.

또한 StyleData와 StyleLightingData를 새롭게 정의하여 Hair, Face, PVC, Angel Ring, Rim Light 등을 각각 독립된 Layer로 계산한 뒤 마지막 단계에서 Combine Rule(Add, Multiply, Lerp, Max)을 이용하여 원하는 방식으로 결합하는 구조를 사용할 계획이다.

초기 구현에서는 기존 GBuffer의 Metallic, Specular, Smoothness 채널을 재활용하여 Style Data를 저장하고, 이후 필요하다면 Style 전용 GBuffer를 추가하는 방향으로 확장할 예정이다.

또한 Angel Ring은 Clear Coat처럼 별도의 Layer로 계산하여 빛 방향에 따라 움직이는 독립적인 반사 효과로 구현하는 방향을 우선적으로 실험할 예정이다.
