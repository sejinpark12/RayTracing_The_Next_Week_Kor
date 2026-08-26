## 3 Bounding Volume Hierarchies
BVH는 레이 트레이서에서 가장 어렵고 복잡한 부분입니다. 이번 쳅터에서는 BVH를 구현하여 코드가 더 빠르게 실행되도록 하겠습니다. 또한 BVH 구현 과정에서 `hittable` 구조를 리팩토링할 것입니다. 이렇게 하면 직사각형과 박스를 추가할 때, 다시 앞으로 돌아가서 이것들을 리팩토링하지 않아도 됩니다.

레이 트레이서에서 광선-오브젝트 교차는 시간적으로 가장 큰 병목이고, 그 실행시간은 오브젝트 수에 비례하여 선형적(linear)으로 증가합니다. 광선-오브젝트 교차는 동일 씬에 대한 반복적인 탐색이므로, 이진 탐색과 같은 로그시간 탐색(logarithmic search)을 할 수 있어야 합니다. 동일 씬에 수백만에서 수십억 개의 광선을 쏘기 때문에, 씬의 모델들을 미리 한 번에 정렬하는 과정이 필요합니다. 그러면 각 광선의 교차 탐색을 선형보다 빠르게(sublinear search) 수행할 수 있습니다. 가장 일반적인 정렬 방법 두 가지는 1) 공간을 세분화하는 방법과 2) 오브젝트를 세분화하는 방법입니다. 두 번째 방법이 일반적으로 코딩하기 더 쉽고, 실행 속도도 대부분의 모델에서 첫 번째 방법과 비슷하게 빠릅니다.

### 3.1 The Key Idea
프리미티브 세트에 대한 바운딩 볼륨을 만드는데 필요한 핵심 아이디어는 모든 오브젝트를 완전히 감싸는 볼륨을 찾는 것입니다. 예를 들어, 10개의 오브젝트를 감싸는 구를 계산한다고 가정해 보겠습니다. 광선이 이 바운딩 구(bounding sphere)와 교차하지 않는다면 그 광선은 바운딩 구 내부에 존재하는 10개의 오브젝트 전부와도 절대 교차하지 않습니다. 그러나 광선이 바운딩 구와 교차한다면, 10개의 오브젝트 중 하나와 교차할 가능성이 있습니다. 따라서 바운딩 관련 코드는 항상 다음과 같은 형태입니다.

```cpp
if (광선과 바운딩 오브젝트의 교차)
    return 광선과 바운딩 오브젝트 내부의 오브젝트와의 교차 여부
else
    return false
```

여기서 주목해야 할 점은 씬의 오브젝트를 여러 서브그룹으로 그룹화하는데 바운딩 볼륨을 사용한다는 것입니다. 스크린이나 씬 공간 자체를 나누는 것이 아닙니다. 즉, BVH는 공간 분할이 아니라 오브젝트 분할입니다. 각각의 오브젝트는 같은 계층 수준에서 단 하나의 바운딩 볼륨에만 포함되어야 합니다만, 공간 분할이 아니므로 바운딩 볼륨의 공간적 영역 자체는 서로 겹칠 수 있습니다.

---

### 3.2 Hierarchies of Bounding Volumes
광선의 교차 탐색을 선형 탐색보다 빠르게 만드려면 바운딩 볼륨을 계층적으로 구성해야 합니다. 예를 들어, 오브젝트 세트를 빨간색과 파란색 두 그룹으로 나누고 직사각형 바운딩 볼륨을 사용한다면 코드는 다음과 같습니다.

