# Introducción
Proyecto de creación de un sheader para representar de forma estilizada el render de un personaje 3D, desarrollado en HLSL.

Teniedo integrado los siguientes puntos:
- Un modelo de luz custom basado en Lambert (usa el sistema Wrap) y Phong.
- Debe soportar mapa de textura principal y mapa de normales.
- Debes programar el shader de superficie para emitir luz mediante un efecto de horizonte (rim).
- Debes poder controlar desde el material la intensidad del mapa de normal.
- Debes tener un color principal (Albedo) para dar color a toda la iluminación.
- Debes tener color (Phong color) para pintar el brillo del phong.
- Utiliza la técnica de "Banded" para modificar tu modelo de luz y crear una Ramp Texture de forma procedural.

## Desarrollo
Codigo:
```HLSL
Shader "Custom/SDUltimate"
{
    Properties
    {
        _Albedo("Albedo Color", Color) = (1, 1, 1, 1)
        _MainTex("Main Texture", 2D) = "white" {}
        _BumpMap("Normal Map", 2D) = "bump" {}
        _SpecularColor("Specular Color", Color) = (1, 1, 1, 1)
        _SpecularPower("Specular Power", Range(1.0, 10.0)) = 5.0
        _SpecularGross("Specular Gloss", Range(1.0, 5.0)) = 1.0
        _GlossSteps("Gloss Steps", Range(1.0, 8.0)) = 4.0 
        [HDR] _RimColor("Rim Color", Color) = (1, 1, 1, 1)
        _RimPower("Rim Power", Range(0.0, 8.0)) = 1.0
        _NormalStrength("Normal Strength", Range(-5.0, 5.0)) = 1.0
        _NormalTex("Normal Texture", 2D) = "bump" {}
        _Steps("Banded Steps", Range(1.0, 100.0)) = 20.0
    }

    Subshader
    {
        Tags
        {
            "Queque" = "Geometry"
            "RenderType" = "Opaque"
        }

        CGPROGRAM

            #pragma surface surf PhongCustom

            half4 _Albedo;
            half4 _SpecularColor;
            half _SpecularPower;

            half _SpecularGross;
            int _GlossSteps;
            half4 _RimColor;
            float _RimPower;
            float _NormalStrength;
            fixed _Steps;
            
            sampler2D _NormalTex;
            sampler2D _MainTex;
            sampler2D _BumpMap;

            //#pragma surface surf PhongCustom

            half4 LightingPhongCustom(SurfaceOutput s, half3 lightDir, half3 viewDir, half atten)
            {
                lightDir = normalize(lightDir);
                viewDir = normalize(viewDir);

                half NdotL = max(0, dot(s.Normal, lightDir));
                half lightBandsMultiplier = _Steps / 256;
                half lightBnadsAdditive = _Steps / 2;
                fixed BandedLightModel = (floor((NdotL * 256 + lightBnadsAdditive) / _Steps)) * lightBandsMultiplier;
                half3 reflectedLight = reflect(-lightDir, s.Normal);
                half RdotV = max(0, dot(reflectedLight, viewDir));
                half diff = NdotL * 0.5 + 0.5;
                half3 specularity = pow(RdotV, _SpecularGross / _GlossSteps) * _SpecularPower * _SpecularColor.rgb;
                half4 c;
                c.rgb = (NdotL * s.Albedo + specularity) * _LightColor0.rgb * BandedLightModel * (atten * diff * 2);
                c.a = s.Alpha;
                return c;
            }

            struct Input
            {
                float3 viewDir;
                float a;
                float2 uv_MainTex;
                float2 uv_BumpMap;
                float2 uv_NormalTex;
            };

            void surf(Input IN, inout SurfaceOutput o)
            {
                half4 texColor = tex2D(_MainTex, IN.uv_MainTex);
                half4 bumpColor = tex2D(_BumpMap, IN.uv_BumpMap);
                half4 normalColor = tex2D(_NormalTex, IN.uv_NormalTex);
                half3 normal = UnpackNormal(normalColor);

                o.Albedo = texColor.rgb * _Albedo;
                o.Normal = UnpackNormal(bumpColor);
                o.Normal = normalize(normal);

                float3 nVwd = normalize(IN.viewDir);
                float3 NdotV = dot(nVwd, o.Normal);
                half rim = 1 - saturate(NdotV);
                o.Emission = _RimColor.rgb * pow(rim, _RimPower);
                normal.z = normal.z / _NormalStrength;

            }

        ENDCG
    }    
}
```
