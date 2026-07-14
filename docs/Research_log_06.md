# Research Log - GBufferInput.hlsl 분석 및 Stylized Shader 구조 설계

## 연구 목표

이번 분석의 목표는 URP Deferred Rendering Pipeline에서 GBuffer가 어떠한 구조로 저장되고, 이후 Deferred Lighting 단계에서 어떤 과정을 거쳐 실제 라이팅 계산에 사용되는지를 이해하는 것이다. 기존에는 GBuffer를 단순히 렌더링 중간 결과를 저장하는 버퍼 정도로만 이해하고 있었으나, 실제 코드를 분석한 결과 GBuffer는 단순한 Color Buffer가 아니라 Material과 Surface의 모든 정보를 저장하는 중간 데이터베이스 역할을 수행한다는 점을 확인하였다. 또한 Deferred Rendering은 이 데이터를 바로 사용하는 것이 아니라 GBufferInput.hlsl에서 다시 해석하여 Lighting 단계로 전달하는 구조를 가지고 있었다. 따라서 본 연구에서는 기존 URP의 데이터 흐름을 정확하게 이해한 뒤, 이를 기반으로 물리 기반 라이팅 대신 Stylized Rendering을 위한 데이터 구조를 설계하는 것을 목표로 하였다.

---

## GBufferInput.hlsl의 역할 분석

처음에는 Deferred Lighting이 시작되면 바로 라이팅 계산이 이루어질 것이라고 생각하였다. 그러나 GBufferInput.hlsl를 분석한 결과 이 파일은 라이팅을 계산하는 파일이 아니라 GBuffer에 저장되어 있는 데이터를 Deferred Lighting에서 사용할 수 있는 형태로 복원하는 역할을 수행하고 있었다. 즉 GBuffer는 단순히 색상을 저장하는 버퍼가 아니라 Material과 Surface 정보를 저장하는 데이터베이스이며, Lighting 단계는 이 데이터를 직접 사용하는 것이 아니라 한 번 해석된 구조체를 통해 계산을 수행한다. 이러한 구조는 입력 계층과 계산 계층을 명확하게 분리하기 위한 설계이며, 향후 새로운 Buffer를 추가하거나 Material 정보를 확장하더라도 Lighting 코드를 크게 수정하지 않고 기능을 추가할 수 있다는 장점을 가진다. 따라서 본 연구에서도 기존 GBufferData를 확장하거나 별도의 Style Buffer를 추가하는 방향이 기존 구조와 가장 잘 어울린다고 판단하였다.

---

## GBuffer 입력 방식

가장 먼저 분석한 부분은 GBuffer를 읽는 방법이었다. URP는 플랫폼에 따라 Framebuffer Fetch와 Texture Sampling 두 가지 방식을 사용하고 있었다. Framebuffer Fetch가 가능한 플랫폼에서는 현재 Render Pass에서 생성된 Framebuffer를 직접 입력으로 사용하며, 지원하지 않는 플랫폼에서는 _GBuffer0, _GBuffer1, _GBuffer2 Texture를 읽는 구조를 사용한다. 두 방식은 구현 방법은 다르지만 최종적으로 생성되는 데이터는 동일하며 이후 Deferred Lighting 단계에서는 어떤 방법으로 읽혀 왔는지 전혀 신경 쓰지 않는 구조였다. 이는 입력 방식과 이후 계산을 완전히 독립시키기 위한 구조이며, 플랫폼이 달라져도 Lighting 코드의 수정이 거의 필요 없다는 장점을 가진다. 또한 향후 Style Buffer를 추가하는 경우에도 동일한 방식으로 Framebuffer Fetch와 Texture Sampling 두 경로를 모두 지원하도록 설계하면 기존 구조와 높은 호환성을 유지할 수 있을 것으로 판단하였다.

---

## LoadGBuffers()

