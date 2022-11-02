# Graviton 에서의 C/C++

### Arm 아키텍처별 기능 활성화

최적의 프로세서 기능으로 코드를 작성하려면 다음을 사용하세요. Graviton3용으로 컴파일된 코드는 Graviton3에서만 실행되며 Graviton2에서는 실행되지 않으므로 Graviton2 프로세서 기능을 사용할 것을 권장합니다. arm64에서 `-mcpu=`는 적절한 아키텍처를 지정하는 동시에 튜닝하는 역할을 하며 특정 CPU를 위해 빌드하는 경우에는 일반적으로 `-march` 보다 `-mcpu=`를 사용하는 것이 좋습니다.

CPU       | Flag    | GCC 버전      | LLVM 버전
----------|---------|-------------------|-------------
Graviton2 | `-mcpu=neoverse-n1`\* | GCC-9^ | Clang/LLVM 10+
Graviton3 | `-mcpu=neoverse-512tvb`% | GCC 11+ | Clang/LLVM 14+

^ Amazon Linux2 GCC-7에 있음.

\* GCC-9 이상이 필요. 그렇지 않으면 `-mcpu=cortex-a72`를 사용하는 것이 좋음

% 만약 컴파일러가 `neoverse-512tvb`를 지원하지 않는다면, Graviton2 튜닝을 사용

### 컴파일러

신규 컴파일러는 Graviton 프로세서에 대한 더 나은 지원과 최적화를 제공합니다. 우리는 아마존 리눅스 2 시스템의 컴파일러 gcc-7 대신 gcc-10을 사용할 때 Graviton2에서 15% 더 나은 성능을 보았습니다. 가능한 경우 시스템에서 사용할 수 있는 최신 컴파일러 버전을 사용하세요. 아래 표는 리눅스 배포판에서 사용할 수 있는 GCC 및 LLVM 컴파일러 버전을 보여줍니다. 별표가 있는 버전은 기본 시스템 컴파일러를 표시합니다.

배포판            | GCC                  | Clang/LLVM
----------------|----------------------|-------------
Amazon Linux 2  | 7*, 10               | 7, 11*
Ubuntu 22.04    | 9, 10, 11*, 12       | 11, 12, 13, 14*
Ubuntu 20.04    | 7, 8, 9*, 10         | 6, 7, 8, 9, 10, 11, 12
Ubuntu 18.04    | 4.8, 5, 6, 7*, 8     | 3.9, 4, 5, 6, 7, 8, 9, 10
Debian10        | 7, 8*                | 6, 7, 8
Red Hat EL8     | 8*, 9, 10            | 10
SUSE Linux ES15 | 7*, 9, 10            | 7


### Large-System Extensions (LSE)

Graviton2와 Graviton3 프로세서는 vArmv8.1에서 처음 도입된 LSE(Large-System Extensions)를 지원한다. load/store exclusives 대신 LSE를 사용할 경우 최대 몇 배까지 성능이 개선될 수 있습니다.

POSIX 스레드 라이브러리에는 LSE atomic 명령이 필요합니다. LSE는 Lock 및 스레드 동기화 루틴에 중요합니다. 다음 시스템은 LSE 지침으로 컴파일된 libc를 배포합니다:
- Amazon Linux 2,
- Amazon Linux 2022,
- Ubuntu 18.04 (needs `apt install libc6-lse`),
- Ubuntu 20.04,
- Ubuntu 22.04.

컴파일러는 atomic 연산을 사용하는 응용 프로그램에 대한 LSE 명령을 생성해야 합니다. 예를 들어, PostgreSQL와 같은 데이터베이스의 코드는 atomic constructs를 포함하고 있으며, c++11 코드와 std::atomic 문은 atomic 연산으로 변환됩니다. GCC의 `-march=armv8.2-a` 플래그는 LSE를 포함해 Graviton2가 지원하는 모든 명령을 가능하게 합니다. LSE 명령이 생성되었는지 확인하려면 `objdump` 명령줄 유틸리티의 출력에 LSE 명령이 포함되어 있어야 합니다:
```
$ objdump -d app | grep -i 'cas\|casp\|swp\|ldadd\|stadd\|ldclr\|stclr\|ldeor\|steor\|ldset\|stset\|ldsmax\|stsmax\|ldsmin\|stsmin\|ldumax\|stumax\|ldumin\|stumin' | wc -l
```
응용 프로그램 바이너리에 load and store exclusives가 포함되어 있는지 확인하려면 다음과 같이 하세요:
```
$ objdump -d app | grep -i 'ldxr\|ldaxr\|stxr\|stlxr' | wc -l
```

