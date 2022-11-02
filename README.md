# AWS Graviton 테크니컬 가이드

이 저장소는 [AWS Graviton Technical Guide](https://github.com/aws/aws-graviton-getting-started)의 한글 번역본입니다. 이 저장소는 [AWS Graviton 프로세서로 구동되는 Amazon EC2 인스턴스](https://aws.amazon.com/ec2/graviton/)(최신 세대 Graviton3 프로세서 포함)를 사용하는 사용자와 개발자를 위한 기술지침을 제공해드리기 위해 작성되었습니다. Graviton 프로세서의 특정 기능을 다루겠지만, 이 저장소는 일반적으로 Arm 기반 시스템에서 코드를 실행하는 모든 사람에게 유용합니다.

# Contents
* [Graviton으로 전환](#Graviton으로-전환)
* [Graviton 프로세서](#Graviton-프로세서)
* [Graviton 최적화하기](optimizing.md)
* [Arm Advanced SIMD 활용하기](SIMD_and_vectorization.md)
* [Graviton과 관련된 최신 소프트웨어 업데이트](#Graviton과-관련된-최신-소프트웨어-업데이트)
* 프로그래밍 언어별 고려사항
	* [C/C++](c-c++.md)
	* [Go](golang.md)
	* [Java](java.md)
	* [.NET](dotnet.md)
	* [Python](python.md)
	* [Rust](rust.md)
* [컨테이너에서 Graviton 사용](containers.md)
* [Lambda에서 Graviton 사용](#Lambda에서-Graviton-사용)
* [운영체제 지원](os.md)
* [서드파티 소프트웨어 벤더](isv.md)
* [AWS System Manager 또는 Cloud Formation을 사용하여 Graviton용 AMI 찾기 및 관리](amis_cf_sm.md)
* [DPDK, SPDK 그리고 다른 데이터패스 소프트웨어 사용 시](dpdk_spdk.md)
* [TensorFlow](machinelearning/tensorflow.md)
* [알려진 문제점들과 해결방법](#알려진-문제점들과-해결방법)
* [Graviton을 사용할 수 있는 AWS Managed Service들](managed_services.md)
* [Graviton 성능 런북](perfrunbook/graviton_perfrunbook.md)
* [추가 자료](#추가-자료)

# Graviton으로 전환
Graviton을 처음 접하시거나, 워크로드를 식별하는 방법, 프로젝트 전환을 계획하는 방법, AWS Graviton2에서 워크로드를 테스트하는 방법, 마지막으로 운영 환경에 배포하는 방법을 알고 싶다면 [워크로드를 AWS Graviton2 기반 Amazon EC2 인스턴스로 전환할 때 주요 고려 사항](transition-guide.md)을 참조하세요. 

# Graviton 프로세서

|프로세서	|Graviton2	|Graviton3	|
|---	|---	|---	|
|인스턴스 타입	|[M6g/M6gd](https://aws.amazon.com/ec2/instance-types/m6/), [C6g/C6gd/C6gn](https://aws.amazon.com/ec2/instance-types/c6/), [R6g/R6gd](https://aws.amazon.com/ec2/instance-types/r6/), [T4g](https://aws.amazon.com/ec2/instance-types/t4), [X2gd](https://aws.amazon.com/ec2/instance-types/x2/), [G5g](https://aws.amazon.com/ec2/instance-types/g5g/), [Im4gn/Is4gen](https://aws.amazon.com/ec2/instance-types/i4g/)	|[C7g](https://aws.amazon.com/ec2/instance-types/c7g/)	|
|코어	|[Neoverse-N1](https://developer.arm.com/documentation/100616/0301)	|[Neoverse-V1](https://developer.arm.com/documentation/101427/latest/)	|
|Frequency	|2500MHz	|2600MHz	|
|Turbo 지원	|아니오	|아니오	|
|Instruction latencies	|[Instruction Latencies](https://developer.arm.com/documentation/PJDOC-466751330-9707/r4p1/?lang=en)	|[Instruction Latencies](https://developer.arm.com/documentation/pjdoc466751330-9685/latest/)	|
|Interconnect	|CMN-600	|CMN-700	|
|아키텍처 리비전	|ARMv8.2-a	|ARMv8.4-a	|
|추가 기능	|fp16, rcpc, dotprod, crypto	|sve, rng, bf16, int8, crypto	|
|권장 -mcpu flag	|neoverse-n1	|neoverse-512tvb	|
|RNG Instructions	|아니오	|네	|
|SIMD instructions	|2x Neon 128bit vectors	|4x Neon 128bit vectors / 2x SVE 256bit	|
|LSE (atomic mem operations)	|네	|네	|
|Pointer Authentication	|아니오	|네	|
|코어수	|64	|64	|
|L1 캐시 (코어당)	|64KB inst / 64KB data	|64KB inst / 64KB data	|
|L2 캐시 (코어당)	|1MB	|1MB	|
|LLC (shared)	|32MB	|32MB	|
|DRAM	|8x DDR4	|8x DDR5	|
|DDR 암호화	|네	|네	|

# Graviton 최적화하기
일반적인 디버깅 및 프로파일링 정보는 [여기](optimizing.md)를 참조하세요. Graviton에서 성능 최적화 및 디버깅에 대한 자세한 체크리스트는 [성능 런북](perfrunbook/graviton_perfrunbook.md)을 참조하세요.

아키텍처와 시스템에 따라 기능이 다르므로, 기존 아키텍처에서 익숙하게 사용했던 일부 도구들이 AWS Graviton에서는 동일하게 동작하지 않을 수 있습니다. [여기](Monitoring_Tools_on_Graviton.md)에서 해당 유틸리티들을 참조하세요.

# Graviton과 관련된 최신 소프트웨어 업데이트
ARM 소프트웨어 생태계에는 엄청난 양의 개선이 매일 이루어지고 있습니다. 컴파일러 그리고 런타임은 최신 버전을
가능하다면 사용하셔야 합니다. 아래 표에는 많이 사용하는 패키지에 대한 최신 업데이트 내용이 나와 있습니다. 
(혹시 다른 내용이 있다면 알려주세요)

Package | Version | Improvements
--------|:-:|-------------
bazel	| [3.4.1+](https://github.com/bazelbuild/bazel/releases) | Graviton/Arm64를 위한 사전 빌드된 바이너리가 제공됩니다. 설치를 위해서는 [아래](#Linux에서의-Bazel)를 참고하세요. 
ffmpeg  |   5.1+  | ffmpeg 다중 스레드 인코더의 성능과 확장성을 향상시키는 더 나은 NEON 벡터화로 libswscale의 성능이 50% 향상되었습니다. 변경 사항은 FFMPEG 버전 4.3에서 사용할 수 있으며 5.1에서 스케일링 및 모션 추정을 추가로 개선할 수 있습니다. 이 두 가지 모두에 대한 추가 개선 사항은 2022년 8월 현재 아직 출시되지 않은 5.2에서 이용할 수 있습니다.
HAProxy  | 2.4+  | [심각한 버그](https://github.com/haproxy/haproxy/issues/958) 가 수정이 되었습니다. 또한 `CPU=armv81`로 빌드하면 HAProxy 성능이 4배 향상되므로 이 플래그로 코드를 다시 빌드하세요.
mongodb | 4.2.15+ / 4.4.7+ / 5.0.0+ | 특히 내부 JS 엔진의 경우 Graviton 성능이 향상되었습니다. LSE 지원이 [SERVER-56347](https://jira.mongodb.org/browse/SERVER-56347)에서 추가되었습니다.
MySQL   | 8.0.23+ | 컴파일러가 지원하는 경우 -moutline-atomics로 컴파일했을 때 향상된 스핀락 동작을 보입니다.
.NET | [5+](https://dotnet.microsoft.com/download/dotnet/5.0) | .NET 5는 [ARM64의 성능을 크게 향상시켰습니다](https://devblogs.microsoft.com/dotnet/Arm64-performance-in-net-5/). 다음은 관련 [AWS 블로그](https://aws.amazon.com/blogs/compute/powering-net-5-with-aws-graviton2-benchmark-results/)와 몇 가지 성능 결과가 나와 있습니다.
OpenH264 | [2.1.1+](https://github.com/cisco/openh264/releases/tag/v2.1.1) | Graviton/Arm64를 위해 사전 빌드된 Cisco OpenH264 바이너리가 제공됩니다.
PCRE2   | 10.34+  | PCRE의 JIT에 네온 벡터화를 추가하여 첫 번째와 한 쌍의 문자를 일치시킵니다. 이것은 매칭 성능을 최대 8배까지 향상시킬 수 있습니다. 이 라이브러리 버전은 현재 Ubuntu 20.04 및 PHP 8과 함께 제공되고 있습니다.
PHP     | 7.4+    | PHP 7.4에는 성능이 최대 30% 향상되는 여러 가지 성능 향상 기능이 포함되어 있습니다.
pip     | 19.3+   | Graviton Graviton에서 Python wheel 바이너리 설치를 활성화합니다.
PyTorch | 1.7+    | FP32에 대해 Arm64 컴파일, Neon 최적화를 활성화합니다. [소스로부터 설치](https://github.com/aws/aws-graviton-getting-started/blob/master/python.md#41-pytorch). **노트:** *현재 GCC9 이상이 필요합니다. Ubuntu 20.xx 사용을 권장합니다.*
ruby    | 3.0+ | 성능을 최대 40% 향상시키는 arm64 최적화 지원 변경 사항은 Amazon Linux 2, Fedora, and Ubuntu 20.04와 함께 Ruby로 백포팅되었습니다.
zlib    | 1.2.8+  | Graviton2에서 최고의 성능을 얻으려면 [zlib-cloudflare](https://github.com/cloudflare/zlib)를 사용하세요.

# 컨테이너에서 Graviton 사용
Docker, Kubernetes, Amazon ECS, 그리고 Amazon EKS를 Graviton에서 실행할 수 있습니다. Amazon ECR은 multi-arch 컨테이너를 지원합니다. Graviton에서 컨테이너 기반 워크로드를 실행하는 방법에 대한 내용은 [여기](containers.md)를 참조하세요.

# [Lambda에서 Graviton 사용](/aws-lambda/README.md)
[AWS Lambda](https://aws.amazon.com/lambda/)를 사용하면 x86 기반 기능 외에도 Arm 기반 AWS Graviton2 프로세서에서 실행할 새 기능과 기존 기능을 구성할 수 있습니다. 이 프로세서 아키텍처 옵션을 사용하면 가격 성능을 최대 34% 향상시킬 수 있습니다. 기간 요금은 [밀리초 단위](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-1ms-billing-granularity-adds-cost-savings/)로 x86의 현재 가격보다 20% 저렴합니다. 이는 [Provisioned Concurrency](https://aws.amazon.com/blogs/aws/new-provisioned-concurrency-for-lambda-functions/))을 사용할 때의 기간 요금에도 적용됩니다. Compute [Savings Plans](https://aws.amazon.com/blogs/aws/savings-plan-update-save-up-to-17-on-your-lambda-workloads/)는 Graviton2로 구동되는 Lambda 기능을 지원합니다.

[Lambda](/aws-lambda/README.md) 페이지에서는 몇 가지 마이그레이션 고려 사항을 강조하고 Arm/Graviton2를 사용하여 Lambda 기능을 빌드하고 마이그레이션하는 방법을 탐색하는 데 사용할 수 있는 간단한 배포 데모를 제공하고 있습니다.

# 운영체제 지원

Graviton 기반 인스턴스에서 실행할 운영 체제에 대한 자세한 내용은 [여기](os.md)를 참조하세요.

# 알려진 문제점들과 해결방법

## Postgres
[LSE](https://github.com/aws/aws-graviton-getting-started/blob/main/c-c%2B%2B.md#large-system-extensions-lse)를 사용하지 않으면 Postgres 성능이 크게 저하될 수 있습니다.
오늘날 Ubuntu 배포 등에서의 postgres 바이너리는 LSE를 활성화하는 `-moutline-atomics`나 `-march=armv8.2-a`로 빌드되지 않는다는 문제가 있습니다. 참고: Amazon RDS for PostgreSQL은 이에 영향을 받지 않습니다.

2021년 11월 PostgresSQL은 `-moutline-atomics`로 최적화된 우분투 20.04 패키지를 배포하기 시작했습니다.
Ubuntu 20.04의 경우 Ubuntu Focol에서 배포한 패키지 대신 PostgreSQL PPA를 사용하는 것이 좋습니다.
PostgreSQL PPA를 설치하기 위해 [Postgre 설정 지침](https://www.postgresql.org/download/linux/ubuntu/)을 따라하세요.

## 일부 리눅스 배포판에 Python 설치방법
일부 리눅스 배포판은 Graviton으로 출시된 Python wheel 패키지를 설치하기에는 오래 \(<19.3\)되었습니다. 이 문제를 해결하려면 다음을 사용하여 Pip 설치를 업그레이드하는 것이 좋습니다:
```
sudo python3 -m pip install --upgrade pip
```

## Linux에서의 Bazel
이제 [Bazel 빌드 도구](https://www.bazel.build/)에서 arm64용 사전 빌드 바이너리를 릴리스합니다. 2020년 10월 기준으로 커스텀 데비안 레포에서는 사용이 불가능하고, 바젤은 공식적으로 RPM을 제공하지는 않습니다. 대신 "bazel" 명령과 [bazel을 최신 상태로 유지](https://github.com/bazelbuild/bazelisk/blob/master/README.md)을 대체하는 [bazelisk instra](build/build/build/master/install-bazelisk.bazel.bazel.bazel.build)를 사용하는 것이 좋습니다.

아래는 2020년 10월 기준 [bazelisk의 최신 Arm 바이너리 릴리스](https://github.com/bazelbuild/bazelisk/releases/latest)를 사용한 예입니다:
```
wget https://github.com/bazelbuild/bazelisk/releases/download/v1.7.1/bazelisk-linux-arm64
chmod +x bazelisk-linux-arm64
sudo mv bazelisk-linux-arm64 /usr/local/bin/bazel
bazel
```

Bazelisk 자체는 Bazel을 계속 업데이트하는 것이 유일한 목적이기 때문에 추가 업데이트가 필요하지 않습니다.

## Linux에서의 zlib
리눅스 배포판은 일반적으로 최적화 없이 원본 zlib을 사용합니다. zlib-cloudflare는 Arm과 x86에서 더 좋고 빠른 압축을 제공하기 위해 추가되었습니다. zlib-cloudflare를 사용하려면:
```
git clone https://github.com/cloudflare/zlib.git
cd zlib
./configure --prefix=$HOME
make
make install
```
/etc/ld.so.conf의 $HOME/lib에 있는 lib에 대한 전체 경로가 있는지 확인하고 ldconfig를 실행하세요.

시스템 zlib에 동적으로 연결된 OpenJDK 사용자의 경우 새로 빌드된 zlib-cloudflare 버전이 있는 디렉토리를 가리키도록 LD_LIBRARY_PATH를 설정하거나 해당 라이브러리를 LD_PRELOAD로 로드할 수 있습니다.

JDK가 동적으로 링크되는 libz를 확인할 수 있습니다:
```
$ ldd /Java/jdk-11.0.8/lib/libzip.so | grep libz
libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007ffff7783000)
```

현재 Amazon Corretto 사용자는 zlib-cloudflare에 대해 링크를 걸 수 없습니다.

# 추가 자료

 * [AWS Graviton2](https://aws.amazon.com/ec2/graviton/)
 * [Neoverse N1 소프트웨어 최적화 가이드](https://documentation-service.arm.com/static/5f05e93dcafe527e86f61acd)
 * [Armv8 레퍼런스 매뉴얼](https://documentation-service.arm.com/static/60119835773bb020e3de6fee)
 * [패키지 저장소 검색 도구](https://pkgs.org/)

**피드백** ec2-arm-dev-feedback@amazon.com

**한글 번역에 대한 피드백** sanghwa@amazon.com

**업데이트: 2022-11-04**