LoadGBuffers() 함수는 이름 그대로 GBuffer의 원본 데이터를 읽어오는 역할만 수행한다. Base Color나 Normal과 같은 의미를 해석하지 않으며 단순히 GBuffer0, GBuffer1, GBuffer2, Depth, ShadowMask, Rendering Layers를 메모리에서 읽어오는 것이 전부이다. 즉 이 함수는 Raw Data를 가져오는 단계이며, 이후 데이터의 의미를 해석하는 작업은 다음 단계인 UnpackGBuffers()에서 수행된다. 이러한 구조를 통해 URP는 입력과 해석을 명확하게 분리하고 있으며, 입력 방식이 변경되더라도 이후 Deferred Lighting 코드는 영향을 받지 않는다. 이는 Layer를 분리한 설계의 대표적인 예라고 볼 수 있으며, 향후 연구에서 Style Buffer를 추가할 경우에도 동일한 계층 구조를 유지하는 것이 유지보수성과 확장성 측면에서 가장 적절하다고 판단하였다.

---

## UnpackGBuffers()

이번 분석에서 가장 중요했던 함수는 UnpackGBuffers()였다. 이 함수는 LoadGBuffers()에서 읽어온 Raw Data를 실제 Deferred Lighting에서 사용하는 GBufferData 구조체로 변환한다. 분석 결과 GBuffer0에는 Base Color와 Material Flags가 저장되고, GBuffer1에는 Specular 또는 Metallic 정보와 Occlusion이 저장되며, GBuffer2에는 Normal과 Smoothness가 저장되는 것을 확인하였다. 또한 Depth, ShadowMask, Rendering Layers 역시 하나의 구조체에 함께 저장된다. 즉 Deferred Lighting은 GBuffer를 Texture처럼 사용하는 것이 아니라 현재 픽셀의 Material 정보를 담고 있는 데이터베이스처럼 사용하는 구조였다. 따라서 향후 Material ID, Pattern ID, Style ID, Falloff Type과 같은 Stylized Rendering 전용 데이터를 추가하기 위해서는 별도의 Style Buffer를 생성하고 이 단계에서 함께 해석하는 방식이 기존 URP 구조와 가장 자연스럽게 연결될 것으로 판단하였다.

---

## Material Flags

Material Flags는 일반적인 Color 데이터가 아니라 여러 개의 옵션을 하나의 값에 저장하기 위한 Bit Flag 구조를 사용하고 있었다. 저장 과정에서는 Bit 정보를 하나의 채널에 Packing하여 저장하고, Deferred Lighting 단계에서 다시 Unpacking하여 원래의 Flag 정보를 복원하는 방식을 사용한다. 이를 통해 URP는 적은 저장 공간으로 다양한 Material 정보를 효율적으로 저장하고 있었다. 분석 과정에서 Hair, Face, Cloth, Latex와 같은 세부 Material 정보를 Material Flags에 저장하는 방법도 고려하였으나, 기존 URP가 이미 여러 Bit를 사용하고 있기 때문에 충돌 가능성이 존재한다는 점을 확인하였다. 따라서 Material Flags는 Style 사용 여부와 같은 간단한 상태 정보만 저장하고, 세부 Material ID는 별도의 Buffer에서 관리하는 것이 더욱 적절하다고 판단하였다.

---

## Normal

Normal은 GBuffer에 그대로 저장되지 않고 압축(Packing)된 형태로 저장된다. Deferred Lighting에서는 Unpack 과정을 통해 다시 원래의 Normal을 복원한 뒤 Normalize를 수행하여 길이를 1로 맞춘다. 이러한 과정은 저장 공간을 줄이기 위한 최적화이며, 복원 과정에서 발생할 수 있는 오차를 최소화하기 위해 Normalize가 추가로 수행된다. Normal은 이후 Rim Light, Highlight, Fresnel, Stroke 방향, Edge Detection 등 대부분의 NPR 표현에서 기준 벡터로 사용되므로 Stylized Rendering에서도 가장 중요한 입력 데이터 중 하나가 될 것으로 판단하였다. 따라서 향후 모든 Style 계산은 Normal을 기준으로 설계하는 방향이 적절하다고 판단하였다.

---

## Occlusion

