Anime Stylized Rendering Pipeline 연구 기록

Unity 6 URP 기반 캐릭터용 NPR(Non-Photorealistic Rendering) 셰이더 제작 기록
최종 갱신: 2026-08-19

1. 프로젝트 목표

기본 URP Lit 셰이더를 확장하여 애니메이션 및 Live2D풍 캐릭터 렌더링 파이프라인을 구현한다.

핵심 목표는 다음과 같다.

명암 경계를 직접 조절할 수 있는 셀 셰이딩

얼굴과 머리카락 등 부위마다 다른 조명 규칙 적용

얼굴에 불필요한 그림자를 줄이고 코·목 등 필요한 위치에만 그림자 표현

머리카락용 엔젤링 및 스타일라이즈드 하이라이트

그림자 영역의 환경광·반사광 표현

캐릭터 내부선과 전체 실루엣 외곽선을 분리한 이중 외곽선

향후 표정과 카메라 방향에 반응하는 동적 표현으로 확장

2. 현재 셰이더 구조

주요 파일

StylizedLit.shader

머티리얼 Properties와 URP Pass 정의

UnityPerMaterial CBUFFER 선언

외곽선 Pass 포함

StylizedLighting.hlsl

툰 명암 및 부위별 라이팅 계산

얼굴 그림자와 코 그림자 실험 코드

NoseShadowAnchorBinder.cs

코 기준 Transform의 월드 좌표를 셰이더에 전달하는 실험용 컴포넌트

렌더링 흐름

현재 파이프라인은 URP Lit 구조를 기반으로 다음 패스를 사용한다.

Shadow

GBuffer 또는 Forward Lit

Depth 관련 Pass

Motion Vector 관련 Pass

추가 SRPDefaultUnlit 외곽선 Pass

3. 구현 완료 항목

3.1 기본 셀 셰이딩

조명과 노멀의 내적을 기준으로 명암을 두 단계로 나누고, 경계 위치와 부드러움을 머티리얼에서 조절할 수 있게 만들었다.

_ShadowThreshold("Shadow Threshold", Range(-1, 1)) = 0
_ShadowSoftness("Shadow Softness", Range(0, 0.5)) = 0.02
_ShadowColor("Shadow Color", Color) = (0.6, 0.6, 0.7, 1)

Shadow Threshold: 그림자가 시작되는 기준

Shadow Softness: 명암 경계의 부드러움

Shadow Color: 그림자 영역에 곱해지는 색

초기에는 ShaderLab의 Properties와 HLSL 변수 선언 위치가 섞이면서 구문 오류가 발생했다. 특히 HLSL 타입 선언 뒤 세미콜론이 빠져 Inspector 표시가 깨졌으나, 선언부를 수정하여 해결했다.

3.2 Material ID 분기

하나의 셰이더 안에서 재질 부위별 조명 규칙을 다르게 적용하기 위해 Material ID를 추가했다.

[Enum(Default, 0, Face, 1, Hair, 2)]
_StyleMaterialID("Style Material ID", Float) = 0

구분은 다음과 같다.

ID

부위

역할

0

Default

기본 툰 라이팅

1

Face

얼굴 전용 그림자 계산

2

Hair

엔젤링 및 머리카락 전용 효과

캐릭터가 여러 머티리얼을 사용하므로 얼굴·머리카락·의상 머티리얼에 새 셰이더를 각각 적용했다.

3.3 얼굴 방향 기반 그림자

얼굴은 일반적인 노멀 기반 그림자가 눈과 입 주변을 지저분하게 만들 수 있어, 얼굴 정면 및 좌우 방향과 광원 방향의 관계를 이용하는 별도 계산을 실험했다.

얼굴이 광원을 바라보는 정도 계산

좌우 방향에 따라 그림자 방향 전환

얼굴 그림자 색을 일반 그림자와 분리

눈과 입 주변의 불필요한 음영을 줄이고 목·코 중심으로 제한하는 방향으로 설계

현재 얼굴용 각도 값과 SDF 값 일부는 계산되지만 최종 toonMask에 완전히 반영되지 않은 구간이 있으므로 후속 정리가 필요하다.