![Figure 1: Bounding volume hierarchy](https://raytracing.github.io/images/fig-2.01-bvol-hierarchy.jpg)
**<p align="center">Figure 1**: _Bounding volume hierarchy</p>_

```cpp
if (보라색 바운딩 볼륨과 교차)
    hit0 = 파란색 바운딩 볼륨에 포함된 오브젝트들과 교차
    hit1 = 빨간색 바운딩 볼륨에 포함된 오브젝트들과 교차
    if (hit0 or hit1)
        return true and 둘 중 더 가까운 교차 정보 반환
return false
```

---

### 3.3 Axis-Aligned Bounding Boxes (AABBs)
위에서 설명한 것들이 제대로 동작하기 위해서는, 오브젝트를 효율적으로 분할하는 방법과 광선을 바운딩 볼륨에 교차시키는 방법이 필요합니다. 그리고 광선와 바운딩 볼륨의 교차는 빨라야 하며, 바운딩 볼륨의 크기는 상당히 작아야 합니다. 실제로 대부분의 모델에서는 축 정렬 박스(axis-aligned boxes)를 사용한 방식이 위에서 설명한 구형 바운딩 볼륨(bounding sphere)을 사용한 방식이나 다른 방식들보다 더 잘 동작합니다. 하지만 어떤 바운딩 볼륨을 사용할 것인지는 항상 설계적으로 고려해 볼만한 사항입니다.

사실 축 정렬 바운딩 직육면체(axis-aligned bounding rectangular parallelepipeds)가 정확한 명칭이 맞긴하지만, 지금부터는 축 정렬 바운딩 직육면체를 _축 정렬 바운딩 박스_ (_axis-aligned bounding boxes_), 즉 AABB라고 부르겠습니다. 코드에서는 "bounding box"를 "bbox"로 줄여 쓰는 것을 볼 수 있습니다. AABB와 광선의 교차를 계산하는데는 어떠한 방법을 사용해도 좋습니다. 또한, 여기서는 오직 광선과 AABB의 교차 여부만 계산하므로 교차점, 법선, 오브젝트를 그리는 데 필요한 그 어떤 것도 계산할 필요가 없습니다.

광선과 AABB의 교차 계산에는 보통 "slab"이라는 방법을 사용합니다. 이 방법은 n차원 AABB가 흔히 "slabs"라고 불리는 $n$ 개의 축 정렬 구간들의 교집합이라는 사실에 기반합니다. 구간이란 두 끝점 사이에 존재하는 점들이라는 것을 기억하세요. 예를 들어, $3 \leq x \leq 5$ 를 만족하는 $x$, 또는 더 간결하게 $[3,5]$ 를 만족하는 $x$ 라고 표현할 수 있습니다. 2차원에서 AABB(직사각형)는 x축과 y축에 대한 두 구간의 겹치는 영역으로 정의됩니다. 

![Figure 2: 2D axis-aligned bounding box](https://raytracing.github.io/images/fig-2.02-2d-aabb.jpg)
**<p align="center">Figure 2**: _2D axis-aligned bounding box</p>_

광선이 구간과 교차하는지 알기 위해서는, 먼저 광선이 그 구간의 경계와 교차하는지 알아야 합니다. 예를 들어, 1차원에서 광선과 두 평면(경계)의 교차로부터 광선 파라미터 $t_0$ 와 $t_1$ 를 계산할 수 있습니다. 단, 광선이 평면과 평행하다면 교차는 정의되지 않습니다.

![Figure 3: Ray-slab intersection](https://raytracing.github.io/images/fig-2.03-ray-slab.jpg)
**<p align="center">Figure 3**: _Ray-slab intersection</p>_

어떻게 하면 광선과 평면의 교차를 계산할 수 있을까요? 광선은 파라미터 $t$ 에 대한 위치 $\mathbf{P}(t)$ 를 리턴하는 다음과 같은 함수로 정의된다는 것을 기억하세요.

$$ \mathbf{P}(t) = \mathbf{Q} + t \mathbf{d} $$

이 방정식은 x, y, z 세 좌표에 모두 적용됩니다. 예를 들면 x 좌표에 대해서는 $x(t) = Q_x + t d_x$ 와 같습니다. 이 광선은 아래의 식을 만족하는 파라미터 $t$ 에서 평면 $x = x_0$ 와 교차합니다.

$$ x_0 = Q_x + t_0 d_x $$

따라서 교차점에서의 $t$ 는 다음과 같습니다.

$$ t_0 = \frac{x_0 - Q_x}{d_x} $$

$x_1$ 에서도 마찬가지로 계산하면 됩니다.

$$ t_1 = \frac{x_1 - Q_x}{d_x} $$

이 1차원 교차 계산을 2차원이나 3차원 교차 테스트에 적용하기 위한 핵심 아이디어는 각 축에서 얻은 $t$ 구간들의 영역이 서로 겹치는지를 확인하는 것입니다. 광선이 각 축의 모든 평면 쌍으로 둘러싸인 박스와 교차한다면, 각 축에서 구한 모든 $t$ 구간들은 서로 겹치게 됩니다. 예를 들어, 다음 그림과 같이 2차원에서 초록색 $t$ 구간과 파란색 $t$ 구간의 겹침은 광선이 그 바운딩 박스와 교차할 경우에만 발생합니다.

![Figure 4: Ray-slab t-interval overlap](https://raytracing.github.io/images/fig-2.04-ray-slab-interval.jpg)
**<p align="center">Figure 4**: _Ray-slab $t$-interval overlap</p>_

위 그림에서 위쪽 광선의 $t$ 구간은 서로 겹치지 않으므로, 위쪽 광선은 초록색 평면과 파란색 평면으로 둘러싸인 2D 박스와 _교차하지 않는다는_ 것을 알 수 있습니다. 아래쪽 광선은 $t$ 구간이 서로 _겹치므로_, 아래쪽 광선은 그 바운딩 박스와 _교차한다는_ 것을 알 수 있습니다.

---

### 3.4 Ray Intersection with an AABB
다음 수도 코드는 slab의 $t$ 구간이 서로 겹치는지 여부를 판단합니다.

```cpp
interval_x <- compute_intersection_x (ray, x0, x1)
interval_y <- compute_intersection_y (ray, y0, y1)
return overlaps(interval_x, interval_y)
```

slab이 널리 쓰이는 이유는 연산이 놀라울 정도로 간단하고, 3D 버전으로도 손쉽게 확장시킬 수 있다는 점 때문입니다.

```cpp
interval_x <- compute_intersection_x (ray, x0, x1)
interval_y <- compute_intersection_y (ray, y0, y1)
interval_z <- compute_intersection_z (ray, z0, z1)
return overlaps(interval_x, interval_y, interval_z)
```

하지만 위의 기본 형태에서 코드를 더 복잡하게 만드는 몇가지 사항들이 존재합니다. 1차원에서의 $t_0$ , $t_1$ 에 대한 방정식을 다시 살펴 보겠습니다.

$$ t_0 = \frac{x_0 - Q_x}{d_x} $$
$$ t_1 = \frac{x_1 - Q_x}{d_x} $$

첫 번째로, 광선이 $\mathbf{x}$ 축의 음의 방향으로 진행한다고 가정해 보겠습니다. 위 식으로 계산한 구간 $(t_{x0}, t_{x1})$ 은 예를 들어 $(7, 3)$ 와 같이 순서가 뒤집힐 수도 있습니다. 두 번째로, 광선이 평면과 평행할 경우에 분모 $d_x$ 가 0이 되어 $t$ 값은 무한대가 될 수 있습니다. 그리고 광선의 원점이 두 slab 경계들 중 한 slab 위에 놓인다면, 분자는 0이 될 수 있습니다. 따라서 만약 광선이 평면과 평행한 동시에 평면 위에 놓인다면 분모, 분자가 모두 0이므로 $t$ 값은 `NaN` 가 될 것입니다. 또한, IEEE 부동소수점에서는 0도 +, - 부호를 가집니다.

$d_x = 0$ 인 경우에서 다행인 점은, 광선 원점($Q_x$)이 $x_0$ 와 $x_1$ 사이에 존재하지 않는다면(광선이 평면과 평행) $t_{x0}$ 와 $t_{x1}$ 이 같은 값이라는 것입니다. 두 값은 둘 다 +∞ 이거나 -∞ 입니다. 그러므로 `min`, `max` 함수를 사용하면 올바른 값을 구할 수 있습니다.

$$ t_{x0} = \min(
    \frac{x_0 - Q_x}{d_x},
    \frac{x_1 - Q_x}{d_x})
$$

$$ t_{x1} = \max(
    \frac{x_0 - Q_x}{d_x},
    \frac{x_1 - Q_x}{d_x})
$$

`min`, `max` 방식으로 처리하더라도 아직 문제가 되는 경우는 $d_x = 0$ 이면서 동시에 $x_0 - Q_x = 0$ 이거나 $x_1 - Q_x = 0$ (광선이 slab 경계와 평행하면서, 동시에 그 경계 위에 놓여 있는 경우)되어 `NaN` 이 발생하는 경우입니다. 이런 경우에는 이를 교차한 것으로 간주하든 교차하지 않은 것으로 간주하든 상관없지만, 이 문제에 대해서는 아래에서 다시 다루겠습니다.

이제, 수도 함수 `overlaps` 를 살펴보겠습니다. 각 구간의 끝이 거꾸로 뒤집히지 않았다고 가정하고, 두 구간이 서로 겹친다면 true를 리턴하도록 합니다. `overlaps()` 는 구간 $t$ 의 `t_interval1` 와 `t_interval2` 에서 겹칩을 검사합니다. 겹치면 true, 겹치지 않으면 false를 리턴합니다.

```cpp
bool overlaps(t_interval1, t_interval2)
    t_min <- max(t_interval1.min, t_interval2.min)
    t_max <- min(t_interval1.max, t_interval2.max)
    return t_min < t_max
```

만약 계산 과정에서 `NaN` 이 하나라도 생긴다면, `overlaps` 의 비교 연산은 false를 리턴할 것입니다. 따라서 광선이 경계를 스치는 경우까지 고려하려면 바운딩 박스에 약간의 패딩을 주어야 합니다. 레이 트레이서에서는 온갖 경우가 언젠가는 다 발생하기 때문에 이런 경우까지 _고려하는 것이 좋습니다_.

그러기 위해서 `interval` 클래스에 새로운 맴버 함수 `expand` 를 추가하겠습니다. `expand` 함수는 구간에 주어진 크기만큼 패딩을 적용합니다.

To accomplish this, we'll first add a new `interval` function `expand`, which pads an interval by a given amount:

```cpp
class interval {
  public:
    ...
    double clamp(double x) const {
      if (x < min) return min;
      if (x > max) return max;
      return x;
    }

///////////////////////// 추가 ////////////////////////////
    interval expand(double delta) const {               //
      auto padding = delta / 2;                         //
      return interval(min - padding, max + padding);    //
    }                                                   //
//////////////////////////////////////////////////////////

    static const interval empty, universe;
};
```

**<p align="center">Listing 7:** [<span>interval</span>.h] _interval::expand() method</p>_

이제 새 AABB 클래스 구현에 필요한 모든 것을 갖추었습니다.

```cpp
#ifndef AABB_H
#define AABB_H

class aabb {
  public:
    interval x, y, z;

    aabb() {} // The default AABB is empty, since intervals are empty by default.

    aabb(const interval& x, const interval& y, const interval& z)
      : x(x), y(y), z(z) {}

    aabb(const point3& a, const point3& b) {
      // Treat the two points a and b as extrema for the bounding box, so we don't require a 
      // particular minimum/maximum coordinate order.

      x = (a[0] <= b[0]) ? interval(a[0], b[0]) : interval(b[0], a[0]);
      y = (a[1] <= b[1]) ? interval(a[1], b[1]) : interval(b[1], a[1]);
      z = (a[2] <= b[2]) ? interval(a[2], b[2]) : interval(b[2], a[2]);
    }

    const interval& axis_interval(int n) const {
      if (n == 1) return y;
      if (n == 2) return z;
      return x;
    }

    bool hit(const ray& r, interval ray_t) const {
      const point3& ray_orig = r.origin();
      const vec3&   ray_dir  = r.direction();

      for (int axis = 0; axis < 3; axis++) {
        const interval& ax = axis_interval(axis);
        const double adinv = 1.0 / ray_dir[axis];

        auto t0 = (ax.min - ray_orig[axis]) * adinv;
        auto t1 = (ax.max - ray_orig[axis]) * adinv;

        if (t0 < t1) {
          if (t0 > ray_t.min) ray_t.min = t0;
          if (t1 < ray_t.max) ray_t.max = t1;
        } else {
          if (t1 > ray_t.min) ray_t.min = t1;
          if (t0 < ray_t.max) ray_t.max = t0;
        }

        if (ray_t.max <= ray_t.min)
          return false;
      }
      return true;
    }
};

#endif
```

**<p align="center">Listing 8:** [<span>aabb</span>.h] _Axis-aligned bounding box class</p>_

### 3.5 Constructing Bounding Boxes for Hittables

---

### 3.6 Creating Bounding Boxes of Lists of Objects

---

### 3.7 The BVH Node Class

---

### 3.8 Splitting BVH Volumes

---

### 3.9 The Box Comparision Functions

---

### 3.10 Another BVH Optimization

---

## 출처

[Ray Tracing: The Next Week - 3 Bounding Volume Hierarchies](https://raytracing.github.io/books/RayTracingTheNextWeek.html#boundingvolumehierarchies)
