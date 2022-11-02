# Graviton의 컨테이너 기반 워크로드

AWS Graviton 프로세서는 컨테이너 기반 워크로드에 이상적입니다.

### Graviton 준비하기

컨테이너 호스트로서 Graviton 기반 인스턴스의 이점을 활용하는 첫 번째 단계는 모든 소프트웨어 그리고 종속성이 arm64 아키텍처를 지원하도록 하는 것입니다. x86_64 호스트를 위해 작성된 이미지는 arm64 호스트에서 실행할 수 없고 그 반대도 마찬가지입니다.

대부분의 컨테이너 에코시스템은 두 아키텍처를 모두 지원하며, 호스트 아키텍처에 맞는 올바른 이미지가 자동으로 배포되는 [멀티 아키텍처](https://www.docker.com/blog/multi-platform-docker-builds/) 이미지를 통해 투명하게 지원하는 경우가 많습니다.

[Dockerhub](https://hub.docker.com), [Quay](https://www.quay.io), 그리고 [Amazon Elastic Container Registry (ECR)](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) 등의 대부분 컨테이너 이미지 레포지토리들은 모두 [멀티 아키텍처](https://aws.amazon.com/blogs/containers/introducing-multi-architecture-container-images-for-amazon-ecr/) 이미지를 지원합니다.

#### 멀티 아키텍처 이미지 만들기

대부분의 이미지는 이미 멀티 아키텍처(즉, arm64 및 x86_64/amd64)를 지원하지만, 필요한 경우 개발자가 멀티 아키텍처 이미지를 직접 만들 수 있는 몇 가지 방법을 설명하겠습니다.

1. [Docker Buildx](https://github.com/docker/buildx#getting-started)
2. [Amazon CodePipeline](https://github.com/aws-samples/aws-multiarch-container-build-pipeline)과 같은 CI/CD 빌드 파이프라인을 사용하여 네이티브 빌드 및 매니페스트 생성을 관리합니다.

### Graviton으로 배포

대부분의 컨테이너 오케스트레이션 플랫폼은 arm64 및 x86_64 호스트를 모두 지원합니다.

[Amazon Elastic Container Service(ECS)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html#amazon-linux-2-(arm64)) 및 [Amazon Elastic Kubernetes Service(EKS)](https://aws.amazon.com/blogs/containers/eks-on-graviton-generally-available/)는 모두 Graviton-powered 인스턴스를 지원합니다.

arm64를 명시적으로 지원하는 컨테이너 생태계 내에서 인기 있는 소프트웨어 목록을 아래에 작성하였습니다:

#### 에코시스템 지원

| 이름                          | URL                                                                                                                | 설명                                                                                                 |
| :---------------------------- | :----------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| Istio                         | https://github.com/istio/istio/releases/                                                                           | 1.15.0 릴리스 이후 도커 허브에서 arm64용 컨테이너 이미지 제공                         |
| Envoy                         | https://www.envoyproxy.io/docs/envoy/v1.18.3/start/docker                                                          |                                                                                                         |
| Tensorflow                    | https://hub.docker.com/r/armswdev/tensorflow-arm-neoverse                                                          |                                                                                                         |
| Tensorflow serving            | 763104351884.dkr.ecr.us-west-2.amazonaws.com/tensorflow-inference-graviton:2.7.0-cpu-py38-ubuntu20.04-e3-v1.0      |                                                                                                         |
| PyTorch                       | https://hub.docker.com/r/armswdev/pytorch-arm-neoverse                                                             | 성능상의 이유로 *-openblas*가 포함된 태그를 사용할 것                             |
| Traefik                       | https://github.com/containous/traefik/releases                                                                     |                                                                                                         |
| Flannel                       | https://github.com/coreos/flannel/releases                                                                         |                                                                                                         |
| Helm                          | https://github.com/helm/helm/releases/tag/v2.16.9                                                                  |                                                                                                         |
| Jaeger                        | https://github.com/jaegertracing/jaeger/pull/2176                                                                  |                                                                                                         |
| Fluent-bit                    | https://github.com/fluent/fluent-bit/releases/                                                                     | 소스로부터 컴파일                                                                                     |
| core-dns                      | https://github.com/coredns/coredns/releases/                                                                       |                                                                                                         |
| external-dns                  | https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md#which-architectures-are-supported          | 0.7.5 부터 지원+                                                                                     |
| Prometheus                    | https://prometheus.io/download/                                                                                    |                                                                                                         |
| containerd                    | https://github.com/containerd/containerd/issues/3664                                                               | arm64를 위해 nightly builds 제공                                                                        |
| kube-state-metrics            | https://github.com/kubernetes/kube-state-metrics/issues/1037                                                       | arm64는 k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.0.0-beta를 사용                              |
| cluster-autoscaler            | https://github.com/kubernetes/autoscaler/pull/3714                                                                 | arm64는 v1.20부터 지원.0                                                                             |
| gRPC                          | https://github.com/protocolbuffers/protobuf/releases/                                                              | protoc/protobuf 지원                                                                                 |
| Nats                          | https://github.com/nats-io/nats-server/releases/                                                                   |                                                                                                         |
| CNI                           | https://github.com/containernetworking/plugins/releases/                                                           |                                                                                                         |
| Cri-o                         | https://github.com/cri-o/cri-o/blob/master/README.md#installing-crio                                               | Ubuntu 18.04와 20.04 테스트                                                                       |
| Trivy                         | https://github.com/aquasecurity/trivy/releases/                                                                    |                                                                                                         |
| Argo                          | https://github.com/argoproj/argo-cd/releases                                                                       | 2.3.0 부터 arm64 images 배포                                                                       |
| Cilium                        | https://docs.cilium.io/en/stable/contributing/development/images/                                                  | v1.10.0 부터 Multi arch 지원                                                                       |
| Calico                        | https://hub.docker.com/r/calico/node/tags?page=1&ordering=last_updated                                             | master에서 Multi arch 지원                                                                           |
| Tanka                         | https://github.com/grafana/tanka/releases                                                                          |                                                                                                         |
| Consul                        | https://www.consul.io/downloads                                                                                    |                                                                                                         |
| Nomad                         | https://www.nomadproject.io/downloads                                                                              |                                                                                                         |
| Packer                        | https://www.packer.io/downloads                                                                                    |                                                                                                         |
| Vault                         | https://www.vaultproject.io/downloads                                                                              |                                                                                                         |
| Terraform                     | https://github.com/hashicorp/terraform/issues/14474                                                                | v0.14부터 arm64 지원                                                                             |
| Flux                          | https://github.com/fluxcd/flux/releases/                                                                           |                                                                                                         |
| Pulumi                        | https://github.com/pulumi/pulumi/issues/4868                                                                       | v2.23 부터 arm64 지원                                                                             |
| New Relic                     | https://download.newrelic.com/infrastructure_agent/binaries/linux/arm64/                                           |                                                                                                         |
| Datadog - EC2                 | https://www.datadoghq.com/blog/datadog-arm-agent/                                                                  |                                                                                                         |
| Datadog - Docker              | https://hub.docker.com/r/datadog/agent-arm64                                                                       |                                                                                                         |
| Dynatrace                     | https://www.dynatrace.com/news/blog/get-out-of-the-box-visibility-into-your-arm-platform-early-adopter/            |                                                                                                         |
| Grafana                       | https://grafana.com/grafana/download?platform=arm                                                                  |                                                                                                         |
| Loki                          | https://github.com/grafana/loki/releases                                                                           |                                                                                                         |
| kube-bench                    | https://github.com/aquasecurity/kube-bench/releases/tag/v0.3.1                                                     |                                                                                                         |
| metrics-server                | https://github.com/kubernetes-sigs/metrics-server/releases/tag/v0.3.7                                              | v.0.3.7 부터 docker image가 multi-arch 지원                                                                  |
| AWS Copilot                   | https://github.com/aws/copilot-cli/releases/tag/v0.3.0                                                             | v0.3.0 부터 arm64 지원                                                                               |
| AWS ecs-cli                   | https://github.com/aws/amazon-ecs-cli/pull/1110                                                                    | v1.20.0 바이너리가 us-west-2 s3 에 있음                                                                        |
| Amazon EC2 Instance Selector  | https://github.com/aws/amazon-ec2-instance-selector/releases/                                                      | 추가로 특정 영역에서 arm64 기반 인스턴스를 검색하기 위해 -a cpu_architecture 플래그를 지원 |
| AWS Node Termination Handler  | https://github.com/aws/aws-node-termination-handler/releases/                                                      | kubernetes에서 arm64 지원 (helm 통해 지원)                                                               |
| AWS IAM Authenticator         | https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html                                |                                                                                                         |
| AWS ALB Ingress Controller    | https://github.com/kubernetes-sigs/aws-alb-ingress-controller/releases/tag/v1.1.9                                  | v1.1.9 부터 멀티 아키텍처 이미지 지원                                                                           |
| AWS EFS CSI Driver            | https://github.com/kubernetes-sigs/aws-efs-csi-driver/pull/241                                                     | 8/27/2020 부터 지원                                                                                |
| AWS EBS CSI Driver            | https://github.com/kubernetes-sigs/aws-ebs-csi-driver/pull/527                                                     | 8/26/2020 부터 지원                                                                                |
| Amazon Inspector Agent        | https://docs.aws.amazon.com/inspector/latest/userguide/inspector_installing-uninstalling-agents.html#install-linux |                                                                                                         |
| Amazon CloudWatch Agent       | https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html                       |                                                                                                         |
| AWS Systems Manager SSM Agent | https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-manual-agent-install.html                      |                                                                                                         |
| AWS CLI                       | https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#ARM                                      | v1 과 v2 모두 지원됨                                                                                |
| FireLens for Amazon ECS       | https://github.com/aws/aws-for-fluent-bit/issues/44                                                                | v2.9.0 부터 arm64 지원                                                                               |  |
| Flatcar Container Linux       | https://www.flatcar.org                                                                                            | Stable channel 3033.2.0 부터 arm64 지원                                                          |


**소프트웨어가 위에 나열되지 않았다고 해서 작동하지 않는 것은 아닙니다!**

많은 제품들이 arm64에서 작동하지만 arm64 바이너리를 *(아직)* 명시적으로 배포하거나 멀티 아키텍처 이미지를 빌드하지는 않습니다. AWS, Arm 및 커뮤니티의 많은 개발자들은 완전한 바이너리와 멀티 아키텍처 지원을 가능하게 하기 위해 메인테이너와 협력하고 전문 지식과 코드를 제공하고 있습니다.

### 쿠버네티스

쿠버네티스 (및 EKS)는 arm64를 지원하므로 Graviton 인스턴스를 지원합니다. 컨테이너형 워크로드가 모두 arm64를 지원하는 경우 Graviton 노드를 사용하여 클러스터를 단독으로 실행할 수 있습니다. 그러나 amd64(x86) 인스턴스에서만 실행할 수 있는 워크로드가 있거나 동일한 클러스터에서 amd64(x86) 및 arm64 노드를 모두 실행하려는 경우 다음과 같은 몇 가지 방법을 사용할 수 있습니다:

#### 멀티 아키텍처 이미지
클러스터의 모든 컨테이너에 대해 멀티 아키텍처 이미지(위 참조)를 사용할 수 있는 경우 추가 작업 없이 amd64 및 arm64 노드의 혼합을 실행할 수 있습니다. 멀티아키텍처 이미지 매니페스트는 지정된 노드의 아키텍처에 대해 올바른 이미지 레이어를 Pull 할 수 있도록 합니다.

#### 빌트인 레이블 (Built-in labels)
`kubernetes.io/arch` [레이블](https://kubernetes.io/docs/reference/labels-annotations-taints/#kubernetes-io-arch)에 따라 노드에 파드를 스케줄링할 수 있습니다. 이 레이블은 쿠버네티스에 의해 노드에 자동으로 추가되며, 다음과 같은 [노드 셀렉터](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodes selector)를 사용하여 포드을 적절하게 스케줄링할 수 있습니다:

```
nodeSelector:
  kubernetes.io/arch: amd64
```

#### 테인트(Taints) 사용하기
테인트(Taints)는 특히 amd64 전용(x86 전용) 컨테이너가 대부분인 기존 클러스터에 Graviton 노드를 추가할 때 유용합니다. 빌트인 `kubernetes.io/arch` 레이블을 사용하려면 노드 셀렉터를 사용하여 올바른 인스턴스에 arm64 전용 컨테이너를 배치해야 하지만 Graviton 인스턴스를 테인트를 추가하면 기존 구성을 변경할 필요 없이 호환되지 않는 컨테이너는 예약이 안되도록 만들 수 있습니다. 예를 들어 eksctl을 사용하여 관리 노드 그룹에서 이 작업을 수행하려면 `--kubelet-extra-args '--register-with-taints=arm=true:NoSchedule'`을 kubelet 시작 인수에 [문서](https://eksctl.io/usage/eks-managed-nodes/)와 같이 추가하면 됩니다. arm64 인스턴스만 테인트를 추가하고 노드 셀렉터를 지정하지 않으면 Graviton 인스턴스에 대해 빌드하는 이미지가 x86 인스턴스 유형에서도 실행할 수 있는 멀티 아키텍처 이미지인지 확인해야 합니다. 또는 arm64 전용 이미지를 빌드하고 노드 셀렉터를 사용하여 arm64 노드에만 스케줄링 되도록 할 수 있습니다.

#### Cluster Autoscaler에 대한 고려사항
x86 기반 및 Graviton 인스턴스 유형을 모두 가진 클러스터에서 쿠버네티스 [Cluster Autoscaler](https://github.com/kubernetes/autoscaler)를 사용하는 경우 [문서](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html)에 따라 각 오토 스케일링 그룹에 `k8s.io/cluster-autoscaler/node-template/label/*` 또는 `k8s.io/cluster-autoscaler/node-template/taint/*` 태그를 지정해서 오토 스케일러가 어떤 ASG에 어떤 포드를 배치할 수 있는지 알 수 있게 해야합니다. (이것은 실제로 노드에 레이블이나 테인트를 적용하지 않고 클러스터 오토 스케일러에 스케줄링 힌트를 제공하는 역할만 합니다.)

---

### 더 읽기

* [EC2 Graviton에서의 native build를 통한 docker Buildx 속도 높이기](https://www.docker.com/blog/speed-up-building-with-docker-buildx-and-graviton2-ec2/)
* [buildx로 multi arch docker 이미지 빌드하기](https://tech.smartling.com/building-multi-architecture-docker-images-on-arm-64-c3e6f8d78e1c)
* [도커와 Arm 소프트웨어 개발 통합](https://community.arm.com/developer/tools-software/tools/b/tools-software-ides-blog/posts/unifying-arm-software-development-with-docker)
* [도커를 사용한 최신 멀티 아키텍처 빌드](https://duske.me/posts/modern-multiarch-builds-with-docker/)
* [EKS와 함께 Spot 및 Graviton2 활용](https://spot.io/blog/eks-simplified-on-ec2-graviton2-instances/)
