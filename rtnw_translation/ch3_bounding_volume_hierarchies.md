## 3 Bounding Volume Hierarchies
BVH는 레이 트레이서에서 가장 어렵고 복잡한 부분입니다. 이번 쳅터에서는 BVH를 구현하여 코드가 더 빠르게 실행될 수 있도록 하겠습니다. 또한 BVH 구현 과정에서 `hittable` 구조를 리팩토링할 것입니다. 그러면 직사각형과 박스를 추가할 때, 다시 앞으로 돌아가서 이것들을 리팩토링하지 않아도 됩니다.

레이 트레이서에서 광선-오브젝트 교차는 시간적으로 가장 큰 병목이고, 오브젝트 수에 비례하여 선형적(linear)으로 실행시간이 증가합니다. 광선-오브젝트 교차는 동일 씬에 대한 반복적인 탐색이므로, 이진 탐색과 같은 로그시간 탐색(logarithmic search)을 할 수 있어야 합니다. 동일 씬에 수백만에서 수십억 개의 광선을 쏘기 때문에, 씬의 모델들을 미리 한 번에 정렬하는 과정이 필요합니다. 그러면 각 광선의 교차 탐색을 선형보다 빠르게(sublinear search) 수행할 수 있습니다. 가장 일반적인 정렬 방법 두 가지는 1) 공간을 세분화하는 방법과 2) 오브젝트를 세분화하는 방법입니다. 두 번째 방법이 일반적으로 코딩하기 더 쉽고, 실행 속도도 대부분의 모델에서 첫 번째 방법과 비슷하게 빠릅니다.

### 3.1 The Key Idea
프리미티브 세트에 대한 바운딩 볼륨을 생성하는 핵심 아이디어는 모든 오브젝트를 완전히 감싸는 볼륨을 찾는 것입니다. 예를 들어, 10개의 오브젝트를 감싸는 구를 계산한다고 가정해 보겠습니다. 광선이 이 바운딩 구(bounding sphere)와 교차하지 않는다면 그 광선은 바운딩 구 내부에 존재하는 10개의 오브젝트 전부와도 절대 교차하지 않습니다. 그러나 광선이 바운딩 구와 교차한다면, 10개의 오브젝트 중 하나와 교차할 가능성이 있습니다. 따라서 바운딩 관련 코드는 항상 다음과 같은 형태입니다.

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
위에서 설명한 것들이 제대로 동작하기 위해서는, 오브젝트를 효율적으로 분할하는 방법과 광선을 바운딩 볼륨에 교차시키는 방법이 필요합니다. 그리고 광선와 바운딩 볼륨의 교차는 빨라야 하고, 바운딩 볼륨의 크기는 상당히 작아야 합니다. 실제로 대부분의 모델에서는 축 정렬 박스(axis-aligned boxes)를 사용한 방식이 위에서 설명한 구형 바운딩 볼륨(bounding sphere)을 사용한 방식이나 다른 방식들보다 더 잘 동작합니다. 하지만 어떤 바운딩 볼륨을 사용할 것인지는 항상 설계적으로 고려해 볼만한 사항입니다.

사실 축 정렬 바운딩 직육면체(axis-aligned bounding rectangular parallelepipeds)가 정확한 명칭이 맞긴하지만, 지금부터는 축 정렬 바운딩 직육면체를 _축 정렬 바운딩 박스_ (_axis-aligned bounding boxes_), 즉 AABB라고 부르겠습니다. 코드에서는 "bounding box"를 "bbox"로 줄여 쓰는 것을 볼 수 있습니다. AABB와 광선의 교차를 계산하는데는 어떠한 방법을 사용해도 좋습니다. 그리고 여기서는 오직 광선과 AABB의 교차 여부만 계산하므로 교차점, 법선 또는 오브젝트를 그리는 데 필요한 그 밖의 어떤 것도 계산할 필요가 없습니다.

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

이 1차원 계산을 2차원이나 3차원 교차 테스트에 적용하기 위한 핵심 아이디어는 각 축에서 얻은 $t$ 구간들의 영역이 겹치는지를 확인하는 것입니다. 광선이 각 축의 모든 평면 쌍으로 둘러싸인 박스와 교차한다면, 각 축에서 구한 모든 $t$ 구간들은 서로 겹치게 됩니다. 예를 들어, 다음 그림과 같이 2차원에서 초록색 $t$ 구간과 파란색 $t$ 구간의 겹침은 광선이 그 바운딩 박스와 교차할 경우에만 발생합니다.

![Figure 4: Ray-slab t-interval overlap](https://raytracing.github.io/images/fig-2.04-ray-slab-interval.jpg)
**<p align="center">Figure 4**: _Ray-slab $t$-interval overlap</p>_

위 그림에서 위쪽 광선의 $t$ 구간은 서로 겹치지 않으므로, 위쪽 광선은 초록색 평면과 파란색 평면으로 둘러싸인 2D 박스와 _교차하지 않는다는_ 것을 알 수 있습니다. 아래쪽 광선은 $t$ 구간이 서로 _겹치므로_, 아래쪽 광선은 그 바운딩 박스와 _교차한다는_ 것을 알 수 있습니다.

---

### 3.4 Ray Intersection with an AABB
### 3.5 Constructing Bounding Boxes for Hittables
### 3.6 Creating Bounding Boxes of Lists of Objects
### 3.7 The BVH Node Class
### 3.8 Splitting BVH Volumes
### 3.9 The Box Comparision Functions
### 3.10 Another BVH Optimization