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
이제 두 번째 씬을 추가하겠습니다. 그리고 앞으로 진행하면서 새로운 씬들을 더 추가할 예정입니다. 실행 시에 원하는 씬을 쉽게 선택하도록 switch 문을 사용하겠습니다. 세련되지 않은 방식이지만, 코드를 아주 간단하게 유지하여 레이 트레이싱에 집중하기 위함입니다. 여러분의 레이 트레이서에서는 명령문 인자를 적용하는 것과 같은 다른 방식을 적용하는 것도 좋습니다.

<span>main.cc</span>의 랜덤 구 씬을 리펙토링하면 다음과 같습니다. `main()` 의 함수명을 `bouncing_spheres()` 로 변경하고, `bouncing_spheres()` 함수를 호출하기 위한 새로운 `main()` 함수를 추가합니다.

```cpp
#include "rtweekend.h"

#include "bvh.h"
#include "camera.h"
#include "hittable.h"
#include "hittable_list.h"
#include "material.h"
#include "sphere.h"
#include "texture.h"

///////////////////////// 삭제 //////////////////////////
// int main() {                                       //
////////////////////////////////////////////////////////
///////////////////////// 추가 //////////////////////////
void bouncing_spheres() {                             //
////////////////////////////////////////////////////////
  hittable_list world;

  auto ground_material = make_shared<lambertian>(color(0.5, 0.5, 0.5));
  world.add(make_shared<sphere>(point3(0, -1000, 0), 1000, ground_material));

  ...

  cam.render(world);
}

///////////////////////// 추가 //////////////////////////
int main() {                                          //
  bouncing_spheres();                                 //
}                                                     //
////////////////////////////////////////////////////////
```

**<p align="center">Listing 27:** [<span>main</span>.cc] _Main dispatching to selected scene_

이제 체크 패턴 구 두 개를 위아래로 배치한 씬을 추가합니다.

```cpp
#include "rtweekend.h"

#include "bvh.h"
#include "camera.h"
#include "hittable.h"
#include "hittable_list.h"
#include "material.h"
#include "sphere.h"
#include "texture.h"

void bouncing_spheres() {
  ...
}

///////////////////////// 추가 ////////////////////////////////////////////////////////////////
void checkered_spheres() {                                                                  //
  hittable_list world;                                                                      //
                                                                                            //
  auto checker = make_shared<checker_texture>(0.32, color(.2, .3, .1), color(.9, .9, .9));  //
                                                                                            //
  world.add(make_shared<sphere>(point3(0, -10, 0), 10, make_shared<lambertian>(checker)));  //
  world.add(make_shared<sphere>(point3(0,  10, 0), 10, make_shared<lambertian>(checker)));  //
                                                                                            //
  camera cam;                                                                               //
                                                                                            //
  cam.aspect_ratio      = 16.0 / 9.0;                                                       //
  cam.image_width       = 400;                                                              //
  cam.samples_per_pixel = 100;                                                              //
  cam.max_depth         = 50;                                                               //
                                                                                            //
  cam.vfov     = 20;                                                                        //
  cam.lookfrom = point3(13, 2, 3);                                                          //
  cam.lookat   = point3(0, 0, 0);                                                           //
  cam.vup      = vec3(0, 1, 0);                                                             //
                                                                                            //
  cam.defocus_angle = 0;                                                                    //
                                                                                            //
  cam.render(world);                                                                        //
}                                                                                           //
//////////////////////////////////////////////////////////////////////////////////////////////


int main() {
///////////////////////// 수정 ////////////////////////////////////////////////////////////////
  switch (2) {                                                                              //
    case 1: bouncing_spheres();  break;                                                     //
    case 2: checkered_spheres(); break;                                                     //
  }                                                                                         //
//////////////////////////////////////////////////////////////////////////////////////////////
}
```

**<p align="center">Listing 28:** [<span>main</span>.cc] _Two checkered spheres_

결과는 다음과 같습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-2.03-checker-spheres.png"></p>

**<p align="center">Image 3:** _Checkered spheres</p>_

렌더링 결과가 조금 이상하게 보인다고 생각할 수 있습니다. `checker_texture` 가 3차원 공간에 정의된 spatial texture이기 때문에, 구 표면이 3차원 체크 공간을 가로질러 지나가게 됩니다. 그 결과, 구 표면의 각 점이 3차원 체크 공간의 어떤 색 영역에 있는지에 따라 체크 패턴이 나타납니다. 체크 패턴이 완벽하거나, 적어도 그럴듯한 경우가 많이 존재합니다. 하지만 그렇지 않은 경우에서도 오브젝트의 표면에 체크 패턴이 일정하길 원합니다. 이 방식은 다음에서 다루겠습니다.

