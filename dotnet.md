# Graviton에서의 .NET
.NET은 다양한 유형의 응용 프로그램을 작성할 수 있는 오픈 소스 플랫폼입니다. 소프트웨어 엔지니어는 C#, F# 및 Visual Basic과 같은 언어로 .NET 기반 애플리케이션을 작성할 수 있습니다. .NET 애플리케이션은 공통 중간 언어(CIL)로 컴파일됩니다. 응용 프로그램이 실행될 때 공통 언어 런타임(CLR)은 해당 응용 프로그램 바이너리를 로드하고 JIT 컴파일러를 사용하여 실행 중인 아키텍처를 위한 기계 코드를 생성합니다. 자세한 내용은 [what is .NET](https://dotnet.microsoft.com/learn/dotnet/what-is-dotnet)을 참조하십시오.


## .NET 버전

버전            | Linux Arm32   | Linux Arm64   | 노트
------------------|-----------|-----------|-------------
[.NET 6](https://dotnet.microsoft.com/download/dotnet/6.0) | Yes | Yes |  V6.0.0은 2021년 11월 8일에 Arm64 리눅스 빌드와 함께 출시되었습니다. 자세한 내용은 [Blog: .NET 6 on AWS](https://aws.amazon.com/blogs/developer/net-6-on-aws/)와 [Video: AWS re:Invent 2021 - Accelerate .NET 6 performance with Arm64 on AWS Graviton2](https://www.youtube.com/watch?v=iMlyZI9NhFw)를 참조하십시오.
[.NET 5](https://dotnet.microsoft.com/download/dotnet/5.0) | Yes | Yes | .NET 5를 사용하여 마이크로소프트는 특정 Arm64 아키텍처를 최적화했습니다. RyuJIT에 의해 코드가 생성됩니다. [Arm64 Performance in .NET 5](https://devblogs.microsoft.com/dotnet/arm64-performance-in-net-5/) 
[.NET Framework 4.x](https://dotnet.microsoft.com/learn/dotnet/what-is-dotnet-framework) | No | No | The original implementation of the .NET Framework does not support Linux hosts, and Windows hosts are not suported on Graviton. 
[.NET Core 3.1](https://dotnet.microsoft.com/download/dotnet/3.1) | Yes | Yes | .NET Core 3.0 added support for [Arm64 for Linux](https://docs.microsoft.com/en-us/dotnet/core/whats-new/dotnet-core-3-0#linux-improvements). 
[.NET Core 2.1](https://dotnet.microsoft.com/download/dotnet/2.1) | Yes* | No | Initial support was for [Arm32 was added to .NET Core 2.1](https://github.com/dotnet/announcements/issues/82). *Operating system support is limited, please see the [official documentation](https://github.com/dotnet/core/blob/main/release-notes/2.1/2.1-supported-os.md).


## .NET 5
.NET 5를 사용하여 마이크로소프트는 특정 Arm64 아키텍처를 최적화했습니다. 이러한 최적화는 .NET 라이브러리 및 JIT 프로세스에 의한 기계 코드 출력에서 수행되었습니다.

 * AWS DevOps Blog [Build and Deploy .NET web applications to ARM-powered AWS Graviton 2 Amazon ECS Clusters using AWS CDK](https://aws.amazon.com/blogs/devops/build-and-deploy-net-web-applications-to-arm-powered-aws-graviton-2-amazon-ecs-clusters-using-aws-cdk/)
 * AWS Compute Blog [Powering .NET 5 with AWS Graviton2: Benchmarks](https://aws.amazon.com/blogs/compute/powering-net-5-with-aws-graviton2-benchmark-results/) 
 * Microsoft .NET Blog [ARM64 Performance in .NET 5](https://devblogs.microsoft.com/dotnet/arm64-performance-in-net-5/)


## Linux Arm64용 빌드 및 게시
.NET SDK는 응용 프로그램이 실행되는 플랫폼을 대상으로 하는 데 사용되는 [Runtime Identifier(RID)](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog)를 선택할 수 있도록 지원합니다. 이러한 RID는 NUGet 패키지의 플랫폼별 리소스를 나타내는 NET 종속성(NuGet 패키지)에 의해 사용됩니다. 다음 값은 RID의 예입니다: linux-arm64, linux-x64, ubuntu.14.04-x64, win7-x64 또는 osx.10.12-x64. 네이티브 종속성이 있는 NuGet 패키지의 경우 RID는 패키지를 복원할 수 있는 플랫폼을 지정합니다.

모든 호스트 운영 체제에서 빌드 및 게시할 수 있습니다. 예를 들어 윈도우즈에서 개발하고 로컬로 빌드하여 Arm64를 타겟으로 하거나 Linux의 Jenkins와 같은 CI 서버를 사용할 수 있습니다. 명령은 동일합니다.

```bash
dotnet build -r linux-arm64
dotnet publish -c Release -r linux-arm64
```

[publishing .NET apps with the .NET CLI](https://docs.microsoft.com/en-us/dotnet/core/deploying/deploy-with-cli)에 대한 자세한 내용을 보려면 공식 문서를 참조하십시오.
