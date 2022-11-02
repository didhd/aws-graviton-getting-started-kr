# Amazon Machine Images (AMIs)

이 문서에서는 Graviton과 호환되는 AMI를 찾는 방법과 AWS System Manager 및 Cloud Formation에서 AMI를 조회하고 사용하는 방법에 대해 설명합니다.

## Amazon Linux 2

### 콘솔을 통해 찾기

Amazon Linux 2 AMI는 마법사를 통해 인스턴스를 시작할 때 [AWS 설명서](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html#finding-an-ami-console)에 설명된 대로 콘솔에서 검색이 가능합니다.

### API 사용 - AWS Systems Manager Parameter Store

스크립트 또는 코드 조각에 AMI를 찾는 프로세스를 통합할 때 [AWS Systems Manager](https://aws.amazon.com/systems-manager/) Parameter Store를 활용하는 것이 더 편리합니다.

AWS Compute 블로그에 [AWS Systems Manager Parameter Store를 사용한 최신 Amazon Linux AMI ID 쿼리](https://aws.amazon.com/blogs/compute/query-for-the-latest-amazon-linux-ami-ids-using-aws-systems-manager-parameter-store/)에 대한 좋은 글이 있습니다.

Graviton2/arm64 Amazon Linux 2 AMI의 경우 네임스페이스는 다음과 같습니다:

- **Parameter Store 접두사 (tree)**: ```/aws/service/ami-amazon-linux-latest/``` 
- **AMI 이름 별칭**: ```amzn2-ami-hvm-arm64-gp2```

다음은 ```us-east-1``` 의 최신 Amazon Linux 2 버전의 AMI_ID를 가져오는 예입니다:

```
$ aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-arm64-gp2 --region us-east-1 --query "Parameters[].Value" --output text
```

AWS CloudFormation은 Parameter Store도 지원하며, CloudFormation 템플릿에서 최신 Graviton2/arm64 Amazon Linux 2 AMI에 대한 참조를 추가하는 방법에 대한 예입니다:

```YAML
Parameters:
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-arm64-gp2'

Resources:
 Graviton2Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: 'c6g.medium'
```