---

### 4.4 Texture Coordinates for Spheres
상수 색상 텍스처는 좌표를 사용하지 않습니다. solid(또는 spatial) texture는 3차원 공간상의 점 좌표를 사용합니다. 이제는 $u, v$ 텍스처 좌표를 사용할 때입니다. $u, v$ 텍스처 좌표는 2D 이미지(또는 어떤 2D 파라미터 공간)에서의 위치를 가리킵니다. 이 텍스처 좌표를 계산하기 위해서는, 3D 오브젝트 표면의 어떤 점에 대해서든지 $u, v$ 좌표를 구할 수 있어야 합니다. 이 매핑 방식에는 절대적인 정답이 있지는 않지만, 일반적으로는 표면 전체에 대응되면서 2D 이미지를 스케일링, 회전하고 늘려서 적당한 형태로 매핑되는 것이 바람직합니다. 먼저, 구의 $u, v$ 텍스처 좌표를 구하는 방법부터 알아보겠습니다.

일반적으로 구의 텍스처 좌표는 경도(longitude), 위도(latitude)와 비슷한 방식인  구면 좌표계 형식으로 정의됩니다. 따라서 구면 좌표계에서는 $(\theta, \phi)$ 를 계산합니다. 여기서 $\theta$ 는 구의 아래쪽 극점에서 위쪽으로(-Y 방향으로부터 위쪽으로) 측정한 각도이고, $\phi$ 는 Y축을 중심으로 도는(-X에서 출발하여 +Z, +X, -Z, 다시 -X로 돌아오는) 방향으로 측정한 각도입니다.

$\theta$ 와 $\phi$ 를 각각 $[0, 1]$ 범위의 텍스처 좌표 $u$ 와 $v$ 로 매핑하겠습니다. $(u = 0, v = 0)$ 은 텍스처의 왼쪽 아래 모서리로 매핑됩니다. 따라서 $(\theta, \phi)$ 에서 $(u, v)$ 로의 정규화는 다음과 같습니다.

$$ u = \frac{\phi}{2\pi} $$
$$ v = \frac{\theta}{\pi} $$

원점 중심 단위 구 위의 주어진 점에 대한 $\theta$ 와 $\phi$ 를 계산하기 위해, 먼저 그 점에 대응하는 데카르트 좌표계(Cartesian coordinates)의 방정식을 사용합니다.

$$ \begin{align*}
    y &= -\cos(\theta)            \\
    x &= -\cos(\phi) \sin(\theta) \\
    z &= \quad\sin(\phi) \sin(\theta)
    \end{align*}
$$

$\theta$ 와 $\phi$ 를 구하기 위해서는 위의 방정식을 뒤집어야 합니다. `<cmath>` 의 `std::atan2()` 함수는 정확히 $\cos(\phi), \sin(\phi)$ 자체를 입력으로 넣어야 하는 게 아니라, sine과 cosine에 각각 같은 비례계수($\sin(\theta)$)가 곱해진 값을 입력으로 받더라도 각도를 리턴하므로, $x$ 와 $z$ 를 인자로 전달하여 $\phi$ 를 구할 수 있습니다. 이때 두 값에 공통으로 들어 있는 $\sin(\theta)$ 항은 상쇄됩니다.

$$ \phi = \mathrm{atan2}(z, -x) $$

`std::atan2()` 은 $-\pi$ 부터 $\pi$ 까지 범위의 값을 리턴하지만, 그 값은 0에서  $\pi$ 까지 증가하다가, 갑자기 $-\pi$ 로 뒤집히고 다시 0까지 증가합니다. 이 방식은 수학적으로는 맞지만, 여기서 원하는 $u$ 의 범위는 $0$에서 $1/2$ 로 증가하다가, 갑자기 $-1/2$ 에서 다시 $0$ 으로 증가하는 형태가 아닌, 0에서 1까지 한 방향으로 증가하는 형태입니다. `atan2(a, b)` 가 벡터 $(b, a)$ 의 각도를 구한다고 생각해 보겠습니다. 두 입력의 부호를 모두 뒤집으면

$$ (b,a) \rightarrow (-b,-a) $$

위와 같이 되고, 이것은 원래 벡터를 정확히 180도, 즉 $\pi$ 만큼 회전시킨 것입니다. 따라서 벡터 $(b, a)$ 와 벡터 $(-b, -a)$ 의 각도 차이는 항상 $\pi$ 입니다. 하지만 각도는 한 바퀴($2\pi$) 를 돌면 같은 방향이 되므로 $\phi$ 와 $\phi + 2\pi$ 는 같은 방향입니다. 따라서 다음 공식이 성립하게 됩니다.

