# Graviton 최적화하기

## 디버깅 문제

잘못된 코드는 기존 시스템에서 잘 작동하지만 새 컴파일러를 사용할 때 잘못된 결과를 생성할 수 있습니다. 이것은
언어에서 정의되지 않은 동작(예: C/C++에서 char가 signed라고 가정하거나 signed integer overflow 
동작)에 의존하거나 공격적인 컴파일러 최적화 또는 잘못된 순서에 의해 노출되는 메모리 관리 버그를 포함하고 있기
때문일 수 있습니다. 아래는 내부 서비스를 최신 컴파일러 및 Graviton2로 마이그레이션하는 동안 문제를 찾기
위해 사용한 몇 가지 기술과 도구입니다.

### Sanitizer 사용하기
컴파일러는 x86 시스템과 비교하여 Graviton에서 코드와 레이아웃 데이터를 약간 다르게 생성할 수 있으며 
이는 이전에 숨겨져 있던 잠재적인 메모리 버그를 노출시킬 수 있습니다. GCC에서 이러한 버그를 찾는 가장 
쉬운 방법은 표준 컴파일러 플래그에 아래를 추가하여 메모리 Sanitizer를 사용하여 컴파일하는 것입니다:

```
    CFLAGS += -fsanitize=address -fsanitize=undefined
    LDFLAGS += -fsanitize=address  -fsanitize=undefined
```

그런 다음 결과 바이너리를 실행하면, Sanitizer에 의해 감지된 버그가 프로그램을 즉시 종료하고 유용한 스택 
추적 및 기타 정보를 출력합니다.

