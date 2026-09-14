## 4 Texture Mapping
컴퓨터 그래픽스에서 _텍스처 매핑_ 은 해당 씬의 오브젝트에 머티리얼 효과를 적용하는 과정입니다. "텍스처 매핑" 용어에서 "텍스처"는 적용하는 효과를 의미하고, "매핑"은 그 효과를 한 공간에서 다른 공간으로 옮기는 수학적인 대응을 의미입니다. 그리고 이 효과는 어떤 머티리얼 속성이든지 가능합니다. 예를 들어 색상(color), 광택도(shininess), bump geometry(_Bump Mapping_ 이라고 부릅니다), 심지어 머티리얼의 존재 여부(표면의 일부를 잘라낸 것처럼 표현하기 위해서 사용합니다)까지도 텍스처로 사용할 수 있습니다. 

가장 일반적인 텍스처 매핑 방식은 이미지를 오브젝트의 표면에 매핑하여, 오브젝트 표면의 각 지점에서 색상을 정의하는 것입니다. 하지만 실제로는 이 과정을 거꾸로 구현합니다. 오브젝트 표면의 특정 지점이 주어지면, 텍스처 맵에서 그 점에 해당하는 색상을 찾습니다.

먼저, 텍스처 색상을 코드로 생성하도록 구현한 다음, 상수 색상의 텍스처 맵을 만들겠습니다. 대부분의 프로그램에서는 상수 RGB 색상과 텍스처를 서로 다른 클래스에 구현하므로, 한 클래스 안에 둘 다 구현하는 구조를 따르지 않아도 됩니다. 하지만 저는 이 구조를 매우 선호합니다. 이렇게 하면 어떤 색상이든 텍스처로 만들 수 있기 때문입니다.

텍스처에서 색상 값을 조회하기 위해서는 _텍스처 좌표_ 가 필요합니다. 텍스처 좌표는 다양한 방식으로 정의할 수 있으며, 계속 진행하면서 이 아이디어를 발전시켜 나가겠습니다. 지금은 2차원 텍스처 좌표를 입력으로 전달하겠습니다. 텍스처 좌표는 관습적으로 $u$, $v$ 로 네이밍합니다. 상수 색상 텍스처는 모든 $(u, v)$ 텍스처 좌표에서 동일한 상수 색상 값을 가지므로, 사실 텍스처 좌표는 전혀 상관이 없습니다. 그러나 다른 유형의 텍스처에서는 텍스처 좌표가 필요하기 때문에 `color value(...)` 의 메소드 인터페이스에 텍스처 좌표를 가지고 있습니다.

texture 클래스의 핵심 메소드는 매개변수로 입력한 텍스처 좌표를 통해 텍스처 색상을 리턴하는 `color value(...)` 메소드입니다. 텍스처 좌표 $u$, $v$ 뿐만 아니라 점의 3차원 좌표 또한 매개변수로 받습니다. 이러한 이유는 더 진행하면서 명확해질 것입니다.

---

### 4.1 Constant Color Texture

```cpp
#ifndef TEXTURE_H
#define TEXTURE_H

class texture {
  public:
    virtual ~texture() = default;
    
    virtual color value(double u, double v, const point3& p) const = 0;
};

class solid_color : public texture {
  public:
    solid_color(const color& albedo) : albedo(albedo) {}

    solid_color(double red, double green, double blue) : solid_color(color(red, green, blue)) {}

    color value(double u, double v, const point3& p) const override {
      return albedo;
    }

  private:
    color albedo;
};

#endif
```

**<p align="center">Listing 22:** [<span>texture</span>.h] _A texture class</p>_

광선-오브젝트 교차점에 대응하는 $u, v$ 텍스처 좌표를 저장하기 위해서 `hit_record` 클래스를 수정해야 합니다.

```cpp
class hit_record {
  public:
    vec3 p;
    vec3 normal;
    shared_ptr<material> mat;
    double t;
///////////////////////// 추가 ////////////////////////////
    double u;                                           //
    double v;                                           //
//////////////////////////////////////////////////////////
    bool front_face;

    ...
```

**<p align="center">Listing 23:** [<span>hittable</span>.h] _Adding $u$, $v$ coordinates to the `hit_record`</p>_

나중에는 각 `hittable` 유형마다 주어진 점에 대한 $(u, v)$ 텍스처 좌표를 계산해야 합니다. 자세한 내용은 뒤에서 다루겠습니다.

---

### 4.2 Solid Textures: A Checker Texture

---

### 4.3 Rendering The Solid Checker Texture

---

### 4.4 Texture Coordinates for Spheres

---

### 4.5 Accessing Texture Image Data

---

### 4.6 Rendering The Image Texture

---

## 출처

[Ray Tracing: The Next Week - 4 Texture Mapping](https://raytracing.github.io/books/RayTracingTheNextWeek.html#texturemapping)