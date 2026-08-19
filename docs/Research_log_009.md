[Anime_Stylized_Pipeline_Progress_2026-08-19.md](https://github.com/user-attachments/files/31204686/Anime_Stylized_Pipeline_Progress_2026-08-19.md)
# Anime Stylized Pipeline 개발 기록 — 외곽선·코 그림자·브러시 스타일

작성일: 2026-08-19  
환경: Unity 6.1 (`6000.1.8f1`), URP `17.1.0`, DirectX 12

## 1. 이번 작업 목표

- 기존 Inverted Hull 내부선 유지
- 캐릭터 전체를 누끼 딴 것 같은 굵은 외부 실루엣 추가
- 코를 가상의 원뿔처럼 취급하고 빛 방향에 따라 회전하는 코 그림자 구현
- 머티리얼을 하나씩 수정하지 않고 캐릭터 최상위 오브젝트의 컴포넌트에서 그림자 브러시 스타일 변경

## 2. 현재 결과 요약

| 기능 | 상태 | 구현 방식 |
| --- | --- | --- |
| 내부 외곽선 | 완료 | `StylizedLit.shader`의 Inverted Hull `OutlineInner` Pass |
| 외부 누끼선 | 완료 | RenderGraph에서 캐릭터 Renderer를 단일 마스크로 합친 뒤 화면 공간 팽창 |
| 외부선 굵기·색 | 완료 | `CharacterOutlineFeature` 설정값 |
| 내부선 굵기·색 | 완료 | `StylizedLit` 머티리얼 프로퍼티 |
| 코 Anchor 전달 | 완료 | `NoseShadowAnchorBinder` + `MaterialPropertyBlock` |
| 회전형 코 그림자 | 완료 | 얼굴 평면에 투영한 원뿔 부채꼴 마스크 + Main Light 방향 |
| 코 그림자 바깥 감쇠 | 완료 | 반지름 증가에 따른 `spreadFade` |
| 오브젝트별 그림자 스타일 선택 | 동작 | `StylizedArtStyleController` 컴포넌트 |
| 절차적 브러시 질감 | 프로토타입 | Dry Brush / Watercolor / Pencil / Ink |
| 작가 제작 브러시 텍스처 | 다음 작업 | Clip Studio Paint 흑백 심리스 마스크 예정 |

## 3. 생성·수정한 파일

```text
Assets/Stylized/
├─ CharacterOutlineTarget.cs
├─ CharacterOutlineFeature.cs
├─ CharacterSilhouetteOutline.shader
├─ NoseShadowAnchorBinder.cs
├─ StylizedArtStyleController.cs
├─ StylizedLit.shader
├─ StylizedLitInput.hlsl
└─ StylizedLighting.hlsl
```

## 4. 내부선과 외부 누끼선

### 4.1 내부선

기존 Inverted Hull Pass는 각 메시를 노멀 방향으로 부풀려 뒷면을 렌더링한다. 머리카락·얼굴·옷 사이의 내부 경계까지 선으로 보이는 것이 목적이다.

```hlsl
float3 expandedPositionOS =
    input.positionOS.xyz +
    normalize(input.normalOS) *
    _InnerOutlineWidth;
```

프로퍼티:

```hlsl
_InnerOutlineWidth("Inner Outline Width", Range(0.0, 0.01)) = 0.002
_InnerOutlineColor("Inner Outline Color", Color) = (0.08, 0.03, 0.12, 1.0)
```

`OutlineVertex`의 출력 구조체에 사용하지 않는 필드가 남아 있으면 다음 경고가 발생할 수 있다.

```text
Output value 'OutlineVertex' is not completely initialized
```

이 경우 출력 선언 직후 초기화한다.

```hlsl
OutlineVaryings output;
ZERO_INITIALIZE(OutlineVaryings, output);
```

### 4.2 외부 누끼선

Inverted Hull 폭만 크게 만들면 모델을 구성하는 각 메시가 따로 팽창한다. 그 결과 캐릭터 바깥선뿐 아니라 머리카락·얼굴·옷 사이의 내부선도 모두 굵어진다.

외부선은 다음 구조로 분리했다.

1. `CharacterOutlineTarget`이 캐릭터 최상위 오브젝트 아래의 `MeshRenderer`와 `SkinnedMeshRenderer`를 자동 수집한다.
2. `CharacterOutlineFeature`가 URP 17.1 RenderGraph에서 수집된 Renderer를 흰색으로 다시 그린다.
3. 모든 부위가 같은 R8 임시 텍스처에 겹쳐지며 캐릭터 전체 실루엣 마스크가 된다.
4. `CharacterSilhouetteOutline.shader`가 마스크 바깥만 팽창시켜 카메라 컬러에 합성한다.

따라서 별도의 GameObject Layer나 수작업 마스크 텍스처가 필요하지 않다.

현재 권장 외부선 굵기:

```text
Outline Width: 2 px
```

Scene 뷰의 격자선은 렌더 패스 이후 에디터가 그리는 오버레이이므로 화면 공간 외곽선 위에 보일 수 있다. Game 뷰에서는 나타나지 않으며 실제 배경이 외곽선을 뚫은 것이 아니다.

## 5. 코 그림자

### 5.1 Binder 구조

`NoseShadowAnchorBinder`는 다음 작업을 자동화한다.

- 자식 `SkinnedMeshRenderer` 자동 검색
- `NoseAnchor`가 없으면 자동 생성
- Anchor의 월드 좌표를 `_NoseCenterWS`로 전달
- `MaterialPropertyBlock`을 사용해 같은 Renderer의 다른 프로퍼티 보존

`NoseAnchor`는 최종적으로 코끝에 배치한다.

```csharp
propertyBlock.SetVector(
    NoseCenterWS,
    new Vector4(position.x, position.y, position.z, 1f)
);
```

### 5.2 진단 과정

Face 분기 실행 여부:

```hlsl
toonMask = 0.0h;
```

얼굴 전체가 그림자가 되면 Face 분기는 정상이다.

Binder 전달 여부:

```hlsl
toonMask = 1.0h - step(0.5h, _NoseCenterWS.w);
```

얼굴 전체가 그림자가 되면 `_NoseCenterWS.w == 1`이므로 전달 성공이다. 아무 변화가 없으면 Binder가 씬 오브젝트에 붙어 있는지와 Renderer 연결을 확인한다.

### 5.3 실패한 방식과 원인

#### 고정 마름모

코 주변에 그림자는 생겼지만 빛 방향과 관계없이 고정되어 스티커처럼 보였다.

#### 아래 방향 원뿔

원뿔 축을 얼굴의 상하 방향으로 잡아 그림자가 코에서 입 쪽으로 자랐다. 코가 아니라 아래를 향한 고깔 형태가 되었다.

#### 실제 3D 원뿔 교차

가상의 원뿔 부피와 실제 얼굴 메시 위치를 직접 비교했다. 얼굴 메시 자체가 원뿔 표면을 이루지 않기 때문에 마스크가 거의 0이 되어 그림자가 사라졌다.

### 5.4 최종 방식: 투영된 원뿔 부채꼴

실제 원뿔 교차 대신 얼굴 평면에 원뿔의 밑면을 타원으로 투영하고, 코끝에서 바깥으로 향하는 각도를 계산한다. Main Light를 같은 좌우·상하 평면에 투영한 후 빛 반대 방향의 부채꼴만 그림자로 사용한다.

```hlsl
half noseX = dot(noseDeltaWS, noseRightWS);
half noseY = dot(noseDeltaWS, noseUpWS);
half noseZ = dot(noseDeltaWS, noseForwardWS);

half noseWidth = 0.10h;
half noseHeight = 0.14h;
half noseDepth = 0.45h;
half noseShadowStrength = 0.55h;

float2 conePlanePosition =
    float2(
        noseX / noseWidth,
        noseY / noseHeight
    );

half coneRadius = length(conePlanePosition);

half coneShapeMask =
    1.0h - smoothstep(0.78h, 1.0h, coneRadius);

half depthMask =
    1.0h -
    smoothstep(
        noseDepth * 0.5h,
        noseDepth,
        abs(noseZ)
    );

float2 coneDirection =
    conePlanePosition /
    max(length(conePlanePosition), 0.0001);

Light noseMainLight = GetMainLight();

float2 noseLightDirection =
    float2(
        dot(noseMainLight.direction, noseRightWS),
        dot(noseMainLight.direction, noseUpWS)
    );

half lightDirectionLength = length(noseLightDirection);

noseLightDirection =
    lerp(
        float2(1.0, 0.0),
        noseLightDirection /
        max(lightDirectionLength, 0.0001),
        step(0.0001, lightDirectionLength)
    );

half coneLightDot =
    dot(coneDirection, noseLightDirection);

half rotatingShadowMask =
    1.0h -
    smoothstep(-0.75h, -0.40h, coneLightDot);
```

넓게 펼쳐지는 쪽은 농도가 점차 감소한다.

```hlsl
half spreadRatio = saturate(coneRadius);

half spreadFade =
    lerp(
        1.0h,
        0.15h,
        smoothstep(0.20h, 1.0h, spreadRatio)
    );

half proceduralNoseMask =
    coneShapeMask *
    depthMask *
    rotatingShadowMask *
    spreadFade *
    noseShadowStrength;

toonMask = 1.0h - proceduralNoseMask;
```

현재 결과는 Directional Light를 회전하면 작은 부채꼴 그림자가 코끝 주변을 돌며, 꼭짓점은 진하고 넓어지는 끝부분은 흐려진다.

## 6. 오브젝트 단위 그림자 브러시 선택

### 6.1 목표

78개 이상의 머티리얼을 하나씩 수정하지 않고 캐릭터 최상위 오브젝트에서 한 번에 그림체를 바꾼다.

`StylizedArtStyleController`를 `Hiyori_Tomoe` 같은 캐릭터 루트에 Add Component로 붙인다. 컴포넌트는 모든 자식 Renderer를 수집하고 다음 값을 `MaterialPropertyBlock`으로 전달한다.

```hlsl
float _ShadowBrushStyle;
float4 _ShadowBrushParams;
float _ShadowBrushSeed;
```

`_ShadowBrushParams`:

```text
x = Brush Scale
y = Edge Roughness
z = Pigment Breakup
w = Brush Softness
```

스타일 목록:

```csharp
public enum ShadowBrushStyle
{
    Clean = 0,
    DryBrush = 1,
    Watercolor = 2,
    Pencil = 3,
    Ink = 4
}
```

### 6.2 셰이더 연결 위치

`ApplyShadowBrush()`는 Face/Hair/기본 재질 분기가 모두 끝난 뒤, 최종 `litColor`와 `toonColor`를 계산하기 직전에 한 번 호출한다.

```hlsl
// Face/Hair 분기의 마지막 중괄호 다음
toonMask =
    ApplyShadowBrush(
        toonMask,
        inputData.positionWS
    );

half3 litColor =
    surfaceData.albedo * mainLight.color;

half3 toonColor =
    lerp(shadowColor, litColor, toonMask);
```

이 위치에 두어야 얼굴뿐 아니라 머리와 옷에도 같은 브러시가 적용된다.

현재 절차적 구현은 Value Noise와 FBM을 이용해 다음 효과를 만든다.

- Clean: 기존 셀 셰이딩
- Dry Brush: 그림자 안료가 군데군데 끊김
- Watercolor: 저주파 얼룩과 부드러운 농도 변화
- Pencil: 사선 해칭과 미세 입자
- Ink: 강하고 불규칙한 먹 번짐

기능 전환은 정상 작동하지만 절차적 노이즈만으로는 원하는 작가적 붓 방향과 형태가 부족하다. 현재 결과는 프로토타입으로 유지하고, 다음 단계에서 직접 제작한 텍스처를 주 입력으로 사용한다.

## 7. 다음 브러시 텍스처 제작 규격

Clip Studio Paint에서 스타일별 심리스 흑백 마스크를 제작한다.

```text
크기        1024 × 1024
색상        Grayscale
흰색        그림자 안료가 남는 영역
검은색      그림자가 벗겨지는 영역
배경        불투명
파일        PNG
타일링      상하좌우 Seamless 권장
```

우선 제작할 텍스처:

```text
ShadowBrush_Dry.png
ShadowBrush_Watercolor.png
ShadowBrush_Pencil.png
```

Unity Import 설정:

```text
Texture Type  Default
sRGB          Off
Wrap Mode     Repeat
Filter Mode   Bilinear
Compression   None (개발 단계)
```

최종 셰이더에서는 작가 제작 텍스처를 주 패턴으로 사용하고, 현재 FBM 노이즈는 UV 왜곡과 반복 무늬 완화용 보조값으로 사용한다.

## 8. 현재 알려진 경고·주의점

### OnValidate 경고

```text
SendMessage cannot be called during Awake, CheckConsistency, or OnValidate
(Hiyori_Tomoe: OnTransformChildrenChanged)
```

`NoseShadowAnchorBinder`가 `OnValidate` 중 Anchor를 생성하면서 Hierarchy가 변경되고, 동시에 `CharacterOutlineTarget.OnTransformChildrenChanged()`가 Renderer 목록을 갱신하여 발생한다. 현재 렌더 결과에는 영향을 주지 않지만 이후 다음 중 하나로 정리한다.

- `OnTransformChildrenChanged()`에서 즉시 갱신하지 않고 에디터 지연 호출 사용
- 해당 콜백 제거 후 Context Menu의 `Refresh Renderers` 사용
- 런타임과 에디터 갱신 경로 분리

### MaterialPropertyBlock 공유

`NoseShadowAnchorBinder`와 `StylizedArtStyleController`는 모두 같은 Renderer의 `MaterialPropertyBlock`을 사용한다. 기존 블록을 먼저 `GetPropertyBlock()`으로 읽고 자신의 값만 변경한 뒤 다시 설정해야 서로의 값이 사라지지 않는다.

### 모델 크기

현재 모델은 약 20 Unity Unit 규모다. 코 그림자 크기 값은 이 모델 스케일에 맞춰져 있으므로 다른 모델 검증 전에는 범용값으로 간주하지 않는다.

## 9. 다음 작업 순서

1. Clip Studio Paint에서 `ShadowBrush_Dry.png` 제작
2. `StylizedArtStyleController`에 스타일별 `Texture2D` 슬롯 추가
3. 선택된 텍스처를 `_ShadowBrushTexture`로 모든 자식 Renderer에 전달
4. 셰이더에서 텍스처와 FBM 노이즈를 결합
5. Watercolor/Pencil/Ink 텍스처 확장
6. 하이라이트를 그림자 뒤가 아니라 항상 최종 전면에 합성
7. 다른 모델로 외곽선·코 Anchor·브러시 스케일 범용성 검증
8. 수채화·연필·흑백 일러스트 등 전체 화면 스타일 Pass 설계
9. RenderGraph 텍스처 샘플 수, Renderer 수집 할당, 셰이더 분기 최적화

## 10. 추천 커밋

한 번에 올릴 경우:

```text
feat: add silhouette outline and light-responsive nose shading
```

작업을 나눠 기록할 경우:

```text
feat: add render graph silhouette outline for stylized characters
feat: add light-responsive projected cone nose shadow
feat: add object-level shadow brush style controller
docs: document stylized outline and brush rendering experiments
```

## 11. 체크포인트

현재 안정적으로 사용할 수 있는 기준 상태:

- 내부 Inverted Hull 선과 외부 화면 공간 누끼선이 동시에 표시됨
- 캐릭터 루트 컴포넌트에서 외부선 설정 가능
- `NoseAnchor`를 코끝에 두면 Directional Light에 따라 코 그림자가 회전함
- 코 그림자 바깥쪽에 자연스러운 농도 감쇠 적용
- 캐릭터 루트의 `StylizedArtStyleController`에서 그림자 스타일 변경 가능
- 절차적 브러시 결과는 기능 검증용이며 최종 비주얼은 직접 제작 텍스처로 교체 예정