기존 URP에서는 Occlusion을 Ambient Occlusion 계산에 사용하여 환경광이 얼마나 차단되는지를 표현한다. 그러나 본 연구에서는 Occlusion을 단순한 그림자 강도가 아니라 환경색(Environment Color)을 얼마나 받아들일 것인지를 결정하는 계수로 재해석하는 방향을 고려하였다. 예를 들어 Base Line Color와 Environment Color를 Occlusion 값으로 보간하면 별도의 Texture 없이도 주변 환경에 따라 선색이 자연스럽게 변화하는 구조를 만들 수 있을 것으로 예상하였다. 또한 Occlusion 값은 일반적으로 접촉부나 응집된 영역에서 높은 값을 가지므로 Stroke의 강도나 해칭의 밀도, 그림자 패턴의 강도를 조절하는 마스크로도 활용할 수 있을 것으로 판단하였다. 추가적으로 카메라와의 거리 정보를 함께 이용하면 거리 기반 선 두께 조절이나 Pattern LOD에도 활용 가능성이 있을 것으로 생각하였다.

---

## Smoothness

Smoothness는 기존 URP에서 Specular Highlight의 폭과 Roughness를 결정하기 위한 값이다. 그러나 Stylized Rendering에서는 이러한 물리적 의미보다 표현적인 의미가 더욱 중요하다. 따라서 Smoothness를 Highlight 폭뿐 아니라 Stroke의 굵기, Pattern의 밀도, Angel Ring의 폭, Latex Reflection의 집중도와 같은 아티스트 중심의 파라미터로 재해석할 수 있을 것으로 판단하였다. 기존 데이터 구조를 그대로 활용하면서도 표현 방식을 크게 변경할 수 있다는 점에서 프로토타입 제작 단계에서는 충분히 활용 가치가 있을 것으로 보인다.

---

## Depth와 Style LOD

Depth 정보는 원래 World Position을 복원하기 위한 용도로 사용되지만, Stylized Rendering에서는 표현 자체를 단순화하는 기준으로도 사용할 수 있을 것으로 판단하였다. 기존 연구 과정에서 카메라와 멀어질수록 캐릭터가 그림처럼 단순해지는 표현을 구상하였는데, 이를 Camera Distance와 Depth를 이용하여 구현할 수 있을 것으로 보인다. 예를 들어 거리가 멀어질수록 Stroke를 단순화하고 Pattern의 밀도를 줄이며 세부 재질 표현을 제거하는 Style LOD를 구성할 수 있다. 이는 기존 Geometry LOD와는 다른 개념으로, NPR Rendering만을 위한 새로운 표현 방식이 될 수 있을 것으로 기대된다.

---

## SurfaceData와 BRDFData

UnpackGBuffers() 이후 생성된 GBufferData는 Material 종류에 따라 SurfaceData 또는 BRDFData로 변환된다. SurfaceData는 SimpleLit에서 사용하는 단순한 데이터 구조이며 대부분의 정보를 그대로 복사하는 역할만 수행한다. 반면 BRDFData는 Reflectivity, Diffuse, Specular, Metallic, Roughness 등을 계산하여 물리 기반 라이팅에 최적화된 데이터 구조를 생성한다. 즉 GBufferData는 저장을 위한 구조이고, BRDFData는 계산을 위한 구조라고 이해할 수 있었다. 그러나 본 연구에서는 물리 기반 반사 계산보다 아티스트가 의도한 스타일 표현이 더욱 중요하므로 BRDFData 대신 StyleData를 생성하여 LightingStylized()로 전달하는 구조가 적절하다고 판단하였다. 즉 기존 URP의 GBufferData → BRDFData → LightingPhysicallyBased() 구조를 GBufferData → StyleData → LightingStylized() 구조로 변경하는 것이 본 연구의 핵심 방향이다.

---

## 연구 방향 및 향후 구현 계획

이번 분석을 통해 Deferred Rendering에서 가장 중요한 것은 Lighting 함수 자체보다 Lighting 이전에 어떤 데이터를 준비하느냐라는 점을 새롭게 이해하였다. 즉 Stylized Rendering의 핵심은 기존 BRDF 계산식을 수정하는 것이 아니라 Stylized Rendering에 필요한 데이터를 어떻게 생성하고 관리할 것인가에 있다. 따라서 향후 연구는 Style Texture 제작, Material Buffer 설계, Style Volume 생성, Style Mask 계산, Pattern 매핑, Falloff 설계, Environment Color 적용, Style LOD 구현, 최종 Blend Rule 설계 순으로 진행할 예정이다. 최종적으로는 기존의 물리 기반 BRDF를 완전히 대체하는 LightingStylized()를 구현하여 Live2D와 같은 아티스트 중심의 3D Stylized Rendering Pipeline을 구축하는 것을 목표로 한다.
