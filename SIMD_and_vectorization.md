# Arm Advanced SIMD 활용하기

## 배경

Arm-v8 아키텍처는 넓은 레지스터를 활용할 수 있는 많은 응용 프로그램의 성능을 향상시키는 데 도움이 되는 고급 SIMD 명령(NEON)을 포함합니다.

이미 많은 응용 프로그램과 라이브러리가 Arm의 Advanced-SIMD를 활용하고 있지만, 이 가이드는 새로운 코드나 라이브러리를 작성하는 개발자를 위해 작성되었습니다. 우리는 컴파일러 자동 벡터화를 통해서든 내재적 요소를 쓰든 간에 이러한 명령어들을 활용할 수 있는 다양한 방법을 안내할 것입니다.

나중에 우리는 개발자들이 다른 코어를 지원하는 하나의 바이너리를 만들 수 있도록 런타임에 어떤 명령어가 사용 가능한지를 감지하는 이식성 있는 코드를 만드는 방법에 대해 설명할 것입니다.

## 컴파일러 기반 자동 벡터화

컴파일러는 개발자의 명시적 지침이나 특정 코딩 스타일 없이 SIMD 명령어를 활용할 수 있도록 계속 개선됩니다.

일반적으로 GCC 9는 자동 벡터화를 잘 지원하는 반면, GCC 10은 대부분의 경우 GCC 9보다 인상적인 개선을 보였습니다.

*-fopt-info-vec-missed*를 사용하여 컴파일하는 것은 벡터화되지 않은 루프를 확인하는 좋은 방법입니다.

### 큰 코드 변경 없이 자동 벡터화 개선하기

다음 예제는 Graviton 2에서 실행되었으며 Ubuntu 20.04와 gcc 9.3이 함께 실행되었습니다. 서버 버전과 컴파일러 버전의 조합이 다를 경우 다른 결과가 나타날 수 있습니다.

시작 코드는 다음과 같습니다:
```
  1 // test.c 
...
  5 float   a[1024*1024];
  6 float   b[1024*1024];
  7 float   c[1024*1024];
.....
 37 for (j=0; j<128;j++) { // outer loop, not expected to be vectorized
 38   for (i=0; i<n ; i+=1){ // inner loop, ideal for vectorization
 39         c[i]=a[i]*b[i]+j;
 40   }
 41 }
```

컴파일링 하면:

```
$ gcc test.c -fopt-info-vec-missed -O3
test.c:37:1: missed: couldn't vectorize loop
test.c:39:8: missed: not vectorized: complicated access pattern.
test.c:38:1: missed: couldn't vectorize loop
test.c:39:6: missed: not vectorized: complicated access pattern.
```

37번 라인은 외부 루프이며 벡터화될 것으로 예상되지 않습니다.
그러나 내부 루프는 벡터화의 주요 후보이지만, 이 경우에는 수행되지 않았습니다.

내부 루프가 항상 128비트 SIMD에 정렬되도록 하기 위해 코드를 약간 변경해도 gcc 9는 이를 자동 벡터화하기에 충분할 것입니다.

```
  1 // test.c 
...
  5 float   a[1024*1024];
  6 float   b[1024*1024];
  7 float   c[1024*1024];
...
 19 #if(__aarch64__)
 20 #define ARM_NEON_WIDTH  128
 21 #define VF32    ARM_NEON_WIDTH/32
 22 #else
 23 #define VF32    1
 33 #endif
...
 37 for (j=0; j<128;j++) { // outer loop, not expected to be vectorized
 38   for (i=0; i<( n - n%VF32 ); i+=1){ // forcing inner loop to multiples of 4 iterations
 39         c[i]=a[i]*b[i]+j;
 40   }
 41   // epilog loop
 42   if (n%VF32)
 43         for ( ; i < n; i++) 
 44                 c[i] = a[i] * b[i]+j;
 45 }
```

위의 코드는 내부 루프가 4의 배수 (128비트 SIMD / Float당 32비트)를 반복하도록 강제합니다. 결과는:

```
$ gcc test.c -fopt-info-vec-missed -O3
test.c:37:1: missed: couldn't vectorize loop
test.c:37:1: missed: not vectorized: loop nest containing two or more consecutive inner loops cannot be vectorized
```
그리고 외부 루프는 여전히 예상대로 벡터화되지 않았지만 내부 루프는 벡터화되었습니다 (그리고 3~4배 더 빠릅니다).

다시 말하지만 컴파일러 기능이 시간이 지남에 따라 향상됨에 따라 이런 기술은 필요 없을 수도 있습니다. 하지만 애플리케이션이 gcc9 이상으로 구축되는 한 이렇게 하는 것은 좋은 습관입니다.


## intrinsics 사용하기

이식성 있는 코드를 빌드하는 한 가지 방법은 Arm에 의해 정의되고 GCC와 Clang에 의해 구현된 intrinsic 명령어를 사용하는 것입니다.

명령은 *arm_neon에 정의되어 있습니다.h*.

