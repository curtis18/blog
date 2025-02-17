## [CustomShader文档](https://github.com/CesiumGS/cesium/blob/main/Documentation/CustomShaderGuide/README.md)

## 构造器

```JS
const customShader = new Cesium.CustomShader({
  //用户想要添加到着色器的任何自定义Uniforms。
  //这些可以在运行时通过 customShader.setUniform() 改变
  uniforms: {
    u_time: {
      value: 0,//初始值
      type: Cesium.UniformType.FLOAT
    },
    //可以从 URL、资源或 TypedArray 加载纹理。
    //更多细节参见 Uniforms 部分
    u_externalTexture: {
      value: new Cesium.TextureUniform({
        url: "http://example.com/image.png"
      }),
      type: Cesium.UniformType.SAMPLER_2D
    }
  }
  //将出现在自定义顶点和片段着色器文本中的自定义变量。
  varyings: {
    v_customTexCoords: Cesium.VaryingType.VEC2
  },
  //配置CustomShader在片段着色器的材质/照明管道中的位置。后面会有更多关于此的内容。
  mode: Cesium.CustomShaderMode.MODIFY_MATERIAL,
  //PBR（基于物理的渲染）或 UNLIT，具体取决于所需的结果。
  lightingModel: Cesium.LightingModel.PBR,
  //强制着色器渲染为透明，即使图元具有不透明材质
  translucencyMode: Cesium.CustomShaderTranslucencyMode.TRANSLUCENT,
  // 自定义顶点着色器。这是一个从模型空间到模型空间的函数。
  // VertexInput 如下所述。
  vertexShaderText: `
    // 重要提示：函数签名必须使用这些参数名称。这会使运行时更容易生成着色器并进行优化。
    void vertexMain(VertexInput vsInput, inout czm_modelVertexOutput vsOutput) {
        //代码在这里。一个空的代码体不会有任何操作
    }
  `,
  //自定义的片段着色器。下面将会介绍 FragmentInput。不论mode如何，它始终接收一个材质，并直接在原地修改它。
  fragmentShaderText: `
    //重要提示：函数签名必须使用这些参数名称。这使得运行时更容易生成着色器并进行优化。
    void fragmentMain(FragmentInput fsInput, inout czm_modelMaterial material) {
        //代码在这里。例如将漫反射颜色设置为半透明的红色：
        material.diffuse = vec3(1.0, 0.0, 0.0);
        material.alpha = 0.5;
    }
  `,
});
```

## 应用CustomShader

CustomShader可以应用于 3D Tiles 或`Model`如下：

```JS
const customShader = new Cesium.CustomShader(/*...*/);

//应用于 tileset 中的所有 tile。
const tileset = await Cesium.Cesium3DTileset.fromUrl(
  "http://example.com/tileset.json", {
    customShader: customShader
});
viewer.scene.primitives.add(tileset);

//直接应用于模型
const model = await Cesium.Model.fromGltfAsync({,
  url: "http://example.com/model.gltf",
  customShader: customShader
});
```

## Uniforms

CustomShader目前支持以下Uniform类型：

| Uniform类型         | GLSL型       | JS型              |
| :----------- | :---------- | :--------------- |
| `FLOAT`      | `float`     | `Number`         |
| `VEC2`       | `vec2`      | `Cartesian2`     |
| `VEC3`       | `vec3`      | `Cartesian3`     |
| `VEC4`       | `vec4`      | `Cartesian4`     |
| `INT`        | `int`       | `Number`         |
| `INT_VEC2`   | `ivec2`     | `Cartesian2`     |
| `INT_VEC3`   | `ivec3`     | `Cartesian3`     |
| `INT_VEC4`   | `ivec4`     | `Cartesian4`     |
| `BOOL`       | `bool`      | `Boolean`        |
| `BOOL_VEC2`  | `bvec2`     | `Cartesian2`     |
| `BOOL_VEC3`  | `bvec3`     | `Cartesian3`     |
| `BOOL_VEC4`  | `bvec4`     | `Cartesian4`     |
| `MAT2`       | `mat2`      | `Matrix2`        |
| `MAT3`       | `mat3`      | `Matrix3`        |
| `MAT4`       | `mat4`      | `Matrix4`        |
| `SAMPLER_2D` | `sampler2D` | **TextureUniform** |

### Texture Uniforms

