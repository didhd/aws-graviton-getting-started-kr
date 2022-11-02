# Graviton 기반 인스턴스에 사용할 수 있는 운영 체제

 이름 | 버전 | [LSE 지원여부](optimizing.md#locksynchronization-intensive-workload) | Kernel 페이지 크기 | AMI | Metal 지원 여부 | 설명
------ | ------ | ----- | ----- | ----- | ----- | -----
Amazon Linux 2 | 2.26-35 or later| 네 | 4KB | [AMIs](amis_cf_sm.md) | 네 |
Ubuntu | 20.04 LTS or later | 네 | 4KB | [focal](https://cloud-images.ubuntu.com/locator/ec2/) | 네 | 
Ubuntu | 18.04 LTS | 네 (*) | 4KB | [bionic](https://cloud-images.ubuntu.com/locator/ec2/) | 네 | (*) needs `apt install libc6-lse`
SuSE | 15 SP2 or later| Planned | 4KB | [MarketPlace](https://aws.amazon.com/marketplace/pp/B07SPTXBDX) | 네 | 
Redhat Enterprise Linux | 8.2 or later | 네 | 64KB | [MarketPlace](https://aws.amazon.com/marketplace/pp/B07T2NH46P) | 네 | 
~~Redhat Enterprise Linux~~ | ~~7.x~~ | ~~아니오~~ | ~~64KB~~ | ~~[MarketPlace](https://aws.amazon.com/marketplace/pp/B07KTFV2S8)~~ | | Supported on A1 instances but not on Graviton2 based ones
AlmaLinux | 8.4 or later | 네 | 64KB | [AMIs](https://wiki.almalinux.org/cloud/AWS.html) | 네 |
Alpine Linux | 3.12.7 or later | 네 (*) | 4KB | [AMIs](https://www.alpinelinux.org/cloud/) | | (*) LSE enablement checked in version 3.14 |
CentOS | 8.2.2004 or later | 아니오 | 64KB | [AMIs](https://wiki.centos.org/Cloud/AWS#Images) | 네 | |
CentOS Stream | 8 | 아니오 (*) | 64KB (*) | [Downloads](https://www.centos.org/centos-stream/) | |(*) details to be confirmed once AMI's are available|
~~CentOS~~ | ~~7.x~~ | ~~아니오~~ | ~~64KB~~ | ~~[AMIs](https://wiki.centos.org/Cloud/AWS#Images)~~ | | Supported on A1 instances but not on Graviton2 based ones
Debian | 11 | 네 | 4KB | [Community](https://wiki.debian.org/Cloud/AmazonEC2Image/Bullseye) or [MarketPlace](https://aws.amazon.com/marketplace/pp/prodview-jwzxq55gno4p4) | 네 |
Debian | 10 | [Planned for Debian 11](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=956418) | 4KB | [Community](https://wiki.debian.org/Cloud/AmazonEC2Image/Buster) or [MarketPlace](https://aws.amazon.com/marketplace/pp/B085HGTX5J) | 네, Debian 10.7 부터 (2020-12-07) |
FreeBSD | 12.1 or later | 아니오 | 4KB | [Community](https://www.freebsd.org/releases/12.1R/announce.html) or [MarketPlace](https://aws.amazon.com/marketplace/pp/B081NF7BY7) | 아니오 | Device hotplug and API shutdown don't work
FreeBSD | 13.0 or later | 네 | 4KB | [Community](https://www.freebsd.org/releases/13.0R/announce.html) or [MarketPlace](https://aws.amazon.com/marketplace/pp/B09291VW11) | 아니오 | Device hotplug and API shutdown don't work
Rocky Linux | 8.4 or later | 네 (*) | 64KB (*) | [ISOs](https://rockylinux.org/download) | | [Release Notes](https://docs.rockylinux.org/release_notes/8-changelog/)<br>(*) details to be confirmed once AMI's are available


OS 이름 | Graviton2 용 최소 권장 Linux 커널 버전
------ | ------
Amazon Linux 2 | >= 4.14.273-207.502.amzn2, >= 5.4.186-102.354.amzn2, or >= 5.10.106-102.504.amzn2
Ubuntu 20.04 | >= 5.4.0-1047-aws, >= 5.8.0-1034-aws, >= 5.11.0-1009-aws
Ubuntu 18.04 | >= 4.15.0-1101-aws, >= 5.4.0-1047-aws
Redhat Entreprise Linux 8 | >= 4.18.0-305.10
