# Graviton 성능 런북

## 소개

이 문서는 AWS Graviton 기반 인스턴스에서 응용 프로그램 코드를 벤치마킹, 디버그 및 최적화하려는 소프트웨어 개발자를 위한 참조 자료입니다. EC2 Graviton 팀이 수집한 체크리스트, 모범 사례, 예제 및 툴링으로 Graviton의 코드 벤치마킹, 디버깅 또는 최적화 작업을 지원합니다.

이 문서에서는 벤치마크 방법, 성능 디버그 방법 및 최적화 권장 사항을 포함한 많은 항목을 다룹니다. 문서는 처음부터 끝까지 읽도록 의도된 것은 아닙니다. 대신 문서는 시스템 분석에 점차 더 깊이 들어가는 Graviton 인스턴스로 작업할 때 적용할 체크리스트와 가장 잘 알려진 관행입니다. 아래 FAQ를 참조하여 특정 상황에 따라 가장 적합한 체크리스트 및 도구 세트를 찾으세요.

이러한 가이드를 따른 후에도 Graviton2의 성능과 관련하여 해결할 수 없는 문제가 있는 경우, 주저하지 말고 [AWS-Graviton-Getting-Started](https://github.com/aws/aws-graviton-getting-started/issues) 가이드에 문제를 제기하거나 [ec2-arm-dev-feedback@amazon.com](mailto:ec2-arm-dev-feedback@amazon.com)로 문의하십시오. 이 가이드에 누락된 내용이 있으면 이슈를 오픈하거나 더 나은 내용으로 PR을 올려주세요.

## 사전 조건

이 런북에 나열된 일부 작업을 지원하기 위해 체크리스트에서 설명하는 일부 작업에 대한 일부 도우미 스크립트를 만들었습니다. 도우미 스크립트는 테스트 인스턴스가 최신 AL2 또는 Ubuntu 20.04를 실행하고 있다고 가정합니다.LTS 배포 및 사용자는 `sudo`를 사용하여 스크립트를 실행할 수 있습니다. 아래 단계에 따라 테스트 시스템에 유틸리티를 가져오고 설치하십시오:

```bash
# Clone the repository onto your systems-under-test and any load-generation instances
git clone https://github.com/aws/aws-graviton-getting-started.git
cd aws-graviton-getting-started/perfrunbook/utilities

# On AL2 or Ubuntu distribution
sudo ./install_perfrunbook_dependencies.sh

# All scripts expect to run from the utilities directory
```

## 섹션 (WIP)

1. [Introduction to Benchmarking](./intro_to_benchmarking.md)
2. [Defining your benchmark](./defining_your_benchmark.md)
3. [Configuring your load generator](./configuring_your_loadgen.md)
4. [Configuring your system-under-test environment](./configuring_your_sut.md)
5. Debugging Performance
    1. [Debugging performance — “What part of the system is slow?”](./debug_system_perf.md)
    2. [Debugging performance — “What part of the code is slow?”](./debug_code_perf.md)
    3. [Debugging performance — “What part of the hardware is slow?”](./debug_hw_perf.md)
6. [Optimizing performance](./optimization_recommendation.md)
7. [Appendix — Additional resources](./appendix.md)
8. [References](./references.md)

## FAQ

* **I want to benchmark Graviton but I have yet to port my application, where do I find information on helping port my application?**
    Our getting-started-guide has many resources to help with porting code to Graviton for a number of programming languages.  Start by reading those guides to understand minimum runtime, dependency and language requirements needed for Graviton.
* **What benchmark should I run to determine if my application will be a good fit on Graviton?**
    No synthetic benchmark is a substitute for your actual production code.  The best benchmark is running your production application on Graviton with a load that approximates your production load.  Please refer to [Section 1](./intro_to_benchmarking.md) and [2](./defining_your_benchmark.md) on how to benchmark and on selecting a benchmark for more details.
* **I ran micro-benchmark X from github project Y and it shows Graviton has worse performance, does that mean my application is not a good fit?**
    No.  Benchmarks only tell a limited story about performance, and unless this particular benchmark has been vetted as a good indicator for your application’s performance, we recommend running your production code as its own benchmark.  For more details, refer to [Section 1](./intro_to_benchmarking.md) and [2](./defining_your_benchmark.md) on how to define experiments and test Graviton for your needs.
* **I benchmarked my service and performance on Graviton is slower compared to my current x86 based fleet, where do I start to root cause why?**
    Begin by verifying software dependencies and verifying the configuration of your Graviton and x86 testing environments to check that no major differences are present in the testing environment.  Performance differences may be due to differences in environment and not the due to the hardware.  Refer to the below chart for a step-by-step flow through this runbook to help root cause the performance regression:
    ![](./images/performance_debug_flowchart.png)
* **What are the recommended optimizations to try with Graviton2?**
    Refer to [Section 6](./optimization_recommendation.md) for our recommendations on how to make your application run faster on Graviton.
* **I investigated every optimization in this guide and still cannot find the root-cause, what do I do next?**
    Please contact us at [ec2-arm-dev-feedback@amazon.com](mailto:ec2-arm-dev-feedback@amazon.com) or talk with your AWS account team representative to get additional help.

