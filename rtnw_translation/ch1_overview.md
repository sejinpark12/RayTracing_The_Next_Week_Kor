## 1 Overview
Ray Tracing in One Weekend에서는 단순 브루트포스 패스 트레이서(brute force path tracer)를 만들었습니다. 이번 Ray Tracing The Next Week에서는 텍스처, 볼륨(예: 안개), 직사각형, 인스턴스, 광원을 추가하고, BVH를 사용하여 많은 오브젝트를 처리하는 기능을 추가할 것입니다. 이 과정을 마치고 나면, 이제 "제대로 된" 레이 트레이서를 완성할 수 있습니다.

레이 트레이싱에서 대부분의 최적화가 속도 향상이 크게 없으면서 코드만 복잡하게 만든다고, 저를 포함한 많은 사람들은 생각합니다. 그래서 이 책에서는 설계적 결정을 내릴 때마다 가장 단순한 접근 방식을 따를 것입니다. 프로젝트 관련 추가 자료는 [Further Reading](https://github.com/RayTracing/raytracing.github.io/wiki/Further-Readings) 위키 페이지를 참고하세요. 그러나 너무 이른 최적화는 정말로 권장하지 않습니다. 최적화하려는 부분이 실행 시간 프로파일에서 주요 병목으로 나타나지 않는다면, 모든 기능이 다 구현되기 전까지 그 부분을 최적화할 필요가 없습니다!

이 책에서 가장 어려운 두 파트는 BVH(Bounding Volume Hierarchy)와 펄린 텍스처(Perlin texture)입니다. 그래서 책 제목에서도 주말(a weekend)이 아닌 일주일(a week) 동안 진행하라고 한 것입니다. 만약 주말 프로젝트로 진행하고 싶다면 두 파트를 나중으로 미룰 수도 있습니다. 이 책에서 다루는 개념을 이해하는 데 순서는 크게 중요하지 않습니다. 그리고 BVH와 펄린 텍스처를 구현하지 않더라도 코넬 박스(Cornell Box)를 만들 수 있습니다!

프로젝트에 대한 정보는 [프로젝트 README](https://github.com/RayTracing/raytracing.github.io/blob/release/README.md) 파일을 참고하세요. 여기에는 GitHub 저장소, 디렉토리 구조, 빌드/실행 방법, 수정 및 기여 방법이 포함되어 있습니다.

이 시리즈의 책들은 브라우저에서 바로 인쇄해도 제대로 출력되도록 편집되었습니다. 또한 [각 release](https://github.com/RayTracing/raytracing.github.io/releases/) 의 "Assets" 섹션에서 PDF를 다운로드할 수 있습니다.

프로젝트에 도움을 주신 모든 분께 감사드립니다. 책 마지막 부분의 [acknowledgments](https://raytracing.github.io/books/RayTracingTheNextWeek.html#acknowledgments) 섹션에서 명단을 확인할 수 있습니다.

---

## 출처
[Ray Tracing: The Next Week - 1 Overview](https://raytracing.github.io/books/RayTracingTheNextWeek.html#overview)