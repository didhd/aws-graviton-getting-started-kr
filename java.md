# Graviton 에서의 Java

자바는 범용 프로그래밍 언어입니다. 컴파일된 자바 코드는 재컴파일 없이 자바를 지원하는 모든 플랫폼에서 실행될 수 있습니다. 자바 애플리케이션들은 일반적으로 기본 컴퓨터 아키텍처에 관계 없이 모든 자바 가상 머신(JVM)에서 실행될 수 있는 바이트 코드로 컴파일됩니다. _[위키피디아](https://en.wikipedia.org/wiki/Java_(programming_language))_

자바는 arm을 잘 지원하며 일반적으로 arm64에서 즉시 사용할 수 있는 성능을 제공합니다. [Amazon Corretto](https://aws.amazon.com/corretto/)는 OpenJDK(OpenJava Development Kit)의 무료, 멀티 플랫폼, 프로덕션-레디 배포판으로 Graviton-powered 인스턴스를 지원합니다. Java 8은 Arm 프로세서에서 완전히 지원되지만 일부 고객은 Java 11로 전환하기 전까지 Graviton의 완전한 성능 혜택을 얻을 수 없습니다.

이 페이지에는 Graviton에서 Java 애플리케이션을 빌드하고 튜닝하는 것에 대한 구체적인 세부 정보가 포함되어 있습니다.

### Java 버전
arm64용 JDK 바이너리는 다양한 소스에서 사용할 수 있습니다. [Amazon Corretto](https://aws.amazon.com/corretto/)은 Graviton 프로세서에서 실행되는 Java 워크로드의 성능을 지속적으로 개선하고 있으며, JDK를 사용할 수 있는 경우 Corretto를 사용하는 것이 좋습니다. Corretto는 AWS의 성능 향상에 가장 빠르게 액세스할 수 있는 방법을 제공하기 때문입니다.

2020년 10월 이후 출시된 Corretto의 버전은 JVM 내에서 가장 최적의 Atomic 연산을 사용하도록 제작되었습니다: Corretto 11(모든 버전), Correto8(Amazon Linux 2에만 해당). 이로 인해 일부 워크로드에서 GC 시간이 단축되고 Apache Kafka와 같은 네트워크 집약적인 워크로드에서 경합이 발생하지 않습니다.

Corretto 11(>=11.0.12) 버전에는 가벼운 수준에서 중간 정도 수준의 잠금 경합이 있는 워크로드의 성능을 향상시키기 위한 추가 향상이 있습니다: JVM 내부의 스핀 잠금 동작 개선, Graviton의 `Thread.onSpinWait()` 구현 개선.

### Java JVM 옵션
JVM을 제어하고 성능을 향상시킬 수 있는 다양한 옵션이 있습니다.

- 세 개의 플래그 `-XX:-TieredCompilation -XX:ReservedCodeCacheSize=64M -XX:InitialCodeCacheSize=64M` 는 일부 Java 작업 부하에서 큰(1.5배) 성능 향상을 보였습니다. 이 플래그들은 계층적 컴파일을 없애고 Graviton 코어가 분기를 더 잘 예측할 수 있도록 코드 캐시의 크기를 제한합니다. 이러한 기능은 일부 워크로드에는 유용하지만 다른 워크로드에는 아닐 수 있으므로 사용 했을 때와 아닐 때를 테스트하는 것이 중요합니다.

- TLS에서 사용하는 암호화 알고리즘 AES/GCM은 Graviton에 최적화되어 있습니다. Graviton2에서 GCM 암호화/암호화 [성능이 3.5배에서 5배 향상되었습니다](https://github.com/aws/aws-graviton-getting-started/issues/110#issuecomment-989442948). 최적화는 Corretto 및 OpenJDK 18 이상에서 기본적으로 실행됩니다. 최적화는 Corretto 및 OpenJDK 11 및 17로 백포팅되었으며 `-XX:+UnlockDiagnosticVMOptions -XX:+UseAESCTRIntrinsics`를 사용합니다.

### Java 스택 사이즈
Java 스레드의 기본 스택 크기(즉, `ThreadStackSize`)는 aarch64에서 2mb입니다(x86_64에서는 1mb). 다음을 사용하여 기본값을 확인할 수 있습니다:
```
$ java -XX:+PrintFlagsFinal -version | grep ThreadStackSize
     intx CompilerThreadStackSize = 2048  {pd product} {default}
     intx ThreadStackSize         = 2048  {pd product} {default}
     intx VMThreadStackSize       = 2048  {pd product} {default}
```
명령줄에서 `-XX:ThreadStackSize=<kbytes>` 또는 `-Xss<bytes>` 를 사용하여 기본값은 쉽게 변경할 수 있습니다. 주의: `-XX:ThreadStackSize`는 인수를 킬로바이트로 해석하는 반면 `-Xss`는 바이트로 해석합니다. 그래서 `-XX:ThreadStackSize=1024` 와 `-Xss1m` 모두 Java 스레드의 스택 크기를 1MB로 설정합니다:
```
$ java -Xss1m -XX:+PrintFlagsFinal -version | grep ThreadStackSize
     intx CompilerThreadStackSize                  = 2048                                   {pd product} {default}
     intx ThreadStackSize                          = 1024                                   {pd product} {command line}
     intx VMThreadStackSize                        = 2048                                   {pd product} {default}
```

일반적으로 스레드 스택은 커짐에 따라 느리게 커밋되기 때문에 기본값을 변경할 필요가 없습니다. 따라서 기본값이 무엇이든 스레드는 항상 실제 사용하는 만큼의 스택만 커밋합니다(페이지 크기 세분화). 그러나 시스템에서 [Transparent Huge Pages](https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html) (THP)가 기본적으로 켜져 있는 경우 이 규칙에는 한 가지 예외가 있습니다. 이 경우 2MB의 THP 페이지 크기는 aarch64의 2MB 기본 스택 크기와 정확히 일치하며 대부분의 스택은 2MB의 큰 페이지 하나로 백업됩니다. 즉, 스택은 처음부터 메모리에 완전히 커밋됩니다. 수백 개 또는 수천 개의 스레드를 사용하는 경우 메모리 오버헤드가 상당할 수 있습니다.

이 문제를 완화하기 위해 명령줄에서 스택 크기를 수동으로 변경하거나(위에서 설명한 대로), AL2(Linux 커널 5 이상)와 같은 Linux 배포에서 THP의 기본값을 `always`에서 `madvise`로 변경할 수 있습니다. 이 설정은 `always`으로 기본 설정됩니다:
```
# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
# echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
# cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

기본값을 `always`에서 `madvise`로 변경하더라도 `-XX:+UseTransparentHugePages`를 지정하면 JVM은 여전히 Java 힙 및 코드 캐시에 THP를 사용할 수 있습니다.

### JAR에서 x86 공유 객체 찾기
Java JAR은 아키텍처에 고유한 공유 객체를 포함할 수 있습니다. 일부 자바 라이브러리는 이러한 공유 객체가 발견되었는지, 발견되면 JNI를 사용하여 함수의 일반적인 자바 구현에 의존하는 대신 네이티브 라이브러리로 호출합니다. 코드가 작동할 수 있지만 JNI가 없으면 성능이 저하될 수 있습니다.

JAR에 이러한 공유 객체가 포함되어 있는지 확인하는 빠른 방법은 단순히 압축을 풀고 결과 파일이 공유 객체인지, 그리고 aarch64(arm64) 공유 객체가 누락되었는지 확인하는 것입니다:
```
$ unzip foo.jar
$ find . -name "*.so" -exec file {} \;
```
각 x86-64 ELF 파일에 대해 이진 파일에 해당하는 aarch64 ELF 파일이 있는지 확인합니다. 일부 공통 패키지(예: commons-crypto)를 사용하면 arm을 수동으로 지원하는 JAR을 만들 수 있지만 Maven과 같은 아티팩트 저장소에는 업데이트된 버전이 없다는 것을 알 수 있습니다. 특정 아티팩트 버전이 Arm을 지원하는지 확인하려면 [Common JARs with native code Table](CommonNativeJarsTable.md)을 참조하십시오. 언제든지 이 [GitHub repo](https://github.com/aws/aws-graviton-getting-started)에서 이슈를 오픈하거나 ec2-arm-dev-feedback@amazon.com 으로 문의하여 필요한 Jar에 대한 Arm 지원을 받으십시오. 

### 멀티 아키텍처 Jar 빌드하기
자바는 한 번 쓰고, 어디서든 실행되도록 되어 있습니다. 네이티브 코드가 포함된 Java 아티팩트를 빌드할 때 모든 곳에서 원활하고 최적의 성능을 제공하도록 각 주요 아키텍처에 대해 라이브러리를 빌드하는 것이 중요합니다. Graviton과 x86 기반 인스턴스 모두에서 잘 실행되는 코드는 패키지의 유용성을 증가시킵니다.

Maven, SBT, Gradle 등을 사용하여 최종 패키징을 수행하기 전에 일반적으로 지원되는 각 아키텍처에 대해 네이티브 공유 객체를 빌드하는 다단계 프로세스가 있습니다. 아래는 x86 및 Graviton 프로세서 기반의 AWS EC2 인스턴스에서 Java 애플리케이션을 교차 실행하기 위한 여러 배포 및 아키텍처를 포함하는 Maven을 사용하여 JAR을 생성하는 방법의 예입니다:

```
# Create two build instances, one x86 and one Graviton instance.
# Pick one instance to be the primary instance.
# Log into the secondary instance
$ cd java-lib
$ mvn package
$ find target/ -name "*.so" -type f -print

# Note the directory this so file is in, it will be in a directory
# such as: target/classes/org/your/class/hierarchy/native/OS/ARCH/lib.so

# Log into the primary build instance
$ cd java-lib
$ mvn package

# Repeat the below two steps for each OS and ARCH combination you want to release
$ mkdir target/classes/org/your/class/hierarchy/native/OS/ARCH
$ scp slave:~/your-java-lib/target/classes/org/your/class/hierarchy/native/OS/ARCH/lib.so target/classes/org/your/class/hierarchy/native/OS/ARCH/

# Create the jar packaging with maven.  It will include the additional
# native libraries even though they were not built directly by this maven process.
$ mvn package

# When creating a single Jar for all platform native libraries, 
# the release plugin's configuration must be modified to specify 
# the plugin's `preparationGoals` to not include the clean goal.
# See http://maven.apache.org/maven-release/maven-release-plugin/prepare-mojo.html#preparationGoals
# For more details.

# Maven Central 및/또는 Sonatype Nexus로 릴리스하려면:
$ mvn release:prepare
$ mvn release:perform
```

이것은 단일 JAR에 있는 모든 라이브러리와 함께 JAR 패키징을 수행하는 한 가지 방법입니다. 모든 JAR을 빌드하려면 기본 시스템에 빌드하는 것이 좋지만 빌드x 플러그인을 사용하여 도커를 통해 빌드하거나 빌드 환경 내에서 크로스 컴파일하여 빌드할 수도 있습니다.

네이티브 코드로 jar를 릴리스하기 위한 추가 옵션은 다음과 같습니다: [nar maven plugin](https://maven-nar.github.io/)과 같은 매니저 플러그인을 사용하여 각 플랫폼별 jar를 관리합니다. 개별 아키텍처별 jar를 릴리스한 다음 기본 인스턴스를 사용하여 릴리스된 jar를 다운로드하고 최종 `mvn release:perform`과 함께 결합된 jar로 패키지합니다. 이 방법의 예는 [Leveldbjni-native](https://github.com/fusesource/leveldbjni) `pom.xml` 파일에서 찾을 수 있습니다.


### Java 응용프로그램 프로파일링
JIT(Java와 같은)에 의존하는 언어의 경우 캡처되는 기호 정보가 부족하여 런타임의 사용 위치를 이해하기 어렵습니다. 위의 코드 프로파일링 예시와 유사하게, `libperf-jvmti.so`는 JVM이 실행될 때 JITed 코드의 기호를 덤프하는 데 사용될 수 있습니다.

```bash
# Compile your Java application with -g

# find where libperf-jvmti.so is on your distribution

# Run your java app with -agentpath:/path/to/libperf-jvmti.so added to the command line
# Launch perf record on the system
$ perf record -g -k 1 -a -o perf.data sleep 5

# Inject the generated methods information into the perf.data file
$ perf inject -j -i perf.data -o perf.data.jit

# View the perf report with symbol info
$ perf report -i perf.data.jit

# Process the new file, for instance via Brendan Gregg's Flamegraph tools
$ perf script -i perf.data.jit | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > ./flamegraph.svg
```

### Amazon Linux 2에서 libperf-jvmti.so 빌드
Amazon Linux 2는 `libperf-jvmti.so`를 커널 버전 5.10 이하에는 패키징하지 않습니다.
다음 단계를 사용하여 `libperf-jvmti.so` 공유 라이브러리를 빌드합니다:

```bash
$ sudo amazon-linux-extras enable corretto8
$ sudo yum install -y java-1.8.0-amazon-corretto-devel

$ cd $HOME
$ sudo yumdownloader --source kernel

$ cat > .rpmmacros << __EOF__
%_topdir    %(echo $HOME)/kernel-source
__EOF__

$ rpm -ivh ./kernel-*.amzn2.src.rpm
$ sudo yum-builddep kernel
$ cd kernel-source/SPECS
$ rpmbuild -bp kernel.spec

$ cd ../BUILD
$ cd kernel-*.amzn2
$ cd linux-*.amzn2.aarch64
$ cd tools/perf
$ make
```
