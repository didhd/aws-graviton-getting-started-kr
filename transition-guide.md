# 워크로드를 AWS Graviton2 기반 Amazon EC2 인스턴스로 전환할 때 고려 사항

AWS Graviton 프로세서는 Amazon EC2 범용(M6g, M6gd, T4g), 컴퓨팅 최적화(C6gd, C6gn, C7g), 메모리 최적화(R6gd, X2gd) 인스턴스, 스토리지 최적화(Im4gn, Is4gen) 인스턴스 및 다양한 리눅스 워크로드에 최고의 가격 대비 성능을 제공하는 GPU(G5g) 인스턴스를 지원합니다. 그 예로는 애플리케이션 서버, 마이크로 서비스, 고성능 컴퓨팅, CPU 기반 머신 러닝 추론, 비디오 인코딩, 전자 설계 자동화, 게임, 오픈 소스 데이터베이스, 인 메모리 캐시가 있습니다. 대부분의 경우 AWS Graviton으로 전환하는 것은 새 인스턴스 유형 및 관련 운영 체제(OS) Amazon Machine Image([AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html))를 선택하기 위해 Infrastructure-as-code를 업데이트하는 것만큼 간단합니다. 그러나 AWS Graviton 프로세서는 Arm64 명령 집합을 구현하기 때문에 추가적인 소프트웨어 고려사항이 있을 수 있습니다. 이 전환 가이드에서는 워크로드를 평가하여 필요할 수 있는 잠재적인 소프트웨어 변경 사항을 식별하고 해결하는 단계별 접근 방식을 제공합니다.

## 첫단계 - 대상 워크로드 식별

가장 빠르고 쉽게 전환할 수 있는 워크로드는 Linux 기반이며, 소스 코드를 제어하는 오픈 소스 구성 요소 또는 사내 애플리케이션을 사용하여 구축됩니다. 많은 오픈 소스 프로젝트들은 이미 Arm64와 확장판 Graviton을 지원하고 있으며, 소스 코드에 대한 액세스 권한을 가지면 사전 빌드된 아티팩트가 없는 경우 직접 코드로부터 빌드할 수도 있습니다. Graviton에서 사용할 수 있는 서드파티 ISV(Independent Software Vendor) 소프트웨어들도 점점 늘어나고 있습니다 (목록은 [여기](isv.md)에서 확인하세요). 그러나 소프트웨어에 라이센스를 부여한 경우 해당 ISV가 Arm64 명령 집합을 이미 지원했거나 지원할 계획이 있는지를 확인해야 합니다.

다음 전환 가이드는 다음과 같은 논리적 단계로 구성됩니다:

