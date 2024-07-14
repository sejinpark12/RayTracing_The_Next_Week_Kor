## 2. Motion Blur

레이 트레이싱을 한다는 것은 런타임보다 비주얼 퀄리티를 더 중요하게 생각한다는 것입니다. 흐릿한 반사(fuzzy reflection)와 디포커스 블러(defocus blur)를 렌더링할 때, 픽셀 당 여러 샘플(multiple samples per pixel)을 사용했습니다. 긍정적인 사실은 거의 모든 효과가 비슷하게 브루트 포스로 적용될 수 있다는 것입니다. 모션 블러도 그 중에 하나입니다.

실제 카메라에서는 셔터가 짧은 시간 간격으로 열려 있으며, 그 시간 동안 카메라와 물체가 움직일 수 있습니다. 이러한 카메라 샷을 정확하게 재현하기 위해는 셔터가 열려 있는 동안 카메라가 감지하는 것들의 평균을 구합니다.

### 2.1. Introduction of SpaceTime Ray Tracing

셔터가 열려있는 동안 임의의 순간에 단일 광선을 보내면 단일(단순화된) 광자에 대한 임의의 추정치를 얻을 수 있습니다. 물체가 그 순간에 어디에 있는지 파악할 수 있다면 동일한 순간에 해당 광선에 대한 빛을 정확하게 측정 할 수 있습니다. 이건 랜덤(몬테 카를로) 레이 트레이싱이 얼마나 간단한지를 보여주는 또 다른 예입니다. 브루트 포스가 또 다시 승리했습니다!

레이 트레이서의 "엔진"은 각 광선에 대해 물체가 필요한 위치에 있는지만 확인하면 되므로 교차점은 크게 변하지 않습니다. 이것을 수행하기 위해서, 각 광선에 대한 정확한 시간을 저장해야 합니다:

```c++
class ray {
public:
    ray() {}

    ////////////////////////추가////////////////////////////////////
    ray(const point3& origin, const vec3& direction)
    : orig(origin), dir(direction), tm(0) {}

    ray(const point3& origin, const vec3& direction, double time)
    : orig(origin), dir(direction), tm(time) {}
    ////////////////////////////////////////////////////////////////

    const point3& origin() const { return orig; }
    const vec3& direction() const { return dir; }

    ////////////////////////추가////////////////////////////////////
    double time() const { returm tm; }
    ////////////////////////////////////////////////////////////////

    point3 at(double t) const {
        return orig + t * dir;
    }

private:
    point3 orig;
    vec3 dir;
    ////////////////////////추가////////////////////////////////////
    double tm;
    ////////////////////////////////////////////////////////////////
};
```
**Listing 1:** [ray.h] Ray with time information

### 2.2. Updating the Camera to Simulate Motion Blur
### 2.3. Adding Moving Spheres
### 2.4. Tracking the Time of Ray Intersection
### 2.5. Putting Everything Together