### 메모리 접근 순서 문제
Arm은 POWER 및 다른 현대 아키텍처와 유사하게 weakly ordered 구조를 가지고 있습니다. 반면 x86은 
total-store-ordering (TSO)의 변형 구조를 가지고 있습니다. TSO에 의존하는 코드는 메모리 참조를 
적절하게 정렬하는 방어 로직이 부족할 수 있습니다. Graviton과 Graviton2를 포함한 Armv8 기반 시스템은 
[weakly ordered multi-copy-atomic](https://www.cl.cam.ac.uk/~pes20/armv8-mca/armv8-mca-draft.pdf)
으로 동작합니다.

TSO는 쓰기와 함께 읽기가 불규칙하게 일어나도록 하고 프로세서가 다른 사람들에게 보이기 전에 자신의 쓰기를 
관찰할 수 있도록 하는 반면, Armv8 메모리 모델은 성능과 전력 효율성을 더 증대합니다. pthread mutexes에 의존하는
코드나 C++, Java 또는 다른 언어에서 발견되는 잠금 추상화는 특별한 차이는 없습니다.
잠금 없는 데이터 구조를 직접 구현했거나 자체 동기화 프리미티브를 구현하는 코드는 메모리 트랜잭션을 올바르게 
정렬하기 위해 적절한 내재적 요소와 장벽을 사용해야 합니다. 메모리 접근순서와 관련된 문제가 발생하면 언제든지 이 
[GitHub repo](https://github.com/aws/aws-graviton-getting-started)에서 이슈를 오픈하세요. 
AWS 전문가 중 한 명이 귀하에게 연락드릴 것입니다.

### 아키텍처별 최적화
코드는 아키텍처별 최적화 요소를 가질 수 있습니다. 코드는 여러 가지 형태로 나타나는데:
코드는 [CRC](https://github.com/php/php-src/commit/2a535a9707c89502df8bc0bd785f2e9192929422)에 대한 특정 지침을 사용하여 어셈블리에서 최적화됩니다.
다른 경우 특정 [기능](https://github.com/lz4/lz4/commit/605d811e6cc94736dd609c644404dd24c013fd6f)이 특정 아키텍처에 대해 잘 동작하는 것을 보여줍니다.
Arm에 대한 최적화가 누락되었는지 확인하기 위한 방법으로 `__x86_64__` `ifdef` 등을 grep 하여 해당 Arm 코드가 있는지 확인하는 것입니다. 그렇지 않다면
해당 코드는 개선이 필요할 수 있습니다. 필요하다면 [여기](https://github.com/aws/aws-graviton-getting-started)에 이슈를 열어서 내용을 제안해주세요.

### 잠금/동기화가 많은 워크로드
Graviton2는 Arm Large Scale Extensions(LSE)을 지원합니다. LSE 기반 Lock 및 동기화는 코어 수가 많은 경합이 심한 Lock(예: Graviton2의 64)에서 훨씬 더 빠릅니다. Lock이 경합이 심한 워크로드의 경우 `-march=armv8.2-a`로 컴파일하면 LSE 기반 Atomic이 활성화되고 성능이 크게 향상될 수 있습니다. 그러나 이렇게 하면 AWS Graviton 기반 EC2A1 인스턴스와 같은 Arm v8.0 시스템에서 코드가 실행되지 않습니다. GCC 10 이상에서는 `-moutline-atomics` 옵션이 인라인 Atomic을 사용하지 않고 런타임에 사용할 올바른 Atomic 유형을 감지합니다. 이는 `-march=armv8.2-a`보다 성능이 다소 떨어지지만 하위 호환성은 유지됩니다.

### 네트워크 집약적인 워크로드
일부 부하에서는 Graviton2의 패킷 처리 능력은 다른 플랫폼보다 빠르고 지연 시간이 짧아서 리눅스 커널의 
자연스러운 "합병" 기능을 줄이고 인터럽트 속도를 증가시킵니다. 워크로드에 따라 적응형 RX 인터럽트를 사용하는 것이
합리적일 수 있습니다.
(예: `ethtool -C <interface> adaptive-rx on`).

## 코드 프로파일링
원하는 성능을 얻지 못하는 경우 시스템에서 어떤 일이 일어나고 있는지 이해하는 가장 좋은 방법 중 하나는 실행 
프로파일을 비교하고 CPU가 어디에 시간을 소비하는지 이해하는 것입니다. 이 경우 최적화될 수 있는 핫 기능을 자주
가리킵니다. 클러치는 성능이 우수한 시스템과 실행 시간의 상대적 차이를 확인할 수 없는 시스템 간의 프로필을 비교하는
것입니다. 조언이나 도움이 필요하면 언제든지 이 [GitHub repo](https://github.com/aws/aws-graviton-getting-started)
에서 이슈를 열어보세요.

리눅스 성능 도구 설치:
```bash
# Amazon Linux 2
sudo yum install perf

# Ubuntu
sudo apt-get install linux-tools-$(uname -r)
```

프로파일 레코드:
```
# If the program is run interactively
$ sudo perf record -g -F99 -o perf.data ./your_program

# If the program is a service, sample all cpus (-a) and run for 60 seconds while the system is loaded
$  sudo perf record -ag -F99 -o perf.data  sleep 60
```

프로파일 확인:
```
$ perf report
```

시각적으로 더 잘 표현해줄 수 있는 Flame이란 도구도 있습니다:
```
git clone https://github.com/brendangregg/FlameGraph.git
perf script -i perf.data | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > flamegraph.svg
```

예를 들어, 2020년 3월에 성능 향상을 위해
[ffmpeg](http://ffmpeg.org/pipermail/ffmpeg-devel/2019-November/253385.html)에 패치를 
적용했습니다. C5와 M6g의 실행 시간을 비교한 결과 함수 "ff_hscale_8_to_15_neon"이 즉시 Outlier로 발견되었습니다.
일단 특이값으로 식별이 되면 우리는 이 함수를 개선하는 데 집중할 수 있게 됩니다.

```
C5.4XL	                        M6g.4XL
19.89% dv_encode_video_segment	19.57% ff_hscale_8_to_15_neon
11.21% decode_significance_x86	18.02% get_cabac
8.68% get_cabac	                15.08% dv_encode_video_segment
8.43% ff_h264_decode_mb_cabac	5.85% ff_jpeg_fdct_islow_8
8.05% ff_hscale8to15_X4_ssse3	5.01% ff_yuv2planeX_8_neon
```