---
title: "GeoJSON 이란?"
categories:
  - etc
tags:
  - GeoJSON
date: "2021-11-18 22:42:14"
last_modified_at: "2021-11-18 22:42:14 +0800"
toc: true
toc_sticky: true
toc_label: Table of Contents
---

# | GeoJSON 이란?

> **GeoJSON**은 위치정보를 갖는 점을 기반으로 체계적으로 지형을 표현하기 위해 설계된 개방형 공개 표준 형식이다.

[위키백과 링크](https://ko.wikipedia.org/wiki/GeoJSON)

geometry data 들로 이루어진 object를 GeoJSON 형태로 return 해주는 library.

💡 Geojson에 대한 정의는 [해당 링크](https://tools.ietf.org/html/rfc7946)에서 확인할 수 있습니다.
{: .notice--info}

Geojson이란 쉽게 말해서 지형 정보를 json 데이터 형식을 변형해서 표현한 표준 형식이라고 보면 됩니다.

geojson으로 나타낼 수 있는 정보들은 점(Point), 선(LineString), 면(Polygon), 그리고 이들이 여러 개 묶인 MultiPoint, MultiLineString, MultiPolygon 또한 존재합니다.

geojson은 일반적인 구글맵이나 오픈소스맵들과 같이 `[위도, 경도]` 표기법을 사용하지 않고, [WGS84](https://ko.wikipedia.org/wiki/%EC%84%B8%EA%B3%84_%EC%A7%80%EA%B5%AC_%EC%A2%8C%ED%91%9C_%EC%8B%9C%EC%8A%A4%ED%85%9C)가 권장하는 `[경도, 위도]` 표기법을 지원합니다.
<br>

**GeoJSOn은 다음과 같은 장점을 가집니다.**

1. XML과 비교하여 스키마나 태그 규칙에 대해 훨씬 자유롭습니다.
2. 데이터 용량이 다른 포멧에 비해 상대적으로 작습니다.
3. JSON 형식으므로 쉽게 객채화가 가능하며, 특히 Javascript에서 유용합니다.
4. 다양한 응용프로그램에 적재되기에 용이합니다.

하나하나 씩 살펴 보도록 합시다.

모든 자료(이미지 및 예제)는 [위키피디아 출처](https://ko.wikipedia.org/wiki/GeoJSON)입니다.

## Example

```jsx
var data = [
    { name: 'Location A', category: 'Store', street: 'Market', lat: 39.984, lng: -75.343 },
    { name: 'Location B', category: 'House', street: 'Broad', lat: 39.284, lng: -75.833 },
    { name: 'Location C', category: 'Office', street: 'South', lat: 39.123, lng: -74.534 }
];

GeoJSON.parse(data, {Point: ['lat', 'lng']});

{
    "type": "FeatureCollection",
    "features": [
        { "type": "Feature",
            "geometry": {"type": "Point", "coordinates": [-75.343, 39.984]},
            "properties": {
                "name": "Location A",
                "category": "Store"
            }
        },
        { "type": "Feature",
            "geometry": {"type": "Point", "coordinates": [-75.833, 39.284]},
            "properties": {
                "name": "Location B",
                "category": "House"
            }
        },
        { "type": "Feature",
            "geometry": {"type": "Point", "coordinates": [ -75.534, 39.123]},
            "properties": {
                "name": "Location C",
                "category": "Office"
            }
        }
    ]
}
```

array형태와 single obejct 모두 사용 가능합니다.

## 사용 가능한 Geometry type

단순 점좌표 뿐만 아니라 여러 geometry type을 받을 수 있습니다.

1. Point
2. LineString
3. Polygon
4. MultiPoint, MultiLineString, MultiPolygon

# Point

지도 위에서 **점**의 정보를 나타냅니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a553a01a-b4a6-4a0f-b82f-76f71a785e59/Untitled.png)

```json
{
  "type": "Point",
  "coordinates": [30, 10]
}
```

type은 도형의 형태를, coordinates는 위치값을 나타냅니다. coordinates는 [x,y] 로 표시합니다.

이제부터 [x,y] 형태를 **Position**이라고 부르겠습니다.

# LineString

지도 위에서 **선**의 정보를 나타냅니다.

|---|--|
|![geojson_1](/assets/images/posts/geojosn_1.png)|
`json { "type": "LineString","coordinates": [[30, 10], [10, 30], [40, 40]]}` |

<!-- <div class=pull-right>
![geojson_1](/assets/images/posts/geojosn_1.png)
</div> -->
<img style="float:left;" src='{{ "/assets/images/posts/geojson_1.png" | relative_url }}' alt='relative'>
zzz

coordinates를 보시면 아시겠지만 Point는 1차원이였지만 LineString은 2차원으로 한단계 높아진 것을 알 수 있습니다.

coordinates의 첫번째 항목은 LineString이 시작하는 점의 좌표를 나타낸 것 입니다.

# Polygon

지도 위에서 **면**의 정보를 나타냅니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f13804c0-2e59-443e-8671-d41fd3b2e03b/Untitled.png)

```json
{
  "type": "Polygon",
  "coordinates": [
    [
      [30, 10],
      [40, 40],
      [20, 40],
      [10, 20],
      [30, 10]
    ] //coordinates[0]
  ]
}
```

Polygon은 coordinates가 3차원이라는 점이 특징입니다. 또한 coordinates[0]의 **첫 번째 Position**과 맨 **마지막 Position** 값이 **동일**한 점을 알 수 있습니다. 시작점과 마지막 점이 만나야지만 Polygon이 된다고 생각하시면 됩니다.

다음은 Polygon 안에 또 다른 Polygon이 있어 면 안쪽에 **구멍이 난 것을 표현**한 것입니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/eb668a43-aeda-493d-8b04-420c095476e7/Untitled.png)

```json
{
  "type": "Polygon",
  "coordinates": [
    [
      [35, 10],
      [45, 45],
      [15, 40],
      [10, 20],
      [35, 10]
    ], //coordinates[0]
    [
      [20, 30],
      [35, 35],
      [30, 20],
      [20, 30]
    ] //coordinates[1]
  ]
}
```

위의 Polygon의 coordinates를 보시면 coordinates[1] 이 추가된 것을 볼 수 있습니다. coordinates[1]은 coordinates[0]의 안쪽에 있는 Polygon의 위치 정보를 나타낸 것입니다.

즉, **구멍이 x개**라면 coordinates **배열의 갯수는 x+1**이 됩니다.

만약 안쪽 Polygon이 가장 바깥쪽 Polygon 즉, coordinates[0]을 벗어나게 되거나, 안쪽 Polygon끼리 겹치게 된다면 올바른 Polygon 형태가 아닙니다.

# Multi Point

지도 위에서 **여러 개의 점**의 정보를 나타냅니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/89f2c736-09b3-40c6-93ec-a17e11833b84/Untitled.png)