纹理uniforms有更多的选项，已经封装在 `TextureUniform`类中了。纹理可以从URL、资源或 `typedArray` 中加载。以下是一些示例：
> 这里的"资源"指的是Cesium支持的一些资源类型，比如Blob、File和BlobURI等。可以通过将这些资源传递给TextureUniform类的构造函数，来创建纹理。
```js
const textureFromUrl = new Cesium.TextureUniform({
  url: "https://example.com/image.png",
});

const textureFromTypedArray = new Cesium.TextureUniform({
  typedArray: new Uint8Array([255, 0, 0, 255]),
  width: 1,
  height: 1,
  pixelFormat: Cesium.PixelFormat.RGBA,
  pixelDatatype: Cesium.PixelDatatype.UNSIGNED_BYTE,
});

//TextureUniform 还提供了用于控制采样器的选项
const textureWithSampler = new Cesium.TextureUniform({
  url: "https://example.com/image.png",
  repeat: false,
  minificationFilter: Cesium.TextureMinificationFilter.NEAREST,
  magnificationFilter: Cesium.TextureMagnificationFilter.NEAREST,
});
```

## Varyings

Varyings 在`CustomShader`构造函数中声明。这会自动将诸如`out float v_userDefinedVarying;`和`in float v_userDefinedVarying;`之类的行分别添加到 GLSL 顶点和片段着色器的顶部。

用户负责在 `vertexShaderText` 中为这个 `varying` 赋值，并在 `fragmentShaderText` 中使用它。例如：
```c
const customShader = new Cesium.CustomShader({
  //在这里声明了变化
  varyings: {
    v_selectedColor: Cesium.VaryingType.VEC4,
  },
  //用户在顶点着色器中分配变量
  vertexShaderText: `
    void vertexMain(VertexInput vsInput, inout czm_modelVertexOutput vsOutput) {
        float positiveX = step(0.0, vsOutput.positionMC.x);
        v_selectedColor = mix(
            vsInput.attributes.color_0,
            vsInput.attributes.color_1,
            vsOutput.positionMC.x
        );
    }
  `,
  //用户在片段着色器中使用 varying
  fragmentShaderText: `
    void fragmentMain(FragmentInput fsInput, inout czm_modelMaterial material) {
        material.diffuse = v_selectedColor.rgb;
    }
  `,
});
```

CustomShader支持以下不同类型：

| Varyings      | GLSL型   |
| :------ | :------ |
| `FLOAT` | `float` |
| `VEC2`  | `vec2`  |
| `VEC3`  | `vec3`  |
| `VEC4`  | `vec4`  |
| `MAT2`  | `mat2`  |
| `MAT3`  | `mat3`  |
| `MAT4`  | `mat4`  |

## CustomShader模式

自定义片段着色器是可配置的，因此它可以在材质或照明之前/之后运行。这是可用模式的摘要。

| 模式                    | 片段着色器管线            | 描述                         |
| :-------------------- | :----------------- | :------------------------- |
| `MODIFY_MATERIAL`（默认） | 材质 -> CustomShader -> 光照 | CustomShader修改材质阶段的结果            |
| `REPLACE_MATERIAL`    | CustomShader -> 光照       | 根本不运行材质阶段，而是在CustomShader中按程序生成它 |

在上面，“材质”对纹理进行了预处理，从而产生了一个`czm_modelMaterial`. 这主要与 PBR 相关，但即使对于 UNLIT，也会处理基色纹理。

## `VertexInput`结构

包含属性的自动生成的 GLSL 结构。
```c
struct VertexInput {
    //处理过的属性。请参阅下面的属性结构部分。
    Attributes attributes;
    //Feature ID/Batch ID。请参阅下面的 FeatureIds 结构部分。
    FeatureIds featureIds;
    //元数据属性。请参阅下面的元数据结构部分。
    Metadata metadata;
    //元数据类属性。请参阅下面的元数据类结构部分。
    MetadataClass metadataClass;
    //元数据统计。请参阅下面的元数据统计结构部分
    MetadataStatistics metadataStatistics;
};
```

## `FragmentInput`结构

这个结构类似于`VertexInput`，但是有一些更多的自动变量用于各种坐标空间中的位置。
```c
struct FragmentInput {
    //处理的属性值。请参阅下面的属性结构部分。
    Attributes attributes;
    //Feature ID/Batch ID。请参阅下面的 FeatureIds 结构部分。
    FeatureIds featureIds;
    //元数据属性。请参阅下面的元数据结构部分。
    Metadata metadata;
    //元数据类属性。请参阅下面的元数据类结构部分。
    MetadataClass metadataClass;
    //元数据统计。请参阅下面的元数据统计结构部分
    MetadataStatistics metadataStatistics;
};
```