GCC의 `-moutline-atomics` 플래그는 Graviton과 Graviton2 모두에서 실행되는 바이너리를 생성합니다. 동일한 바이너리로 두 플랫폼을 모두 지원하려면 로드 하나와 브랜치 하나라는 약간의 추가 비용이 듭니다. 응용 프로그램이 `-moutline-atomics`로 컴파일되었는지 확인하기 위해 `nm` 명령줄 유틸리티는 응용 프로그램 바이너리에 함수 이름과 전역 변수를 표시합니다. GCC가 LSE 하드웨어 기능을 확인하는 데 사용하는 부울 변수는 `_aarch64_have_lse_atomics`이며 기호 목록에 표시되어야 합니다:
```
$ nm app | grep __aarch64_have_lse_atomics | wc -l
# the output should be 1 if app has been compiled with -moutline-atomics
```

GCC 10.1+는 아마존 리눅스 2에서 사용하는 GCC 7 버전과 마찬가지로 기본적으로 `-moutline-atomics`를 활성화합니다.

### SSE/AVX intrinsic을 NEON으로 포팅

프로그램에 x64 intrinsic 코드가 포함되어 있는 경우, 다음 절차는 Arm에서 작동하는 프로그램을 빠르게 만들고 Graviton 프로세서에서 실행되는 프로그램의 성능을 평가하고 hot path를 프로파일링하며 hot path에서 코드의 품질을 향상시키는 데 도움이 될 수 있습니다.

Arm에서 동작하는 프로토타입을 빠르게 얻고자 한다면 x64 intrinsic을 NEON으로 변환하는 https://github.com/DLTcollab/sse2neon 을 사용할 수 있습니다. sse2neon은 성능 중요 코드를 Arm으로 포팅하는 빠른 시작점을 제공합니다. 이는 Arm 작업 프로그램을 얻는 데 필요한 시간을 단축시켜 프로파일을 추출하고 코드에서 hot path를 식별하는 데 사용할 수 있다. `sse2neon.h`는 `xmmintrin.h`과 같은 표준 x64에 의해 제공되는 헤더는 몇 가지 기능을 포함하고 있는데, x64 intrinsic과 동일한 사용법을 가지고 있습니다. 일단 프로파일이 설정되면, hot path는 일반 sse2neon 변환의 오버헤드를 피하기 위해 NEON intrinsic으로 직접 다시 작성할 수 있습니다.

### Signed 과 Unsigned char
C 표준은 char의 부호를 명시하지 않습니다. x86 문자에서는 기본적으로 signed이고 Arm에서는 기본적으로 unsigned입니다. 이는 (예: `uint8_t` 및 `int8_t`)을 명시적으로 지정하거나 `-fsigned-char`로 컴파일하는 표준 int 유형을 사용하여 해결할 수 있습니다.

### Graviton2 Arm 명령어을 사용하여 기계 학습 속도 향상

Graviton2 프로세서는 기계 학습(양자화) 추론 작업 부하에 일반적으로 사용되는[Arm dot-product instructions](https://community.arm.com/developer/tools-software/tools/b/tools-software-ides-blog/posts/exploring-the-arm-dot-product-instructions)을 활성화하고 [Half precision floating point - \_float16](https://developer.arm.com/documentation/100067/0612/Other-Compiler-specific-Features/Half-precision-floating-point-intrinsics)를 활성화함으로써 부동소수점은 초당 작업 수를 두 배로 증가시켜 단일 정밀 부동소수점에 비해 메모리 풋프린트를 줄이면서도 큰 dynamic range를 가져 성능과 전력을 최적화하고 기계 학습에 최적화되었습니다.

### SVE 사용

확장 가능한 벡터 확장(SVE)에는 SVE(GCC 11+, LLVM 14+)로 자동 벡터화할 수 있는 충분히 새로운 툴 체인과 SVE를 지원하는 4.15+ 커널이 모두 필요합니다. 한 가지 주목할 만한 예외는 4.14 커널이 있는 Amazon Linux 2가 SVE를 지원하지 않는다는 것입니다. 5.4+ AL2 커널로 업그레이드하세요.

### Arm 명령어 세트를 사용하여 공통 코드 시퀀스 속도 향상
Arm 명령 집합에는 공통 코드 시퀀스의 속도를 높이는 데 사용할 수 있는 명령이 포함되어 있습니다. 아래 표에는 일반적인 작업 및 코드 시퀀스에 대한 링크가 나와 있습니다.:

오퍼레이션 | 설명
----------|------------
[crc](sample-code/crc.c) | Graviton 프로세서는 이더넷, 미디어 및 압축에서 사용되는 CRC32와 파일 시스템에서 사용되는 CRC32C(Castagnoli)를 모두 가속하기 위한 명령을 지원합니다.