3.4 머리카락 엔젤링

머리카락에 별도의 엔젤링 텍스처를 적용했다.

카메라 방향에 따라 하이라이트 위치가 이동

머리 회전에 반응

머티리얼 Inspector에서 강도 조절 가능

얼굴과 다른 Material ID 분기에서만 작동

현재 엔젤링의 기본 이동과 회전 반응은 정상적으로 동작한다.

3.5 환경광 및 반사광

그림자 영역이 완전히 죽지 않도록 스타일라이즈드 환경광과 반사광을 추가했다.

머리 아래쪽 절반에 환경광 적용

그림자 영역 내부로 효과 범위 제한

반사광은 주로 그림자 영역과 모델 상부 일부에서 작동

기본 Lit 반사광보다 일러스트에 가까운 색면을 목표로 조정

4. 코 그림자 실험

4.1 UV 기반 다이아몬드 마스크

처음에는 얼굴 UV 중심에 다이아몬드 모양의 절차적 마스크를 만들었다.

half diamondDistance =
    abs(noseP.x) / noseWidth +
    abs(noseP.y) / noseHeight;

그러나 UV 중앙이 실제 코 위치와 정확히 일치하지 않아 얼굴 중앙을 넓게 덮는, 이른바 '괴도 가면' 형태가 나타났다.

문제점

모델마다 얼굴 UV 배치가 다름

표정이나 메시 구조에 따라 마스크 위치가 어긋남

코가 아니라 얼굴 전체에 큰 다이아몬드 그림자가 생성됨

따라서 UV 고정 좌표 방식은 중단했다.

4.2 Transform Anchor 기반 방식

코끝 근처에 빈 GameObject를 생성하고 이름을 NoseShadowAnchor로 지정했다. 고개를 돌려도 함께 움직이도록 최종적으로는 실제 Head 본 아래에 배치하는 것을 전제로 했다.

NoseShadowAnchorBinder.cs는 Anchor의 월드 위치를 MaterialPropertyBlock으로 전달한다.

using UnityEngine;

[ExecuteAlways]
public class NoseShadowAnchorBinder : MonoBehaviour
{
    [SerializeField] private Renderer faceRenderer;
    [SerializeField] private Transform noseAnchor;

    private MaterialPropertyBlock propertyBlock;

    private static readonly int NoseCenterWS =
        Shader.PropertyToID("_NoseCenterWS");

    private void OnEnable() => UpdateShaderValues();
    private void OnValidate() => UpdateShaderValues();
    private void LateUpdate() => UpdateShaderValues();

    private void UpdateShaderValues()
    {
        if (faceRenderer == null || noseAnchor == null)
            return;

        propertyBlock ??= new MaterialPropertyBlock();
        faceRenderer.GetPropertyBlock(propertyBlock);

        Vector3 p = noseAnchor.position;
        propertyBlock.SetVector(
            NoseCenterWS,
            new Vector4(p.x, p.y, p.z, 1f)
        );

        faceRenderer.SetPropertyBlock(propertyBlock);
    }
}

셰이더의 UnityPerMaterial CBUFFER에는 다음 선언을 추가했다.

float4 _NoseCenterWS;

초기에는 Anchor의 좌우·상하·전방 축도 C#에서 전달하려 했다.

_NoseRightWS
_NoseUpWS
_NoseForwardWS

하지만 방향 벡터가 셰이더에서 0벡터로 읽혀 모든 픽셀의 상대 좌표가 0으로 계산되었고, 그 결과 얼굴 전체가 그림자로 판정됐다.

이를 확인하기 위해 각 방향 벡터의 길이를 검사하는 anchorValid 안전장치를 추가했다. 안전장치를 넣자 그림자가 완전히 사라졌으므로 방향값 전달 실패를 확인할 수 있었다.

이후 방향축은 C#에서 전달하지 않고 셰이더에서 오브젝트 축을 월드 방향으로 변환하도록 변경했다.

float3 noseRightWS = normalize(
    TransformObjectToWorldDir(float3(1.0, 0.0, 0.0))
);

float3 noseUpWS = normalize(
    TransformObjectToWorldDir(float3(0.0, 1.0, 0.0))
);