## Attributes结构

该`Attributes`结构是根据CustomShader中使用的变量和要渲染的图元中可用的属性动态生成的。

例如，如果用户在着色器中使用，运行时将生成从模型中的`fsInput.attributes.texCoord_0`属性（如果可用）提供此值所需的代码`TEXCOORD_0`

如果图元不具有满足CustomShader所需的属性，则将尽可能推断出默认值，以便着色器仍然可以编译。否则，将为该图元禁用自定义顶点/片段着色器部分。

内置属性的完整列表如下。`0, 1, 2, ...`一些属性有一个集合索引，它是（例如）之一`texCoord_0`，这些用`N`.

| 模型中对应的属性           | 着色器中的变量       | 类型      | 在顶点着色器中可用吗？ | 在片段着色器中可用吗？ | 描述                                                      |
| :----------------- | :------------ | :------ | :---------- | :---------- | :------------------------------------------------------ |
| `POSITION`         | `positionMC`  | `vec3`  | 是的          | 是的          | 在模型坐标中的位置                                               |
| `POSITION`         | `positionWC`  | `vec3`  | 不           | 是的          | 世界坐标中的位置 (WGS84 ECEF `(x, y, z)`)。精度低。                  |
| `POSITION`         | `positionEC`  | `vec3`  | 不           | 是的          | 在眼睛坐标中的位置。                                              |
| `NORMAL`           | `normalMC`    | `vec3`  | 是的          | 不           | 模型坐标中的单位长度法向量。仅在顶点着色器中可用                                |
| `NORMAL`           | `normalEC`    | `vec3`  | 不           | 是的          | 眼睛坐标中的单位长度法向量。仅在片段着色器中可用                                |
| `TANGENT`          | `tangentMC`   | `vec3`  | 是的          | 不           | 模型坐标中的单位长度切向量。这始终是一个`vec3`. 对于提供`w`组件的模型，在计算双切线向量后将其删除。 |
| `TANGENT`          | `tangentEC`   | `vec3`  | 不           | 是的          | 眼睛坐标中的单位长度切向量。这始终是一个`vec3`. 对于提供`w`组件的模型，在计算双切线向量后将其删除。 |
| `NORMAL`&`TANGENT` | `bitangentMC` | `vec3`  | 是的          | 不           | 模型坐标中的单位长度双切线向量。仅当法线和切线向量都可用时才可用。                       |
| `NORMAL`&`TANGENT` | `bitangentEC` | `vec3`  | 不           | 是的          | 眼睛坐标中的单位长度双切线向量。仅当法线和切线向量都可用时才可用。                       |
| `TEXCOORD_N`       | `texCoord_N`  | `vec2`  | 是的          | 是的          | 第`N`组纹理坐标。                                           |
| `COLOR_N`          | `color_N`     | `vec4`  | 是的          | 是的          | 第`N`组顶点颜色。这总是一个`vec4`; 如果模型未指定 alpha 值，则假定为 1。       |
| `JOINTS_N`         | `joints_N`    | `ivec4` | 是的          | 是的          | 第`N`组联合指数                                            |
| `WEIGHTS_N`        | `weights_N`   | `vec4`  |             |             |                                                         |

自定义属性也可用，但它们被重命名为使用小写字母和下划线。例如，`_SURFACE_TEMPERATURE` 模型中调用的属性将成为`fsInput.attributes.surface_temperature`着色器中的属性。

## `FeatureIds`结构

这个结构是动态生成的，以将所有不同的Feature ID 收集到一个集合中，而不管值是来自属性、纹理或varying。
> 为什么要素id会来自属性、纹理或varying？
> 在三维场景渲染中，要素ID可以来源于不同的数据，这是因为不同的要素ID需要用不同的数据来标识，这些数据可以是：
> 	1. 属性(Attribute)：属性是与每个几何体（比如点、线或多边形）相关联的标量或向量值。例如，一个三角形的颜色可能是一个属性值，而且每个三角形的颜色可能不同。
> 	2. 纹理(Texture)：纹理是一个图像，可以应用到几何体表面上，从而赋予其颜色、图案或其他特征。如果在纹理图像中标记每个几何体或几何体的部分，这些标记就可以用作要素ID。
> 	3. varying：varying在顶点着色器和片元着色器之间传递数据，可以用来传递每个顶点的颜色或法向量等信息。可以利用varying来传递要素ID值到片元着色器中进行处理。
> 因此，要素ID可以来自多种数据，可以根据实际需要选择使用哪种方式来标识要素ID。

