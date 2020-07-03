# VRMC_materials_mtoon 

## Contributors

## Dependencies

Written against the glTF 2.0 spec.

## Overview

この拡張は、 VRM 向けのトゥーンシェーダ実装です。
トゥーンシェーダ的な効果を付与しつつ、 PBR のシーンライティングと協調することを目的とします。
日本の手描きアニメ表現への特化はしません。

## Extending Materials

マテリアルの `extensions` に定義します。

```json
{
    "materials": [
        {
            "name": "MyUnlitMaterial",
            "pbrMetallicRoughness": {
                "baseColorFactor": [ 0.5, 0.8, 0.0, 1.0 ]
                // texture
            },
            // emission

            "extensions": {
                "VRMC_materials_mtoon": {

                }
            }
        }
    ]
}
```

## Definition

### Meta
#### MToon Defined Properties
|               |  型  |           説明           |
| :------------ | :--- | :----------------------- |
| VersionNumber | int  | この拡張のバージョン番号 |



### Rendering
描画に関する MToon の定義を述べます。

#### Dependencies
- [`alphaMode`](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#alpha-coverage)
- [`doubleSided`](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#double-sided)

#### MToon Defined Properties
|                         |  型  |                         説明                          |
| :---------------------- | :--- | :---------------------------------------------------- |
| TransparentWithZWrite   | bool | RenderMode が TransparentWithZWrite であるとき `true` |
| RenderQueueOffsetNumber | int  | RenderMode のデフォルト描画順に対するオフセット値     |

#### RenderMode
このマテリアルがどのようなアルファ処理で描画されるのかを指定します。
モードとして次の 4 つが存在します。

- Opaque - 完全不透明で描画します。アルファ値は無視します。
- Cutout - アルファ値とアルファカットオフ値を基に、完全不透明・完全透明のどちらかで描画します。
- Transparent - アルファ値を基に、背景色とブレンドして描画します。
- TransparentWithZWrite - 描画の仕方は Transparent と同じですが、加えて ZBuffer に対して書き込みを行います。

これらのモードは、glTF の `alphaMode` および MToon の `TransparentWithZWrite` の 2 つの組み合わせで定義されます。
これについて次の表に示します。

|                       | `alphaMode` | `TransparentWithZWrite` |
| :-------------------- | :---------- | :---------------------- |
| Opaque                | `OPAQUE`    | `false`                 |
| Cutout                | `MASK`      | `false`                 |
| Transparent           | `BLEND`     | `false`                 |
| TransparentWithZWrite | `BLEND`     | `true`                  |

また、実装によって TransparentWithZWrite が困難なときは Transparent にフォールバックしてください。

#### CullMode
このマテリアルがどのようなカリング処理で描画されるのかを指定します。
これは glTF の `doubleSided` に準拠します。

#### RenderQueue
このマテリアルがどのような順番で描画されるべきなのかを指定します。
MToon は次のような順番で描画されることを期待します。

1. Opaque
2. Cutout
3. TransparentWithZWrite
4. Transparent

トゥーンシェーダでは残念ながら半透明表現が多用されるため、描画順序に起因する問題が発生しがちです。
MToon ではそのために TransparentWithZWrite を導入していますが、さらに描画順制御の仕組みを導入しています。
`RenderQueueOffsetNumber` はそれぞれの RenderMode のデフォルト描画順に対するオフセット値です。
`RenderQueueOffsetNumber` は RenderMode が `Transparent` `TransparentWithZWrite` のときに作用します。
値が大きいほど、描画の順番は後回しにされます。
値のとりうる範囲について次の表に示します。

|                       | Min Value | Max Value |
| :-------------------- | :-------- | :-------- |
| Opaque                | 0         | 0         |
| Cutout                | 0         | 0         |
| Transparent           | -9        | 0         |
| TransparentWithZWrite | 0         | +9        |

また、どのような値になったとしても、前述の RenderQueue による描画の順番の方が優先されます。
この動作について次の表で例示します。

| Order |      RenderMode       | RenderQueueOffsetNumber |
| :---- | :-------------------- | :---------------------- |
| 1     | Opaque                | N/A                     |
| 2     | Cutout                | N/A                     |
| 3     | TransparentWithZWrite | 0                       |
| 4     | TransparentWithZWrite | +9                      |
| 5     | Transparent           | -9                      |
| 6     | Transparent           | 0                       |

RenderQueueOffset の実装が困難なときは `RenderQueueOffsetNumber` が `0` であるとみなしてください。



### Color
色に関する定義について述べます。

#### Dependencies
- [`pbrMetallicRoughness.baseColorFactor`](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#metallic-roughness-material)
- [`pbrMetallicRoughness.baseColorTexture`](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#metallic-roughness-material)
- [`alphaCutoff`](https://github.com/KhronosGroup/glTF/tree/master/specification/2.0#alpha-coverage)

#### MToon Defined Properties
|                       |   型   |                         説明                          |
| :-------------------- | :----- | :---------------------------------------------------- |
| TransparentWithZWrite | bool   | RenderMode が TransparentWithZWrite であるとき `true` |
| ShadeColor            | float4 | ShadeColor の色成分                                   |
| ShadeMultiplyTexture  | int    | ShadeColor のテクスチャ。Buffer の index を指す       |

#### Lit Color
Albedo を表します。
`pbrMetallicRoughness.baseColorFactor` および `pbrMetallicRoughness.baseColorTexture` に格納されます。

### Shade Color
光に当たらない部分の色合いを表します。
`ShadeColor` および `ShadeMultiplyTexture` に格納されます。

### Cutout Threshold
Cutout 処理におけるアルファ値のしきい値を表します。
これは glTF の `alphaCutoff` に準拠します。



### LightingDefinition
    
* public LitAndShadeMixingDefinition LitAndShadeMixing;
* public LightingInfluenceDefinition LightingInfluence;
    
### LitAndShadeMixingDefinition

* public float ShadingShiftValue;
* public float ShadingToonyValue;
* public float ShadowReceiveMultiplierValue;
* public Texture2D ShadowReceiveMultiplierMultiplyTexture;
* public float LitAndShadeMixingMultiplierValue;
* public Texture2D LitAndShadeMixingMultiplierMultiplyTexture;

### LightingInfluenceDefinition

* public float LightColorAttenuationValue;
* public float GiIntensityValue;

### EmissionDefinition

* public Color EmissionColor;
GLTFの emissionFactor に格納する

* public Texture2D EmissionMultiplyTexture;
GLTFの emissionMultiplyTexture/index に格納する

### MatCapDefinition

* public Texture2D AdditiveTexture;

### RimDefinition

* public Color RimColor;
* public Texture2D RimMultiplyTexture;
* public float RimLightingMixValue;
* public float RimFresnelPowerValue;
* public float RimLiftValue;

### NormalDefinition

* public Texture2D NormalTexture;
GLTFの normalTexture/index

* public float NormalScaleValue;
GLTFの normalTexture/scale

### OutlineDefinition

* public OutlineWidthMode OutlineWidthMode;
        None = 0,
        WorldCoordinates = 1,
        ScreenCoordinates = 2,

* public float OutlineWidthValue;
* public Texture2D OutlineWidthMultiplyTexture;
* public float OutlineScaledMaxDistanceValue;
* public OutlineColorMode OutlineColorMode;
        FixedColor = 0,
        MixedLighting = 1,

* public Color OutlineColor;
* public float OutlineLightingMixValue;

### TextureUvCoordsDefinition

* public Vector2 MainTextureLeftBottomOriginScale;
  KHR_texture_transform で引き受ける
* public Vector2 MainTextureLeftBottomOriginOffset;
  KHR_texture_transform で引き受ける

* public Texture2D UvAnimationMaskTexture;
* public float UvAnimationScrollXSpeedValue;
* public float UvAnimationScrollYSpeedValue;
* public float UvAnimationRotationSpeedValue;

## Extension compatibility and fallback materials

`KHR_materials_unlit` にフォールバックしてください。
