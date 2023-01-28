## 1. Overview

Ray Tracing in One Weekend에서는 단순한 브루트 포스 패스 트레이서(brute force path tracer)를 만들어보았습니다. 이번에는 텍스처, 볼륨(예:안개), 직사각형, 인스턴스, 조명을 추가하고 BVH를 사용하여 많은 수의 객체들을 지원할 것입니다. 그러고나면, "진짜" 레이 트레이서를 갖게될 것입니다.

많은 사람들(저를 포함한)은 레이 트레이싱에서 대부분의 최적화가 속도를 크게 향상시키지 않고 코드를 복잡하게 만든다고 생각합니다. 이 미니북에서는 각 디자인 결정에서 가장 간단한 접근 방식을 따를 것입니다. 보다 정교한 접근 방식은 https://in1weekend.blogspot.com/ 를 확인하세요. 그러나, 너무 이른 최적화를 하지 않는것을 강력히 권장합니다. 실행 시간 프로파일에서 높게 보이지 않는다면, 모든 기능을 지원하기 전에 최적화를 할 필요가 없습니다!

이 책의 가장 어려운 두 파트는 BVH(Bounding Volume Hierarchy)와 펄린 텍스처(Perlin textures)입니다. 제목에서 주말(weekend)이 아닌 일주일(a week)이라고 한것이 바로 이런 이유입니다. 하지만 주말 프로젝트로 하길 원한다면 이 두 파트는 나중에 할 수도 있습니다. 순서는 이 책에서 그다지 중요하지 않습니다. BVH와 펄린 텍스처가 없이도 코넬 박스(Cornell Box)를 만들 수 있습니다!

이 프로젝트에 도움을 주신 모든 분들에게 감사드립니다. 이 분들의 이름은 마지막 색션인 [acknowledgments]() 색션에서 볼 수 있습니다.

---

## 출처
[Ray Tracing: The Next Week - 1 Overview](https://raytracing.github.io/books/RayTracingTheNextWeek.html#overview)