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
solid texture(또는 spatial texture)는 3D 공간에서 오로지 각 점의 위치만으로 텍스처 값을 계산합니다. solid texture는 3D 공간 안의 주어진 특정 오브젝트에 색을 칠하는 것이 아니라, 3D 공간 자체의 모든 점에 색을 칠하는 것이라고 생각할 수 있습니다. 이러한 이유로, solid texture가 3D 공간에 정의해 놓은 색상 영역들이 공간에 고정되어 있고, 오브젝트의 위치가 이동한다면 오브젝트 표면의 색이 달라질 수 있습니다. 하지만 일반적으로는 오브젝트의 위치가 이동하면 텍스처도 같이 이동하도록 오브젝트와 solid texture를 서로 고정하고 싶을 것입니다.

spatial texture에 대해 알아보기 위해, 3차원 체크 패턴을 그리는 `checker_texture` 클래스를 구현하겠습니다. spatial texture 함수는 3차원 공간의 지정된 위치을 기준으로 동작하기 때문에, `value()` 함수는 오직 `p` 파라미터만 사용하고 `u`, `v` 파라미터를 사용하지 않습니다.

체크 패턴을 구현하기 위해서는 먼저 입력점의 각 성분에 대해 내림을 계산합니다. 좌표값의 소수 부분을 그냥 제거하는 방법도 있지만, 그렇게 하면 양수/음수값이 0쪽을 향해 모이므로 0을 기준으로 양쪽에서 같은 색이 나오고, 체크 패턴이 자연스럽지 않게 됩니다. `floor` 함수는 항상 값을 음의 무한대 방향으로 이동시켰을 때 처음 만나는 정수로 변환합니다. 세 정수값 $\lfloor x \rfloor, \lfloor y \rfloor, \lfloor z \rfloor$ 을 구한 뒤, 세 값을 모두 더하고 2로 나눈 나머지를 계산합니다. 그 결과값은 0 또는 1이 됩니다. 0은 짝수 색상으로 대응되고, 1은 홀수 색상으로 대응됩니다.

마지막으로, 씬에서 체크 패턴의 크기를 조절하기 위한 scale 파라미터를 추가합니다.

```cpp
class checker_texture : public texture {
  public:
    checker_texture(double scale, shared_ptr<texture> even, shared_ptr<texture> odd)
      : inv_scale(1.0 / scale), even(even), odd(odd) {}

    checker_texture(double scale, const color& c1, const color& c2)
      : checker_texture(scale, make_shared<solid_color>(c1), make_shared<solid_color>(c2)) {}

    color value(double u, double v, const point3& p) const override {
      auto xInteger = int(std::floor(inv_scale * p.x()));
      auto yInteger = int(std::floor(inv_scale * p.y()));
      auto zInteger = int(std::floor(inv_scale * p.z()));

      bool isEven = (xInteger + yInteger + zInteger) % 2 == 0;

      return isEven ? even->value(u, v, p) : odd->value(u, v, p);
    }

  private:
    double inv_scale;
    shared_ptr<texture> even;
    shared_ptr<texture> odd;
};
```

**<p align="center">Listing 24:** [texture.h] _Checked texture_

`checker_texture` 의 odd/even 파라미터는 단순한 색상뿐 아니라 다른 procedural texture(`checker_texture` 와 같은 코드나 수학 함수로 계산하여 생성한 텍스처) 객체 자체를 참조할 수 있습니다. 이런 구조는 Pat Hanrahan이 1980년대에 소개한 shader network의 설계 철학과 같습니다.

procedural texture를 적용하기 위해, `lambertian` 클래스에서 색상 대신 텍스처를 사용하도록 수정하겠습니다.

```cpp
#include "hittable.h"
///////////////////////// 추가 ////////////////////////////
#include "texture.h"                                    //
//////////////////////////////////////////////////////////

...

class lambertian : public material {
  public:
///////////////////////// 수정 ////////////////////////////////////////////////////
    lambertian(const color& albedo) : tex(make_shared<solid_color>(albedo)) {}  //
//////////////////////////////////////////////////////////////////////////////////
///////////////////////// 추가 ////////////////////////////////////////////////////
    lambertian(shared_ptr<texture> tex) : tex(tex) {}                           //
//////////////////////////////////////////////////////////////////////////////////

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered) const override {
      auto scatter_direction = rec.normal + random_unit_vector();

      // Catch degenerate scatter direction
      if (scatter_direction.near_zero())
        scatter_direction = rec.normal;

      scattered = ray(rec.p, scatter_direction, r_in.time());
///////////////////////// 수정 ////////////////////////////////////////////////////
      attenuation = tex->value(rec.u, rec.v, rec.p);                            //
//////////////////////////////////////////////////////////////////////////////////
      return true;
    }
  
  private:
///////////////////////// 수정 ////////////////////////////////////////////////////
    shared_ptr<texture> tex;                                                    //
///////////////////////// 수정 ////////////////////////////////////////////////////
};
```

**<p align="center">Listing 25:** [material.h] _Lambertian material with texture_

위에서 작업한 것들을 main 씬에 적용하면

```cpp
#include "rtweekend.h"

#include "bvh.h"
#include "camera.h"
#include "hittable.h"
#include "hittable_list.h"
#include "material.h"
#include "sphere.h"
///////////////////////// 추가 //////////////////////////
#include "texture.h"                                  //
////////////////////////////////////////////////////////

int main() {
  hittable_list world;

///////////////////////// 수정 ////////////////////////////////////////////////////////////////////
  auto checker = make_shared<checker_texture>(0.32, color(.2, .3, .1), color(.9, .9, .9));      //
  world.add(make_shared<sphere>(point3(0, -1000, 0), 1000, make_shared<lambertian>(checker)));  //
//////////////////////////////////////////////////////////////////////////////////////////////////

  for (int a = -11; a < 11; a++) {
  ...
}

```

**<p align="center">Listing 26:** [<span>main</span>.cc] _Checkered texture in use_

다음과 같은 결과를 확인할 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-2.02-checker-ground.png"></p>

**<p align="center">Image 2:** _Spheres on checkered ground</p>_

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