* [학습 및 탐색](#학습-및=탐색)
    * 단계 1 - [선택] Graviton 프로세서 이해 및 주요 문서 검토
    * 단계 2 - 워크로드 탐색 및 현재 소프트웨어 스택 인벤토리 생성
* [워크로드 전환 계획하기](#워크로드-전환-계획하기)
    * 단계 3 - 애플리케이션 환경 설치 및 구성
    * 단계 4 - [선택] 애플리케이션 및/또는 컨테이너 이미지 구축
* [워크로드 테스트 및 최적화](#워크로드-테스트-및-최적화)
    * 단계 5 - 워크로드 테스트 및 최적화
    * 단계 6 - 성능 테스트
* [인프라 및 배포](#인프라-및-배포)
    * 단계 7 - Infrastructure as Code 업데이트
    * 단계 8 - Canary 또는 Blue-Green 배포 수행

### 학습 및 탐색

**단계 1 - [선택] Graviton 프로세서 이해 및 주요 문서 검토**


1. [선택] 다음 영상을 보시고 [re:Invent 2020 - Deep dive on AWS Graviton2 processor-powered EC2 instances](https://youtu.be/NLysl0QvqXU) 그리고 [re:Invent 2021 - Deep dive into AWS Graviton3 and Amazon EC2 C7g instances](https://www.youtube.com/watch?v=WDKwwFQKfSI) 영상을 시청하시어 Graviton 기반 인스턴스에 대한 개요와 운영 체제, 언어 및 런타임에 따라 애플리케이션을 실행하는 방법에 대한 지식을 얻을 수 있습니다.
2. [선택] 다음 영상 [re:Invent 2021 - The journey of silicon innovation at AWS]([https://www.youtube.com/watch?v=Yv3B_Zey83Y](https://www.youtube.com/watch?v=2DCAtpeBABY)을 통해 커스텀 실리콘을 통해 혁신하고자 하는 Amazon의 장기적인 관점을 더 잘 이해합니다.
3. 이 항목의 나머지 부분을 숙지하세요 [Getting started with AWS Graviton repository](README.md). 워크로드 전환 시 유용한 역할을 할 것입니다.

**Step 2 -  워크로드 탐색 및 현재 소프트웨어 스택 인벤토리 생성**

전환을 시작하기 전에 Graviton을 지원하는 동등한 소프트웨어 버전의 경로를 식별할 수 있도록 현재 소프트웨어 스택의 인벤토리를 작성해야 합니다. 이 단계에서는 다운로드하는 소프트웨어(예: 오픈 소스 패키지, 컨테이너 이미지, 라이브러리), 빌드하는 소프트웨어 및 조달/라이센스하는 소프트웨어(예: 모니터링 또는 보안 에이전트)의 관점에서 생각해 보는 것이 유용할 수 있습니다. 검토해야 할 영역:

* [운영체제](os.md), Graviton을 지원하는 특정 버전이 있는지 확인하세요 (대개 최신 버전이 더 좋습니다).
* 워크로드가 컨테이너 기반인 경우 Arm64 지원을 위해 사용하는 컨테이너 이미지를 확인합니다. 이제 많은 컨테이너 이미지가 여러 아키텍처를 지원하므로 혼합 아키텍처 환경에서 이러한 이미지를 간편하게 사용할 수 있습니다. multi-arch 이미지에 대한 자세한 내용은 [ECR multi-arch 지원 공지사항](https://aws.amazon.com/blogs/containers/introducing-multi-architecture-container-images-for-amazon-ecr/)을 참조하세요.
* 응용 프로그램에서 사용하는 모든 라이브러리, 프레임워크 및 런타임.
* 애플리케이션을 빌드, 배포 및 테스트하는 데 사용되는 도구(예: 컴파일러, 테스트 제품군, CI/CD 파이프라인, 프로비저닝 도구 및 스크립트). Graviton 프로세서에서 최고의 성능을 얻는 데 유용한 포인터가 있는 시작 가이드에는 언어별 섹션이 있습니다.
* 운영 환경에 애플리케이션을 배포 및 관리하는 데 사용되는 도구 및/또는 에이전트 (예: 모니터링 도구 또는 보안 에이전트)
* 이 가이드에는 언어별 추가 지침을 확인할 수 있는 언어별 섹션이 포함되어 있습니다:
  * [C/C++](c-c++.md)
  * [Go](golang.md)
  * [Java](java.md)
  * [.NET](dotnet.md) 
  * [Python](python.md)
  * [Rust](rust.md)

일반적으로 소프트웨어 환경이 최신일수록 Graviton에서 전체 성능 자격을 얻을 가능성이 높습니다.

소프트웨어 스택의 각 구성 요소에 대해 Arm64/Graviton 지원 여부를 확인합니다. 이 작업의 대부분은 기존 구성 스크립트를 사용하여 수행할 수 있습니다. 스크립트가 실행 및 설치될 때 누락된 구성 요소에 대한 메시지가 표시되며, 일부는 자동으로 원본에서 빌드되고 다른 일부는 스크립트가 실패할 수 있습니다. 일반적으로 소프트웨어의 최신 버전이 많을수록 전환이 쉽고 Graviton 프로세서에서 완전한 성능을 얻을 가능성이 높기 때문에 소프트웨어 버전에 주의하세요. Graviton을 채택하기 전에 업그레이드를 수행해야 하는 경우 기존 x86 환경을 사용하여 변경된 변수 수를 최소화하는 것이 가장 좋습니다. 우리는 x86에서 OS 버전을 업그레이드하는 것이 업그레이드 후 Graviton으로 전환하는 것보다 훨씬 더 시간이 많이 걸리는 예를 보았습니다. 소프트웨어 지원 확인에 대한 자세한 내용은 부록 A를 참조하세요.

참고: 소프트웨어를 찾을 때 GCC를 포함한 일부 도구는 아키텍처를 AArch64라고 부르고 리눅스 커널을 포함한 다른 도구는 arm64라고 부릅니다. 다양한 저장소에서 패키지를 확인할 때 이러한 다른 이름 지정 규칙을 찾을 수 있습니다.

### 워크로드 전환 계획하기

**Step 3-  애플리케이션 환경 설치 및 구성**

애플리케이션을 전환하고 테스트하려면 적합한 Graviton 환경이 필요합니다. 실행 환경에 따라 다음과 같은 작업이 필요할 수 있습니다:

* Graviton 인스턴스를 부팅할 Arm64 AMI를 가져오거나 만듭니다. AMI를 관리하는 방법에 따라 기존 참조 AMI for Arm64에서 직접 시작하거나 참조 이미지 중 하나에서 특정 종속성을 가진 Golden AMI를 구축할 수 있습니다(AMI 링크가 포함된 지원되는 OS의 전체 목록은 [여기](os.md) 참조);
* 컨테이너 기반 환경을 운영하는 경우 Graviton 기반 인스턴스를 지원하는 기존 클러스터를 구축하거나 확장해야 합니다. Amazon ECS와 EKS 모두 기존 x86 기반 클러스터에 Graviton 기반 인스턴스를 추가할 수 있습니다. ECS의 경우 ECS 클러스터에 Graviton 기반 인스턴스를 추가하여 [AWS ECS-optimized AMI for arm64](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html) 또는 ECS 에이전트를 설치한 후 사용자 자신의 AMI로 시작할 수 있습니다. EKS의 경우 [EKS optimized AMI for arm64](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)를 사용하여 시작된 Graviton 기반 인스턴스로 노드 그룹을 생성해야 합니다.
    * 참고: 동일한 오토 스케일링 그룹에서 Graviton 및 x86 인스턴스를 지원할 수 있습니다. 이 [blog](https://aws.amazon.com/blogs/compute/supporting-aws-graviton2-and-x86-instance-types-in-the-same-auto-scaling-group/)는 Launch 템플릿 재정의 기능을 사용하여 프로세스를 자세히 설명합니다.
* 단계 2에서 생성한 인벤토리에 따라 소프트웨어 스택 설치를 완료합니다.
    * 참고: 대부분의 경우 설치 스크립트를 그대로 사용하거나 필요한 경우 아키텍처 특정 버전의 구성 요소를 참조하기 위해 약간 수정하여 사용할 수 있습니다. 이 과정을 거치는 첫 번째 시간은 나머지 종속성을 해결할 때 반복되는 과정일 수 있습니다.


**Step 4 - [선택] 애플리케이션 및/또는 컨테이너 이미지 구축**

참고: 응용 프로그램 또는 전체 응용 프로그램 스택의 구성 요소를 빌드하지 않을 경우 이 단계를 건너뛸 수 있습니다.

Java, PHP 또는 Node.js 등 인터프리터 언어나 또는 JIT'd 언어를 사용하는 애플리케이션의 경우, 현재 상태로 바로 실행하거나 사소한 수정만으로도 실행될 수 있습니다. 이 저장소에는 [Java](java.md), [Python](python.md), [C/C++](c-c++.md), [Golang](golang.md), [Rust](rust.md) or [.Net](dotnet.md)와 같은 언어별 권장 사항과 함께 별도 섹션이 포함되어 있습니다. 참고: 특정 언어 섹션이 없는 경우(예: PHP Version 7.4+), [여기](README.md#Graviton과-관련된-최신-소프트웨어-업데이트)에 설명된 대로 적절한 최신 버전의 언어를 사용하는 것 이상의 구체적인 지침이 없기 때문입니다. .NET-core는 Graviton 기반 인스턴스의 이점을 누릴 수 있는 훌륭한 방법입니다, 이 [블로그](https://aws.amazon.com/blogs/compute/powering-net-5-with-aws-graviton2-benchmark-results/)는 .NET5 성능을 다룹니다.

C, C++ 또는 Go를 포함한 컴파일된 언어를 사용하는 응용 프로그램은 Arm64 아키텍처를 위해 컴파일되어야 합니다. 대부분의 최신 빌드(예: Make 사용)는 Graviton 기반 인스턴스에서 네이티브로 실행될 때만 작동하지만, 이 저장소에서 언어별 컴파일러 권장 사항을 찾을 수 있습니다: [C/C++](c-c++.md), [Go](golang.md), [Rust](rust.md).

운영 체제와 마찬가지로 컨테이너 이미지는 아키텍처에 따라 다릅니다. arm64 컨테이너 이미지를 빌드해야 합니다. 전환을 쉽게 하려면 x86-64 또는 arm64에서 자동으로 실행할 수 있는 멀티 아키텍처 컨테이너 이미지를 빌드하는 것이 좋습니다.
. 자세한 내용은 이 저장소의 [컨테이너 섹션](containers.md)을 참조하세요. 이 [컨테이너 블로그 포스트](https://aws.amazon.com/blogs/containers/
introducing-multi-architecture-container-images-for-amazon-ecr/)는 다중 아키텍처 컨테이너 이미지 지원에 대한 자세한 개요를 제공합니다. 이는 멀티 아키텍처 환경에서 설정 및 유지 관리를 위한 모범 사례로 간주됩니다. 

또한 기능 및 단위 테스트 제품군을 검토하여 x86 아티팩트에 대해 이미 적용된 테스트 범위와 동일한 새 빌드 아티팩트를 테스트할 수 있는지 확인해야 합니다.

### 워크로드 테스트 및 최적화

**Step 5 - 워크로드 테스트 및 최적화**

이제 Graviton에 애플리케이션 스택이 있으므로 테스트 제품군을 실행하여 모든 정기 장치 및 기능 테스트에 합격하도록 만들어야 합니다. 모든 것이 예상대로 작동할 때까지 응용 프로그램 또는 테스트 제품군의 테스트 실패를 해결합니다. 대부분의 오류는 전환 중에 설치한 수정 및 업데이트된 소프트웨어 버전과 관련이 있어야 합니다(팁: 소프트웨어 버전을 업그레이드할 때 기존 x86 환경을 사용하여 먼저 테스트하여 한 번에 변경되는 변수 수를 최소화할 수 있습니다). 문제가 발생하면 새로운 Graviton 환경을 계속하기 전에 현재 x86 환경을 사용하여 문제를 해결하세요.) 아키텍처 관련 문제가 의심될 경우 해당 문제를 문서화하고 해결 방법에 대한 조언을 제공하는 [C/C++ 섹션](c-c++.md)을 참조하세요. 여전히 불분명한 세부 사항이 있는 경우, AWS 계정 팀이나 AWS 지원팀에 연락하여 지원을 요청하세요.

**Step 6 - 성능 테스트**

완전히 기능하는 애플리케이션으로 Graviton에서 성능 기준선을 설정해야 합니다. 대부분의 경우 성능 향상을 기대해야 합니다. 기존 x86-64 인스턴스와 비교할 때 두 시스템을 모두 완전히 로드하여 가능한 최대 가격/성능을 결정하는 테스트를 실행하는 것이 좋습니다. 그런 다음 배포를 수행하기 전에 프로덕션 환경에 적합한 로드 수준을 결정하고 구성할 수 있습니다.

중요: 이 저장소에는 [Optimization](optimizing.md) 및 [Performance Runbook](perfrunbook/graviton_perfrunbook.md) 전용 섹션이 있으며, 이 단계에서 따라야 합니다.

이 저장소에서 문서를 읽고 권장 사항을 준수하지 않은 경우 AWS 계정 팀에 연락하거나 [ec2-arm-dev-feedback@amazon.com](mailto:ec2-arm-dev-feedback@amazon.com)로 자세한 내용을 이메일로 보내어 성능 관찰을 지원하세요.


### 인프라 및 배포

**Step 7 - Infrastructure as Code 업데이트**

이제 테스트되고 성능이 뛰어난 애플리케이션을 갖게 되었습니다. Graviton 기반 인스턴스에 대한 지원을 추가하기 위해 인프라를 코드로 업데이트할 때입니다. 여기에는 일반적으로 다중 아키텍처를 지원하기 위해 인스턴스 유형, AMI ID, ASG 구성 요소 업데이트가 포함됩니다([Amazon EC2 ASG support for multiple Launch Templates](https://aws.amazon.com/about-aws/whats-new/2020/11/amazon-ec2-auto-scaling-announces-support-for-multiple-launch-templates-for-auto-scaling-groups/)) 참조). 그리고 마지막으로 배포 또는 재배포를 수행합니다.

**Step 8 - Canary 또는 Blue-Green 배포 수행**

인프라가 Graviton 기반 인스턴스를 지원할 준비가 되면 Canary 또는 Blue-Green 배포를 시작하여 애플리케이션 트래픽의 일부를 Graviton 기반 인스턴스로 리디렉션할 수 있습니다. 이상적으로 초기 테스트는 프로덕션 트래픽 패턴으로 로드 테스트를 위해 개발 환경에서 실행됩니다. 응용 프로그램을 자세히 모니터링하여 예상 동작을 확인합니다. 애플리케이션이 Graviton에서 예상대로 실행되면 전환 전략을 정의하고 실행할 수 있으며 가격 대비 성능 향상의 이점을 누릴 수 있습니다.


### _부록 A - Arm64/Graviton 패키지 찾기_

기억하세요: 소프트웨어를 찾을 때 GCC를 포함한 일부 툴들은 아키텍처를 AArch64라고 부르고 리눅스 커널을 포함한 다른 툴들은 arm64라고 부릅니다. 다양한 저장소에서 패키지를 확인할 때 이러한 서로 다른 명명 규칙을 찾을 수 있으며, 경우에 따라서는 "ARM"일 수도 있습니다.

확인할 수 있는 주요 방법 및 찾을 장소는:

* 선택한 Linux 배포의 패키지 저장소. 예를 들어, 가장 큰 패키지 저장소를 가지고 있는 데비안은 패키지의 98% 이상을 arm64 아키텍처용으로 제작하였습니다.
* 컨테이너 이미지 레지스트리. Amazon ECR은 이제 [public repository](https://docs.aws.amazon.com/AmazonECR/latest/public/public-repositories.html)에서 [arm64 images](https://gallery.ecr.aws/?architectures=ARM+64&page=1)를 검색할 수 있습니다.). DockerHub를 사용하면 특정 아키텍처([예: arm64](https://hub.docker.com/search?type=image&architecture=arm64))를 검색할 수 있습니다.
    * 참고: Arm64 지원을 추가할 때 현재 사용하는 amd64(x86-64) 컨테이너 이미지가 다중 아키텍처 컨테이너 이미지로 전환될 수 있습니다. 즉, 명시적인 Arm64 컨테이너를 찾지 못할 수 있으므로, 프로젝트가 x86-64 및 Arm64용 개별 이미지를 벤딩하도록 선택할 수 있고 다른 프로젝트가 두 아키텍처를 지원하는 다중 아치 이미지를 벤딩하도록 선택할 수 있으므로 두 컨테이너를 모두 확인해야 합니다.
* GitHub의 릴리스 섹션에서 arm64 버전을 확인할 수 있습니다. 그러나 일부 프로젝트는 릴리스 섹션을 사용하지 않거나 소스 아카이브만 릴리스하므로 기본 프로젝트 웹 페이지를 방문하여 다운로드 섹션을 확인해야 할 수도 있습니다. 또한 GitHub 프로젝트에서 "arm64" 또는 "AArch64"를 검색하여 프로젝트에 arm64 코드 기여 또는 문제가 있는지 확인할 수 있습니다. 프로젝트가 현재 arm64용 빌드를 생산하지 않더라도 대부분의 경우 Linux 배포판이나 추가 패키지 저장소(예: [EPEL](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-enable-epel/))를 통해 해당 패키지의 Arm64 버전을 사용할 수 있습니다. [pkgs.org](https://pkgs.org/)과 같은 패키지 검색 도구를 사용하여 패키지를 검색할 수 있습니다.
* 소프트웨어 공급업체의 다운로드 섹션 또는 플랫폼 지원 매트릭스에서 Arm64, AArch64 또는 Graviton에 대한 참조를 찾습니다.

Categories of software with potential issues:

* ISV에서 소싱된 패키지 또는 애플리케이션은 Graviton에서 아직 제공되지 않을 수 있습니다. AWS는 Graviton에 대한 지원을 추가하면서 많은 소프트웨어 파트너와 협력하여 기술 지침을 제공하고 있지만, 일부는 여전히 누락되었거나 지원을 추가하는 중입니다. 일부 ISV 소프트웨어의 목록은 [here](isv.md)에서 확인할 수 있습니다.
* Python 커뮤니티는 Arm64 아키텍처를 위해 컴파일해야 하는 낮은 수준의 언어(예: C/C++)를 사용하여 구축된 많은 모듈을 사용합니다. 현재 Python 패키지 인덱스에서 사전 빌드 바이너리로 사용할 수 없는 모듈을 사용할 수 있습니다. AWS는 가장 인기 있는 모듈을 사용할 수 있도록 오픈 소스 커뮤니티와 적극적으로 협력하고 있습니다. 한편, AWS는 Graviton Getting Starting Guide의 [Python 섹션](python.md#1-installing-python-packages)에서 누락된 패키지에 대한 빌드 시간 종속성을 해결하기 위한 구체적인 지침을 제공합니다.


다른 소프트웨어에서 Arm64를 지원하지 않는 경우 AWS 팀에 알리거나 [ec2-arm-dev-feedback@amazon.com](mailto:ec2-arm-dev-feedback@amazon.com)로 이메일을 보내십시오.