Feature ID 表示为 GLSL `int`，但在 WebGL 1 中这有几个限制：
*   以上`2^24`, values 可能会丢失精度，因为 WebGL 1 实现 `highp int`为浮点值。
*   理想情况下，类型应该是`uint`，但直到 WebGL 2 才可用

### 3D Tiles 1.0 Batch ID

在 3D Tiles 1.0 中，在图元中识别Feature的相同概念被称为`BATCH_ID`或 legacy `_BATCHID`。这些Batch ID 被重命名为单个Feature ID，索引始终为 0：

*   `vsInput.featureIds.featureId_0`（顶点着色器）
*   `fsInput.featureIds.featureId_0`（片段着色器）

### `EXT_mesh_features`/`EXT_instance_features` Feature IDs

当使用glTF 扩展`EXT_mesh_features`或时，Feature ID 出现在两个地方：`EXT_instance_features`

1.  任何 glTF 基元都可以有一个`featureIds`数组。该`featureIds`数组可能包含Feature ID 属性、隐式Feature ID 属性和/或Feature ID 纹理。无论Feature ID 的类型如何，它们都会出现在CustomShader中，因为数组中Feature ID 的索引在`(vsInput|fsInput).featureIds.featureId_N`哪里。`NfeatureIds`
2.  `EXT_mesh_gpu_instancing`任何带有和 的glTF 节点都`EXT_instance_features`可以定义Feature ID。这些可能是Feature ID 属性或隐式Feature ID 属性，但不是Feature ID 纹理。这些将出现在CustomShader中，因为数组中Feature ID 的索引在`(vsInput|fsInput).featureIds.instanceFeatureId_N`哪里。`NfeatureIds`

此外，Feature ID 纹理仅在片段着色器中受支持。

如果一组要素 ID 包含一个`label`属性（ 中的新增内容`EXT_mesh_features`），则该标签将作为别名使用。例如，如果`label: "alias"`，则将 `(vsInput|fsInput).featureIds.alias`与 一起在着色器中可用`featureId_N`。

例如，假设我们有一个具有以下Feature ID 的 glTF primitive：

```c
"nodes": [
  {
    "mesh": 0
    "extensions": {
      "EXT_mesh_gpu_instancing": {
        "attributes": {
          "TRANSLATION": 3,
          "_FEATURE_ID_0": 4
        }
      },
      "EXT_instance_features": {
        "featureIds": [
          {
            //默认Feature ID（实例 ID）
            //
            //顶点着色器：vsInput.featureIds.instanceFeatureId_0 或 vsInput.featureIds.perInstance
            //片段着色器：fsInput.featureIds.instanceFeatureId_0 或 fsInput.featureIds.perInstance
            "label": "perInstance",
            "propertyTable": 0
          },
          {
            //Feature ID 属性。这对应于上面实例化扩展中的 _FEATURE_ID_0。请注意，这被标记为 instanceFeatureId_1，因为它是 featureIds 数组中设置的第二个Feature ID
            //
            //顶点着色器：vsInput.featureIds.instanceFeatureId_1
            //片段着色器：fsInput.featureIds.instanceFeatureId_1
            //
            //由于没有标签字段，因此必须使用 instanceFeatureId_1。
            "propertyTable": 1,
            "attribute": 0
          },
        ]
      }
    }
  }
],
"meshes": [
  {
    "primitives": [
      {
        "attributes": {
          "POSITION": 0,
          "_FEATURE_ID_0": 1,
          "_FEATURE_ID_1": 2
        },
        "extensions": {
          "EXT_mesh_features": {
            "featureIds": [
              {
                //Feature ID 纹理
                //
                //顶点着色器：（不支持）
                //片段着色器：fsInput.featureIds.featureId_0 或 fsInput.featureIds.texture
                "label": "texture",
                "propertyTable": 2,
                "index": 0,
                "texCoord": 0,
                "channel": 0
              },
              {
                //默认要素 ID（顶点 ID）
                //
                //顶点着色器：vsInput.featureIds.featureId_1 或 vsInput.featureIds.perVertex
                //片段着色器：fsInput.featureIds.featureId_1 或 fsInput.featureIds.perVertex
                "label": "perVertex",
                "propertyTable": 3,
              },
              {
                //Feature ID 属性 (_FEATURE_ID_0)。请注意，它在 featureIds 数组中的索引被标记为 featureId_2
                //
                //顶点着色器：vsInput.featureIds.featureId_2
                //片段着色器：fsInput.featureIds.featureId_2
                //
                //由于没有标签，因此必须使用 featureId_2。
                "propertyTable": 4,
                "attribute": 0
              },
              {
                //Feature ID 属性 (_FEATURE_ID_1)。请注意，它在 featureIds 数组中的索引被标记为 featureId_3
                //
                //顶点着色器：vsInput.featureIds.featureId_3
                //片段着色器：fsInput.featureIds.featureId_3
                "propertyTable": 5,
                "attribute": 1
              }
            ]
          }
        }
      },
    ]
  }
]
```