float3 noseForwardWS = normalize(
    TransformObjectToWorldDir(float3(0.0, 0.0, 1.0))
);

4.3 발생했던 오류

_NoseCenterWS를 찾을 수 없음

방향 변수 세 개를 제거하는 과정에서 _NoseCenterWS까지 함께 삭제하여 발생했다. StylizedLighting.hlsl이 해당 값을 사용하기 전에 StylizedLit.shader의 CBUFFER에서 선언되도록 복구했다.

anchorValid를 찾을 수 없음

anchorValid 계산부는 삭제했지만 마스크 곱셈에 변수명만 남아 발생했다.

half proceduralNoseMask =
    diamondMask *
    depthMask *
    styleData.noseShadowStrength;

위처럼 남은 anchorValid 참조를 제거하여 컴파일 오류를 해결했다.

4.4 현재 상태

컴파일 오류는 해결되었지만 Anchor 기반 코 그림자는 아직 화면에 나타나지 않는다. _NoseCenterWS.w를 이용해 MaterialPropertyBlock 전달 여부를 확인하려던 시점에서 코 그림자 작업을 잠시 중단했다.

재개 시 첫 검증 코드는 다음과 같다.

toonMask = 1.0h - step(0.5h, _NoseCenterWS.w);

판정 기준:

얼굴 전체가 그림자가 됨: _NoseCenterWS 전달 성공

아무 변화 없음: 해당 Renderer에 Property Block이 전달되지 않음

코 그림자 코드는 삭제하지 않고 비활성화한 상태로 유지한다.

5. 외곽선 구현

5.1 Inverted Hull 방식

StylizedLit.shader의 SubShader 마지막, XRMotionVectors Pass 다음에 외곽선 Pass를 추가했다.

Pass
{
    Name "OutlineInner"
    Tags
    {
        "LightMode" = "SRPDefaultUnlit"
    }

    Cull Front
    ZWrite On
    ZTest LEqual

    HLSLPROGRAM

    #pragma target 2.0
    #pragma vertex OutlineVertex
    #pragma fragment OutlineFragment

    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

    struct OutlineAttributes
    {
        float4 positionOS : POSITION;
        float3 normalOS   : NORMAL;
    };

    struct OutlineVaryings
    {
        float4 positionCS : SV_POSITION;
    };

    OutlineVaryings OutlineVertex(OutlineAttributes input)
    {
        OutlineVaryings output;

        float outlineWidth = 0.002;
        float3 expandedPositionOS =
            input.positionOS.xyz +
            normalize(input.normalOS) * outlineWidth;

        output.positionCS =
            TransformObjectToHClip(expandedPositionOS);

        return output;
    }

    half4 OutlineFragment(OutlineVaryings input) : SV_Target
    {
        return half4(0.08h, 0.03h, 0.12h, 1.0h);
    }

    ENDHLSL
}

작동 원리는 다음과 같다.

메시를 노멀 방향으로 조금 부풀린다.

Cull Front로 앞면을 제거한다.

본체 뒤에서 드러나는 뒷면만 단색으로 그린다.

외곽선 자체는 정상적으로 표시되었다.

5.2 이중 톤 실험

하나의 외곽선 안에서 바깥쪽을 어둡게 만들기 위해 노멀과 시선 방향의 내적으로 색을 섞는 방식도 검토했다.

그러나 이 방식은 정확히 폭이 나뉜 두 개의 선이 아니라 한 선 내부의 그라데이션에 가깝기 때문에 최종 목표와 맞지 않았다.

5.3 굵은 Hull을 하나 더 만드는 방식의 문제

기존 Pass의 폭을 0.004로 늘리고 어두운 색을 적용하면 캐릭터 바깥뿐 아니라 다음 경계까지 모두 굵어진다.

얼굴과 머리카락 사이

머리카락 조각 사이

얼굴과 목 사이

서로 다른 메시 또는 머티리얼 경계

즉 각 메시가 개별적으로 부풀기 때문에 캐릭터 전체를 누끼 딴 것 같은 단일 실루엣이 되지 않는다.

따라서 현재 Hull Pass는 다시 다음 용도로 확정했다.

