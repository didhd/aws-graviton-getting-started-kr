# Graviton 에서의 Go

Go는 구글에서 설계한 정적 타이핑 컴파일 프로그래밍 언어입니다. Go는 arm64를  지원하며 성능을 향상시키는 최근 변경 사항과 함께 모든 일반 배포판에서 사용할 수 있으므로 최신 버전의 Go 컴파일러와 툴체인을 사용하십시오.

다음은 몇 가지 주목할 만한 성능 업그레이드입니다:


## Go 1.18 \[released 2022/03/14\]
Go 컴파일러의 주요 구현인 [golang/go](https://github.com/golang/go)이 스택 대신 레지스터를 사용하여 
함수 인수 및 결과를 전달하는 새로운 방법을 구현하여 Arm에서의 성능을 개선하였습니다. 이 변경 사항은 1.17의 x86-64에서 사용 가능했었는데, 약 5%의 성능 향상을 가져왔습니다. Arm에서의 이러한 변경은 일반적으로 10% 이상의 더 높은 성능 향상을 제공합니다.

Go 1.18의 성능 향상을 통해 얻을 수 있는 활용 사례에 대해 자세히 알아보려면 블로그 게시물을 확인하십시오. [Go 1.18 및 AWS Graviton을 통해 Go 워크로드를 최대 20% 더 빠르게 만들기](https://aws.amazon.com/blogs/compute/making-your-go-workloads-up-to-20-faster-with-go-1-18-and-aws-graviton/).

## Go 1.17 \[released 2021/08/16\]
Go 컴파일러의 주요 구현인 [golang/go](https://github.com/golang/go)는 다음과 같은 표준 라이브러리 패키지의 성능을 향상시켰습니다:

- crypto/ed25519 - 패키지가 다시 작성되었으며, 모든 작업이 현재 arm64와 amd64 모두에서 약 2배 더 빠릅니다.
- crypto/elliptic - CurveParams 메서드는 이제 사용 가능한 경우 알려진 곡선(P-224, P-256 및 P-521)에 대해 더 빠르고 안전한 전용 구현을 자동으로 호출합니다. P521 곡선 구현도 다시 작성되었으며 현재 amd64 및 arm64에서 상수 시간으로 동작하고 3배 더 빨라졌습니다.


## Go 1.16 \[released 2021/02/16\]
Go 컴파일러의 주요 구현체인 [golang/go](https://github.com/golang/go)는 아래와 같은 몇 가지 변경 사항으로 Arm의 성능을 향상시켰습니다. Go 1.16으로 프로젝트를 구축하면 다음과 같은 개선 효과를 얻을 수 있습니다:

 * [ARMv8.1-A Atomics 명령어](https://go-review.googlesource.com/c/go/+/234217), Graviton 2와 v8.1 이상의 최신 명령 집합을 가진 현대 암 코어의 뮤텍스 공정성과 속도를 획기적으로 개선합니다.
.
 * [copy 성능 개선](https://go-review.googlesource.com/c/go/+/243357), 특히 주소가 정렬되지 않은 경우.

## 최근에 업데이트된 패키지
Arm의 성능을 향상시키는 일반적으로 사용되는 패키지를 변경하면 경우에 따라 성능이 눈에 띄게 달라질 수 있습니다. 다음은 알아야 할 패키지의 일부 목록입니다.

Package   | Version   | Improvements
----------|-----------|-------------
[Snappy](https://github.com/golang/snappy) | 커밋 [196ae77](https://github.com/golang/snappy/commit/196ae77b8a26000fa30caa8b2b541e09674dbc43) | hot path 기능의 어셈블리 구현이 amd64에서 arm64로 이식되었습니다.