### 旧`EXT_feature_metadata` Feature IDs

`EXT_feature_metadata`是 . 的早期草稿`EXT_mesh_features`。尽管要素 ID 概念没有太大变化，但 JSON 结构略有不同。在较旧的扩展中，分别`featureIdAttributes`存储`featureIdTextures` 。在这个 CesiumJS 实现中，Feature属性和Feature纹理被连接到一个列表中，本质上是 `featureIds = featureIdAttributes.concat(featureIdTextures)`. 除了扩展 JSON 中的这种差异外，Feature ID 集的标记方式与 相同`EXT_mesh_features`，即

*   `(vsInput|fsInput).featureIds.featureId_N`对应于来自每个基元的`N`组合数组中的第 - 个Feature ID 集。`featureIds`
*   `(vsInput|fsInput).featureIds.instanceFeatureId_N`对应于来自具有扩展名的节点的数组 `N`中的第 - 个Feature ID 集。`featureIdsEXT_mesh_gpu_instancing`

为了进行比较，这里是与上一节中相同的示例，已翻译为`EXT_feature_metadata`扩展名：
```c
"nodes": [
  {
    "mesh": 0,
    "extensions": {
      "EXT_mesh_gpu_instancing": {
        "attributes": {
          "TRANSLATION": 3,
          "_FEATURE_ID_0": 4
        },
        "extensions": {
          "EXT_feature_metadata": {
            "featureIdAttributes": [
              {
                //来自隐式范围的要素 ID 属性
                //
                //顶点着色器：vsInput.featureIds.instanceFeatureId_0
                //片段着色器：fsInput.featureIds.instanceFeatureId_0
                "featureTable": "perInstanceTable",
                "featureIds": {
                  "constant": 0,
                  "divisor": 1
                }
              },
              {
                //Feature ID 属性。这对应于上面实例化扩展中的 _FEATURE_ID_0。请注意，这被标记为 instanceFeatureId_1，因为它是 featureIds 数组中设置的第二个Feature ID
                //
                //顶点着色器：vsInput.featureIds.instanceFeatureId_1
                //片段着色器：fsInput.featureIds.instanceFeatureId_1
                "featureTable": "perInstanceGroupTable",
                "featureIds": {
                  "attribute": "_FEATURE_ID_0"
                }
              }
            ],
          }
        }
      }
    }
  }
],
"meshes": [
  {
    "primitives": [
      {
        "attributes": {
          "POSITION": 0,
          "_FEATURE_ID_0": 1,
          "_FEATURE_ID_1": 2
        },
        "extensions": {
          "EXT_feature_metadata": {
            "featureIdAttributes": [
              {
                //隐式Feature ID 属性
                //
                //顶点着色器：vsInput.featureIds.featureId_0
                //片段着色器：fsInput.featureIds.featureId_0
                "featureTable": "perFaceTable",
                "featureIds": {
                  "constant": 0,
                  "divisor": 3
                }
              },
              {
                //Feature ID 属性 (_FEATURE_ID_0)。请注意，它在 featureIds 数组中的索引被标记为 featureId_1
                //
                //顶点着色器：vsInput.featureIds.featureId_1
                //片段着色器：fsInput.featureIds.featureId_1
                "featureTable": "perFeatureTable",
                "featureIds": {
                  "attribute": "_FEATURE_ID_0"
                }
              },
              {
                //Feature ID 属性 (_FEATURE_ID_1)。请注意，它在 featureIds 数组中的索引被标记为 featureId_2
                //
                //顶点着色器：vsInput.featureIds.featureId_2
                //片段着色器：fsInput.featureIds.featureId_2
                "featureTable": "otherFeatureTable",
                "featureIds": {
                  "attribute": "_FEATURE_ID_1"
                }
              }
            ],
            "featureIdTextures": [
              {
                //Feature ID 纹理。请注意，这被标记为 featureId_3，因为Feature ID 纹理列表连接到Feature ID 属性列表
                //
                //顶点着色器：（不支持）
                //片段着色器：fsInput.featureIds.featureId_3
                "featureTable": "perTexelTable",
                "featureIds": {
                  "texture": {
                    "texCoord": 0,
                    "index": 0
                  },
                  "channels": "r"
                }
              }
            ]
          }
        }
      },
    ]
  }
]
```