$$ \mathrm{atan2}(a,b) = \mathrm{atan2}(-a,-b) + \pi, $$

위 공식의 오른쪽 항을 사용하면 $0$ 에서 $2\pi$ 까지 연속적으로 증가하는 값을 얻을 수 있습니다. 따라서, $\phi$ 를 다음과 같이 계산할 수 있습니다.

$$ \phi = \mathrm{atan2}(-z, x) + \pi $$

$\theta$ 값을 유도하는 것은 더 간단합니다.

$$ \theta = \arccos(-y) $$

따라서 구의 $(u, v)$ 좌표는, 원점을 중심으로 하는 단위 구 표면의 점을 입력으로 받는 유틸리티 함수로 계산합니다.

```cpp
class sphere : public hittable {
  ...
  private:
    ...

///////////////////////// 추가 ////////////////////////////////////////////////////
    static void get_sphere_uv(const point3& p, double& u, double& v) {          //
      // p: a given point on the sphere of radius one, centered at the origin.  //
      // u: returned value [0, 1] of angle around the Y axis from X = -1.       //
      // v: returned value [0, 1] of angle from Y = -1 to Y = +1.               //
      //    <1 0 0> yields <0.50 0.50>    <-1  0  0> yields <0.00 0.50>         //
      //    <0 1 0> yields <0.50 1.00>    < 0 -1  0> yields <0.50 0.00>         //
      //    <0 0 1> yields <0.25 0.50>    < 0  0 -1> yields <0.75 0.50>         //
                                                                                //
      auto theta = std::acos(-p.y());                                           //
      auto phi = std::atan2(-p.z(), p.x()) + pi;                                //
                                                                                //
      u = phi / (2 * pi);                                                       //
      v = theta / pi;                                                           //
    }                                                                           //
//////////////////////////////////////////////////////////////////////////////////
};
```

**<p align="center">Listing 29:** [sphere.h] _get\_sphere\_uv function_

`sphere::hit()` 함수 안에서 `get_sphere_uv()` 함수를 사용하도록 수정하여 hit record의 UV 좌표를 업데이트합니다.

```cpp
class sphere : public hittable {
  public:
    ...
    bool hit(const ray& r, interval ray_t, hit_record& rec) const override {
      ...

      rec.t = root;
      rec.p = r.at(rec.t);
      vec3 outward_normal = (rec.p - current_center) / radius;
      rec.set_face_normal(r, outward_normal);
///////////////////////// 추가 //////////////////////////////////
      get_sphere_uv(outward_normal, rec.u, rec.v);            //
////////////////////////////////////////////////////////////////
      rec.mat = mat;

      return true;
    }
    ...
};
```

**<p align="center">Listing 30:** [sphere.h] _Sphere UV coordinates from hit_

교차점 $\mathbf{P}$ 를 사용하여, 해당 표면의 $(u,v)$ 표면 좌표를 계산합니다. 이 $(u,v)$ 표면 좌표로 procedural solid texture(대리석과 같은)에서 해당 위치 값을 조회합니다. 또한 이미지를 읽은 뒤, $(u,v)$ 텍스처 좌표를 사용하여 이미지의 해당 위치를 조회할 수도 있습니다.

스케일된 $(u, v)$ 좌표를 이미지에서 직접 사용하는 방법은 $u$ 와 $v$ 를 정수로 반올림하여 그 값을 $(i, j)$ 픽셀 좌표로 사용하는 것입니다. 하지만 이 방법은 불편합니다. 이미지 해상도를 변경할 때마다 코드를 수정해야만 하기 때문입니다. 따라서 그 대신에, 그래픽스에서 가장 널리 쓰이는 비공식 표준 방법 중 하나는 이미지 픽셀 좌표를 직접 쓰는 것 대신에 해상도와 독립적인 텍스처 좌표를 사용하는 것입니다. 이 텍스처 좌표는 이미지 안의 위치를 단지 비율로 나타낸 값일 뿐입니다. 예를 들어, 가로 $N_x$ 세로 $N_y$ 크기 이미지의 픽셀 좌표 $(i, j)$ 에서 이미지 텍스처 좌표는 다음과 같습니다.

$$ u = \frac{i}{N_x-1} $$
$$ v = \frac{j}{N_y-1} $$

이 값은 위치를 비율로 나타낸 것일 뿐입니다.

---

### 4.5 Accessing Texture Image Data

---

### 4.6 Rendering The Image Texture

---

## 출처

[Ray Tracing: The Next Week - 4 Texture Mapping](https://raytracing.github.io/books/RayTracingTheNextWeek.html#texturemapping)