Arm CPU와 컴파일러를 감지(컴파일 시)하는 휴대용 코드는 다음과 같습니다:

```
#if (defined(__clang__) || (defined(__GCC__)) && (defined(__ARM_NEON__) || defined(__aarch64__))
/* compatible compiler, targeting arm neon */
#include <arm_neon.h>
#include <arm_acle.h>
#endif
```

## 지원되는 SIMD 명령의 런타임 탐지

Arm 아키텍처 버전은 특정 명령 지원을 요구하지만 특정 명령어는 아키텍처의 특정 버전에 대해 선택 사항입니다.

예를 들어 Arm-v8.4 아키텍처를 준수하는 CPU 코어는 dot-product을 지원해야 하지만, Arm-v8.2 및 Arm-v8.3에서는 dot-product가 선택 사항입니다.
Graviton2는 Arm-v8.2와 호환되지만 CRC 및 dot-product 명령을 모두 지원합니다.

개발자는 런타임에 지원되는 명령어를 탐지할 수 있는 응용 프로그램 또는 라이브러리를 구축하고자 하는 경우 다음 예를 따를 수 있습니다.:

```
#include<sys/auxv.h>
......
  uint64_t hwcaps = getauxval(AT_HWCAP);
  has_crc_feature = hwcaps & HWCAP_CRC32 ? true : false;
  has_lse_feature = hwcaps & HWCAP_ATOMICS ? true : false;
  has_fp16_feature = hwcaps & HWCAP_FPHP ? true : false;
  has_dotprod_feature = hwcaps & HWCAP_ASIMDDP ? true : false;
  has_sve_feature = hwcaps & HWCAP_SVE ? true : false;
```

arm64 하드웨어 기능의 전체 목록은 [glibc header file](https://github.com/bminor/glibc/blob/master/sysdeps/unix/sysv/linux/aarch64/bits/hwcap.h) 및 [Linux kernel](https://github.com/torvalds/linux/blob/master/arch/arm64/include/asm/hwcap.h)에 정의되어 있습니다.

## SSE/AVX intrinsics 코드를 NEON 으로 포팅하기

### Detecting arm64 systems

arm64에서 `error: unrecognized command-lineoption '-mse2'` 또는 `-mavx`, `-msse3` 등으로 프로젝트가 생성되지 않을 수 있습니다. 이러한 컴파일러 플래그는 x86 벡터 명령어를 가능하게 합니다. 이 오류가 있다는 것은 빌드 시스템이 대상 시스템의 탐지를 놓칠 수 있다는 것을 의미하며, arm64용으로 컴파일할 때 x86 대상 기능 컴파일러 플래그를 계속 사용한다는 것을 의미합니다. arm64 시스템을 감지하기 위해 빌드 시스템은 다음을 사용할 수 있습니다:
```
# (test $(uname -m) = "aarch64" && echo "arm64 system") || echo "other system"
```

arm64 시스템을 감지하는 또 다른 방법은 C 프로그램의 반환 값을 컴파일, 실행 및 확인하는 것입니다:
```
# cat << EOF > check-arm64.c
int main () {
#ifdef __aarch64__
  return 0;
#else
  return 1;
#endif
}
EOF

# gcc check-arm64.c -o check-arm64
# (./check-arm64 && echo "arm64 system") || echo "other system"
```

### Translating x86 intrinsics to NEON

프로그램에 x64 intrinsic 함수의 코드가 포함되어 있는 경우, 다음 절차는 Arm에서 작동하는 프로그램을 빠르게 획득하고 Graviton 프로세서에서 실행되는 프로그램의 성능을 평가하고 hot path를 프로파일링하며 hot path에서 코드의 품질을 향상시키는 데 도움이 될 수 있습니다. 

Arm에서 프로토타입을 신속하게 실행하려면 [SIMDe (SIMD everywhere)](https://github.com/simd-everywhere/simde)을 사용할 수 있습니다.

예를 들어, AVX2 intrinsics를 사용하는 코드를 Graviton에 포팅하는 경우 개발자는 다음 코드를 추가할 수 있습니다:
```
#define SIMDE_ENABLE_NATIVE_ALIASES
#include "simde/x86/avx2.h"
```

SIMDe는 성능 중요 코드를 Arm에 포트하기 위한 빠른 시작 지점을 제공합니다. 이는 Arm 작업 프로그램을 얻는 데 필요한 시간을 단축시켜 프로파일을 추출하고 코드에서 핫 경로를 식별하는 데 사용할 수 있습니다. 프로파일이 설정되면 일반 변환의 오버헤드를 피하기 위해 NEON 내재적 요소를 사용하여 hot path를 직접 다시 작성할 수 있습니다.

## 추가 자료

 * [Neon Intrinsics](https://developer.arm.com/architectures/instruction-sets/intrinsics/)
 * [Coding for Neon](https://developer.arm.com/documentation/102159/latest/)
 * [Neon Programmer's Guide for Armv8-A](https://developer.arm.com/architectures/instruction-sets/simd-isas/neon/neon-programmers-guide-for-armv8-a)