## `Metadata`结构

此结构包含模型可从 [EXT_structural_metadata](https://github.com/CesiumGS/glTF/tree/3d-tiles-next/extensions/2.0/Vendor/EXT_structural_metadata) glTF 扩展（或较旧的 [EXT_feature_metadata](https://github.com/CesiumGS/glTF/tree/3d-tiles-next/extensions/2.0/Vendor/EXT_feature_metadata)扩展）访问的相关元数据属性。

目前支持以下类型的元数据：
*   glTF 扩展中的属性`EXT_structural_metadata`。
*   来自 glTF 扩展的属性纹理`EXT_structural_metadata`。当前仅`componentType: UINT8`支持带有的类型。

无论元数据的来源如何，属性都按属性 ID 收集到一个结构中。考虑以下元数据类：

```c
"schema": {
  "classes": {
    "wall": {
      "properties": {
        "temperature": {
          "name": "Surface Temperature",
          "type": "SCALAR",
          "componentType": "FLOAT32"
        }
      }
    }
  }
}
```

这将在着色器中显示为结构字段，如下所示：

```c
struct Metadata {
  float temperature;
}
```

`vsInput.metadata.temperature`现在可以通过或 访问温度`fsInput.metadata.temperature`。

### Normalized归一化值

如果类属性指定`normalized: true`，该属性将作为适当的浮点类型（例如`float`或`vec3`）出现在着色器中。`[0, 1]`所有组件都将在（无符号）或（有符号）的范围内`[-1, 1]` 。

例如，
```c
"schema": {
  "classes": {
    "wall": {
      "properties": {
        //尽管存储为 UINT8，但伤害在 0.0 和 1.0 之间归一化
        //glTF
        "damageAmount": {
          "name": "Wall damage (normalized)",
          "type": "SCALAR",
          "componentType": "UINT32",
          "normalized": true
        }
      }
    }
  }
}
```

这将显示为`float`0.0 到 1.0 之间的值，可通过 `(vsInput|fsInput).metadata.damageAmount`

### 偏移量和比例

如果属性提供了offset偏移量或scale缩放比例，它会在归一化（如果适用）后自动应用。这有助于将值预先缩放到方便的范围。

例如，考虑采用归一化温度值并自动将其转换为摄氏度或华氏度：
```c
"schema": {
  "classes": {
    "wall": {
      "properties": {
        //缩放到范围 [0, 100]，单位为 °C
        "temperatureCelsius": {
          "name": "Temperature (°C)",
          "type": "SCALAR",
          "componentType": "UINT32",
          "normalized": true,
          //offset 默认为 0，scale 默认为 1
          "scale": 100
        },
        //缩放/移动到范围 [32, 212] °F
        "temperatureFahrenheit": {
          "name": "Temperature (°C)",
          "type": "SCALAR",
          "componentType": "UINT32",
          "normalized": true,
          "offset": 32,
          "scale": 180
        }
      }
    }
  }
}
```

在着色器中，`(vsInput|fsInput).metadata.temperatureCelsius`将是一个`float` 介于 0.0 和 100.0 之间的值，而 `(vsInput|fsInput).metadata.temperatureFahrenheit`将是一个`float`范围为`[32.0, 212.0]`.

### Property ID消毒
> 换句话说，就是通过一系列的处理方式将属性ID（通常是字符串）从潜在的安全威胁中清除掉，保护应用程序或系统不受攻击。这样做的目的是防止数据注入攻击，例如SQL注入或跨站脚本攻击等。

GLSL 只支持字母数字标识符，即不以数字开头的标识符。此外，带有连续下划线 ( `__`) 的标识符，以及带有`gl_`前缀的标识符，在 GLSL 中是保留的。为了规避这些限制，Property ID 修改如下：
1.  将所有非字母数字字符序列替换为单个`_`.
2.  删除保留`gl_`前缀（如果存在）。
3.  如果标识符以数字 ( `[0-9]`) 开头，则前缀为`_`

以下是结构中CustomShader中属性 ID 和结果变量名称的几个示例`(vsInput|fsInput).metadata`：
*   `temperature ℃`->`metadata.temperature_`
*   `custom__property`->`metadata.custom_property`
*   `gl_customProperty`->`metadata.customProperty`
*   `12345`->`metadata._12345`
*   `gl_12345`->`metadata._12345`

如果以上结果导致空字符串或与其他属性 ID 的名称冲突，则行为未定义。例如：
*   `✖️✖️✖️`映射到空字符串，因此行为未定义。
*   `temperature ℃`具有名称和的两个属性`temperature ℉`都将映射到`metadata.temperature`，因此行为未定义

使用点云 ( `.pnts`) 格式时，每个点的属性被转码为属性属性。这些属性 ID 遵循相同的约定。

## `MetadataClass`结构

此结构包含每个元数据属性的常量，如class模式中所定义。

无论元数据的来源如何，属性都按属性 ID 收集到一个结构中。考虑以下元数据类：
```c
"schema": {
  "classes": {
    "wall": {
      "properties": {
        "temperature": {
          "name": "Surface Temperature",
          "type": "SCALAR",
          "componentType": "FLOAT32",
          "noData": -9999.0,
          "default": 72.0,
          "min": -40.0,
          "max": 500.0,
        }
      }
    }
  }
}
```

这将显示在结构字段的着色器中，如下所示：
```c
struct floatMetadataClass {
  float noData;
  float defaultValue;//'default' 是 GLSL 中的保留字
  float minValue;//'min' 是 GLSL 中的保留字
  float maxValue;//'max' 是 GLSL 中的保留字
}
struct MetadataClass {
  floatMetadataClass temperature;
}
```

将选择每个属性的子结构，以便各个属性（例如`noData`和`defaultValue`）与属性的实际值具有相同的类型。

现在可以在顶点着色器中按如下方式访问 noData 和默认值：
```c
float noData = vsInput.metadataClass.temperature.noData;//== -9999.0
float defaultTemp = vsInput.metadataClass.temperature.defaultValue;//== 72.0
float minTemp = vsInput.metadataClass.temperature.minValue;//== -40.0
float maxTemp = vsInput.metadataClass.temperature.maxValue;//== 500.0
```

或类似地来自`fsInput`片段着色器中的结构。

## `MetadataStatistics`结构

如果该模型是从[3D Tiles tileset加载的，它可能在](https://github.com/CesiumGS/3d-tiles/tree/main/specification)tileset.json的statistics属性中定义了统计信息。这些信息可以从`CustomShader`的`MetadataStatistics`结构中获得。

### 组织

无论元数据的来源如何，属性都通过属性 ID 收集到一个源中。考虑以下元数据类：
```c
  "statistics": {
    "classes": {
      "exampleMetadataClass": {
        "count": 29338,
        "properties": {
          "intensity": {
            "min": 0.0,
            "max": 0.6333333849906921,
            "mean": 0.28973701532415364,
            "median": 0.25416669249534607,
            "standardDeviation": 0.18222664489583626,
            "variance": 0.03320655011,
            "sum": 8500.30455558002,
          },
          "classification": {
            "occurrences": {
              "MediumVegetation": 6876,
              "Buildings": 22462
            }
          }
        }
      }
    }
  }
```

这将显示在结构字段的着色器中，如下所示：
```c
struct floatMetadataStatistics {
  float minValue;//'min' 是 GLSL 中的保留字
  float maxValue;//'max' 是 GLSL 中的保留字
  float mean;
  float median;
  float standardDeviation;
  float variance;
  float sum;
}
struct MetadataStatistics {
  floatMetadataStatistics intensity;
}
```

可以从顶点着色器中访问统计值，如下所示：
```c
float minValue = vsInput.metadataStatistics.intensity.minValue;
float mean = vsInput.metadataStatistics.intensity.mean;
```

或类似地来自`fsInput`片段着色器中的结构。

### 类型

对于`SCALAR`、`VECN`和`MATN`type 属性，统计结构字段`minValue`、`maxValue`、`median`和`sum`将使用与它们描述的元数据属性相同的类型声明。字段`mean`、`standardDeviation`和`variance`声明为与元数据属性具有相同维度的类型，但具有浮点组件。

对于`ENUM`类型元数据，该属性的统计结构应该包含一个`occurrence`字段，但该字段尚未实现。

## `czm_modelVertexOutput`结构

此结构是内置的，请参阅[文档注释](https://github.com/CesiumGS/cesium/blob/main/packages/engine/Source/Shaders/Builtin/Structs/modelVertexOutput.glsl)。
```js
/**
 * 表示自定义顶点着色器输出的结构。
 * @property {vec3} positionMC 顶点在模型坐标中的位置
 * @property {float} pointSize gl_PointSize 的自定义值。这仅用于点图元。
 */
struct czm_modelVertexOutput {
  vec3 positionMC;
  float pointSize;
};
```


该结构包含自定义顶点着色器的输出。这包括：
*   `positionMC` - 模型空间坐标中的顶点位置。该结构字段可用于扰动或动画顶点。它被初始化为 `vsInput.attributes.positionMC`.  CustomShader可以修改它，结果用于计算`gl_Position`。
* - `pointSize` - 对应于`gl_PointSize`。这仅适用于呈现为`gl.POINTS` 的模型，否则将被忽略。这将覆盖由`Cesium3DTileStyle`应用于模型的任何点大小样式。

> **实施注意事项**：`positionMC`不修改图元的边界球体。如果顶点移动到包围球之外，则图元可能会被无意中剔除，具体取决于视锥体。

## `czm_modelMaterial`结构

此结构是内置的，请参阅[文档注释](https://github.com/CesiumGS/cesium/blob/main/packages/engine/Source/Shaders/Builtin/Structs/modelMaterial.glsl)。这与旧的 Fabric 系统类似，但由于支持 PBR 照明，因此字段略有不同。`czm_material`

```js
/**
 * 这是一个用于表示{@link Model}材质的结构体。模型渲染管道将在材质、自定义着色器和光照阶段之间传递此结构体。
 * 这不应与{@link czm_material}混淆，后者是旧版Fabric材料系统使用的，尽管它们非常相似。
 * <p>
 * 所有颜色值（漫反射、镜面反射、发射）都在线性颜色空间中。
 * </p>
 * @property {vec3} diffuse 入射光向各个方向均匀散射。
 * @property {float} alpha 此材质的 Alpha。0.0 是完全透明的；1.0 是完全不透明的。
 * @property {vec3} specular PBR 材质中法向入射反射光的镜面反射颜色。这有时在文献中称为 f0。
 * @property {float} roughness 一个从 0.0 到 1.0 的数字，表示表面的粗糙程度。接近 0.0 的值会产生光滑的表面，接近 1.0反之
 * @property {vec3} normalEC 表面在眼睛坐标中的法线。它用于法线贴图等效果。默认值为表面未修改的法线。
 * @property {float} occlusion 在材质上此时收到的环境遮挡。1.0 表示完全点亮，0.0 表示完全遮挡。
 * @property {vec3} emissive 材质向各个方向均匀发射的光。默认值为 vec3(0.0)，不发光。
 */
struct czm_modelMaterial {
    vec3 diffuse;
    float alpha;
    vec3 specular;//非常谈的的颜色
    float roughness;
    vec3 normalEC;
    float occlusion;
    vec3 emissive;//更接近设置色的谈的颜色
};
```

此结构用作片段着色器管道阶段的基本输入/输出。例如：
*   材质阶段产生材质
*   照明阶段接收材质，计算照明，并将结果存储到`material.diffuse`
*   CustomShader（无论它在管线中的哪个位置）接收一种材质（即使它是具有默认值的材质）并对其进行修改。

### 材质色彩空间

1. 材质颜色（例如`material.diffuse`）始终在线性颜色空间中，即使`lightingModel`是`LightingModel.UNLIT`。
2. 当`scene.highDynamicRange`是 时`false`，最终计算的颜色（在CustomShader和光照之后）被转换为`sRGB`.