OutlineInner
폭: 0.002
색: 보라색
역할: 머리카락·얼굴·옷 사이의 얇은 내부선

6. 최종 외곽선 목표

원하는 결과는 성격이 다른 외곽선 두 개를 동시에 사용하는 것이다.

외곽선

표현

구현 방식

내부선

머리카락·얼굴·옷 경계의 얇은 보라색 선

현재의 Inverted Hull OutlineInner

바깥선

캐릭터 전체를 누끼 딴 것처럼 감싸는 굵고 진한 선

캐릭터 전체 마스크 기반 Screen-space Outline

굵은 바깥선은 개별 메시를 부풀리는 방식이 아니라 다음 순서로 구현해야 한다.

캐릭터의 모든 메시를 하나의 화면 마스크로 합친다.

마스크를 화면 공간에서 바깥쪽으로 확장한다.

원본 마스크 영역을 빼서 바깥 테두리만 남긴다.

이를 어두운 외곽선 색으로 합성한다.

기존 OutlineInner는 그대로 유지한다.

처음에는 CharacterOutline 전용 Layer와 Renderer Feature를 Unity Inspector에서 설정하는 방식을 고려했다. 이후 작업 방향을 Inspector 수작업보다 코드 중심으로 구성하는 방식으로 변경했다.

예정 구조:

StylizedLit.shader
└─ OutlineInner
   └─ 얇은 내부선

CharacterOutlineFeature.cs
└─ 캐릭터 마스크 렌더링
└─ 마스크 팽창
└─ 원본 영역 제거
└─ 굵은 실루엣 외곽선 합성

CharacterOutlineFeature.cs는 아직 생성 및 구현 전이다.

7. 현재 정확한 작업 상태

정상 작동

기본 셀 셰이딩

그림자 Threshold 및 Softness 조절

Material ID에 따른 Face/Hair/Default 분기

얼굴 전용 그림자 색

머리카락 엔젤링

환경광 및 반사광

Inverted Hull 기반 내부 외곽선

실험 중단 또는 미완성

Transform Anchor 기반 코 그림자

얼굴 SDF와 코 그림자의 최종 결합

캐릭터 전체 누끼 형태의 굵은 실루엣 외곽선

외곽선 굵기와 색의 Inspector 프로퍼티화

다음 작업

CharacterOutlineFeature.cs 구현

캐릭터 전체 마스크 생성

마스크 기반 굵은 바깥 외곽선 합성

기존 OutlineInner와 동시에 표시되는지 확인

외곽선 색과 굵기를 Inspector 또는 Volume에서 조절 가능하게 변경

이후 _NoseCenterWS 전달 테스트부터 코 그림자 작업 재개

8. 작업 중 얻은 결론

셰이더 변수 선언과 값 전달은 별개다

CBUFFER에 변수를 선언하는 것은 셰이더가 값을 받을 공간을 만드는 것뿐이다. 실제 값은 머티리얼, 전역 셰이더 변수 또는 MaterialPropertyBlock을 통해 별도로 전달해야 한다.

0벡터를 normalize하면 디버깅이 어려워진다

외부에서 전달되는 방향 벡터가 유효한지 먼저 길이를 검사해야 한다. 잘못된 값이 모든 픽셀에 동일한 마스크를 만드는 경우 크기 조절만 반복해서는 원인을 찾을 수 없다.

메시별 Hull과 전체 실루엣은 다른 문제다

Inverted Hull은 메시 내부 경계 표현에는 유용하지만, 여러 메시로 구성된 캐릭터 전체의 단일 바깥선을 만들기에는 부적합하다. 전체 실루엣은 메시가 아니라 렌더링된 마스크를 기준으로 처리해야 한다.

한 번에 여러 원인을 수정하지 않는다

마스크 크기, 좌표 전달, 방향축, 강도 값을 동시에 바꾸면 어떤 수정이 결과를 만들었는지 판단하기 어렵다. 이후에는 다음 순서로 검증한다.

값 전달 여부를 흑백 결과로 확인

좌표계 확인

마스크 크기 확인

강도와 색 적용

기존 라이팅과 결합
