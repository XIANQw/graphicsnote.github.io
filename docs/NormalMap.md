# 法线贴图
法线贴图表示的是切线空间下的法向量, 任何朝向的平面都可以适用同一套法线贴图.

## 使用
### 顶点数据
顶点数据除了`pos`, `texcoord`, `normal`外还需`Tangent`或`Bitangent`.
### VS
vs中建立TBN矩阵, 用于将切线空间的法线转到世界空间便于后续光照计算
```
// 将切线从模型空间转到世界空间
half4 tangentWorld = half4(UnityObjectToWorldDir(v.tangent.xyz), v.tangent.w);
half3x3 tangentToWorld = CreateTangentToWorldPerVertex(normalWorld, tangentWorld.xyz, tangentWorld.w);

// Unity构建TBN矩阵的函数
half3x3 CreateTangentToWorldPerVertex(half3 normal, half3 tangent, half tangentSign)
{
	// For odd-negative scale transforms we need to flip the sign
	half sign = tangentSign * unity_WorldTransformParams.w;
	half3 binormal = cross(normal, tangent) * sign;
	return half3x3(tangent, binormal, normal);
}
```


### Fragment

```
half3 normalTangent = UnpackScaleNormal(tex2D (_BumpMap, texcoords.xy), _BumpScale);
```
`UnpackNormal`采样出来的切线空间法向量, 将`[0, 1]`空间转换回`[-1, 1]`, `DXT5nm`压缩的法线贴图只会存储`xy`值所以需要算出`z`
```
// Unpack normal as DXT5nm (1, y, 1, x) or BC5 (x, y, 0, 1)
// Note neutral texture like "bump" is (0, 0, 1, 1) to work with both plain RGB normal and DXT5nm/BC5
fixed3 UnpackNormalmapRGorAG(fixed4 packednormal)
{
    // This do the trick
   packednormal.x *= packednormal.w;

    fixed3 normal;
    normal.xy = packednormal.xy * 2 - 1;
    normal.z = sqrt(1 - saturate(dot(normal.xy, normal.xy)));
    return normal;
}
inline fixed3 UnpackNormal(fixed4 packednormal)
{
#if defined(UNITY_NO_DXT5nm)
    return packednormal.xyz * 2 - 1;
#else
    return UnpackNormalmapRGorAG(packednormal);
#endif
}
```
最后将`normalTangent`转换到世界空间坐标再归一化就可以使用了
```
half3 normalWorld = normalize(mul(normalTangent, tangentToWorld));
```


## 补充
`odd-negative scaling` : 奇数轴负缩放, xyz中有1或3个轴的缩放值为负. 具有`odd-negative scaling`和没有的物体渲染状态不同, 因此也不能动态合批\
在Unity中`vertex.tangent.w` = 1 或 -1
之所以加了这个值, 是因为在`OpenGL`和`DirectX`平台上UV走向不一样.

`OpenGL`平台，U水平向右，V垂直向上即+y\
`DirectX`平台，U水平向右，V垂直向下即-y\
而Unity遵循的是`OpenGL`规范，在导入模型的时候uv走向是+y，所以如果在windows上Unity用`DirectX`的情况下，叉乘得到的`B'`刚好反向了，手性错误。所以Unity存储了一个手性信息在`tangent.w`里，在正交化最后得到切线`T'`的时候计算当前平台的手性值并存在`tangent.w`中。

所以副切线的计算时才会 `*sign`