```json
{
  "type": "MultiPoint",
  "coordinates": [
    [10, 40],
    [40, 30],
    [20, 20],
    [30, 10]
  ]
}
```

눈치 채셨겠지만 **coordinates가 한 차원 높아졌음**을 알 수 있습니다.

Multi가 붙은 것들은 기존의 도형보다 한 차원 높아지면 됩니다.

# Multi LineString

지도 위에서 **여러 개의 선**의 정보를 나타냅니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/864d8bd7-5bab-453d-a254-03d69c5aa3ab/Untitled.png)

```json
{
  "type": "MultiLineString",
  "coordinates": [
    [
      [10, 10],
      [20, 20],
      [10, 40]
    ],
    [
      [40, 40],
      [30, 30],
      [40, 20],
      [30, 10]
    ]
  ]
}
```

# Multi Polygon

지도 위에서 **여러 개의 면**의 정보를 나타냅니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/879f74bc-394e-4db5-b507-df5863ff29b2/Untitled.png)

```json
{
  "type": "MultiPolygon",
  "coordinates": [
    [
      [
        [30, 20],
        [45, 40],
        [10, 40],
        [30, 20]
      ]
    ],
    [
      [
        [15, 5],
        [40, 10],
        [10, 20],
        [5, 10],
        [15, 5]
      ]
    ]
  ]
}
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6c1b8937-0614-4ab4-a0b0-015280a7ffff/Untitled.png)

```json
{
  "type": "MultiPolygon",
  "coordinates": [
    [
      [
        [40, 40],
        [20, 45],
        [45, 30],
        [40, 40]
      ]
    ],
    [
      [
        [20, 35],
        [10, 30],
        [10, 10],
        [30, 5],
        [45, 20],
        [20, 35]
      ],
      [
        [30, 20],
        [20, 15],
        [20, 25],
        [30, 20]
      ]
    ]
  ]
}
```

# Feature

Feature 객체는 공간적으로 경계된 사물을 나타낸 것입니다. 쉽게 말해서 위에서 살펴본 geojson 타입을 개별적으로 나타낸 정보입니다.

Feature의 형식은 다음과 같습니다.

```json
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [102.0, 0.5]
  },
  "properties": {
    "prop0": "value0"
  }
}
```

- **type**: Feature 객체라는 것을 명시합니다.
- **geometry**: 위에서 본 geojson 객체를 나타냅니다.
- **properties**: 해당 feature의 속성을 나타냅니다.

feature를 사용하게 되면 geometry에 속성(properties)을 부여할 수 있게 됩니다.

propertie에는 속성정보가 Key-value 형태로 저장됩니다.

properties를 활용하는 방법은 예를 들어 geometry의 배경색, 선 색상, 선 두께 등을 나타낼 수 있습니다.

```json
{
    "type": "Feature",
    "geometry": {
        "type": "Polygon",
        "coordinates": [
	        [[35, 10], [45, 45], [15, 40], [10, 20], [35, 10]],
	        [[20, 30], [35, 35], [30, 20], [20, 30]]
				]
    },
    "properties": {
			"fill": "000000"
      "stroke": "#000000",
			"stroke-width": "1.5",
    }
}
```

# FeatureCollection

FeatureCollection은 feature들의 모음입니다. 이 객체를 사용해서 속성을 가진 geojson들의 묶음을 나타낼 수 있습니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5f5ec844-b661-4c83-9255-b50ab1bc4971/Untitled.png)

```json
{
    "type": "FeatureCollection",
    "features": [
        {
            "type": "Feature",
            "properties": {
                "stroke": "#000000",
                "stroke-width": "1.5",
                "stroke-opacity": "1",
                "fill": "#000000",
                "fill-opacity": "0.2"
            },
            "geometry": {
                "type": "MultiPolygon",
                "coordinates": [
                    [
                        [[127.075463411,37.255838587],[127.071086046,37.251192948],
											  , // ...
                ]
            }
        }
		]
}
```

위의 정보는 FeatureCollection을 하나의 예시를 든 것입니다.

위의 지도에 나타나는 도형을 FeatureCollection 형태로 표현할 수 있습니다.

## 파싱 방법

각 객체마다 **type 프로퍼티**는 무조건 존재합니다. 따라서 type 프로퍼티가 어떤 값을 가지고 있는 지 먼저 확인해야 합니다.

type은 **"Point" | "MultiPoint" | "LineString" | "MultiLineString" | "Polygon" | "MultiPolygon" | "Feature" | "Feature"** 가 될 수 있습니다.

그리고 type마다 가지고 있는 형태가 있습니다.

먼저 type이 **"Point" | "MultiPoint" | "LineString" | "MultiLineString" | "Polygon" | "MultiPolygon"** 라면 해당 객체는 **coordinates 배열 프로퍼티**를 가지고 있고 각 type에 따라 coordinates의 차원이 다릅니다.

그리고 type이 **"Feature"** 이라면 해당 객체는 **geometry와 properties 프로퍼티**를 가지고 있습니다.

마지막으로 type이 **"FeatureCollection"** 이라면 **features 배열 프로퍼티**를 가지고 있습니다.

이를 이용하면 쉽게 geojson 데이터를 파싱할 수 있습니다.

# | Default data set

비슷한 여러 데이터들을 통합해 파싱하도록 설정이 가능하다.

```jsx
var data1 = [{ name: 'Location A', street: 'Market', x: 34, y: -75 }];

var data2 = [{ name: 'Location B', date: '11/23/2012', x: 54, y: -98 }];

GeoJSON.defaults = {Point: ['x', 'y'], include: ['name']};

GeoJSON.parse(data1, {});

...
```
