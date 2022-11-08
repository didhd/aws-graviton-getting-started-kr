# Graviton에서의 Lambda
[AWS Lambda](https://aws.amazon.com/lambda/)를 사용하면 x86 기반 기능 외에도 Arm 기반 AWS Graviton2 프로세서에서 실행할 새 기능과 기존 기능을 구성할 수 있습니다. 이 프로세서 아키텍처 옵션을 사용하면 가격 성능을 최대 34% 향상시킬 수 있습니다. 기간 요금은 [밀리초 단위](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-1ms-billing-granularity-adds-cost-savings/)로 청구되며 현재 x86 가격 대비 20% 저렴합니다. 이는 [Provisioned Concurrency](https://aws.amazon.com/blogs/aws/new-provisioned-concurrency-for-lambda-functions/)을 사용할 때의 기간 요금에도 적용됩니다. Compute [Savings Plans](https://aws.amazon.com/blogs/aws/savings-plan-update-save-up-to-17-on-your-lambda-workloads/)는 Graviton2로 구동되는 Lambda 기능을 지원합니다.

블로그 게시물 [AWS Lambda 함수를 Arm 기반 AWS Graviton2 프로세서로 마이그레이션](https://aws.amazon.com/blogs/compute/migrating-aws-lambda-functions-to-arm-based-aws-graviton2-processors/)은 마이그레이션 프로세스가 코드 및 워크로드에 따라 다르므로 x86에서 arm64로 이동할 때 고려해야 할 몇 가지 사항을 보여줍니다.

이 페이지에서는 마이그레이션 고려 사항 중 일부를 강조하고 Arm/Graviton2를 사용하여 Lambda 기능을 빌드하고 마이그레이션하는 방법을 살펴보는 데 사용할 수 있는 간단한 배포 데모를 제공합니다.

아키텍처 변경은 함수가 호출되거나 함수가 응답을 전달하는 방식에 영향을 미치지 않습니다. API, 서비스, 애플리케이션 또는 툴과의 통합은 새로운 아키텍처의 영향을 받지 않으며 이전과 같이 계속 작동합니다.
[Amazon 리눅스 2](https://aws.amazon.com/amazon-linux-2/)가 사용하는 다음 런타임들은 Arm에서 지원됩니다:
*	Node.js 12 and 14
*	Python 3.8 and 3.9
*	Java 8 (java8.al2) and 11
*	.NET Core 3.1
*	Ruby 2.7
*	[사용자 지정 런타임](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html) (provided.al2)

## x86 Lambda 함수를 arm64로 마이그레이션
### 아키텍처별 종속성 또는 바이너리가 없는 기능
대부분의 Lambda 함수는 Graviton2의 가격/성능을 활용하기 위해 구성 변경만 필요할 수 있습니다. 다른 기능은 암별 종속성을 사용하여 Lambda 기능을 다시 패키징하거나 기능 이진 또는 용기 이미지를 재구성해야 할 수 있습니다.

기능에서 아키텍처별 종속성 또는 바이너리를 사용하지 않는 경우 하나의 설정 변경만으로도 한 아키텍처에서 다른 아키텍처로 전환할 수 있습니다. Node.js, Python과 같은 인터프리터 언어를 사용하는 많은 함수들 또는 자바 바이트코드로 컴파일된 함수들은 어떠한 변화도 없이 전환될 수 있습니다. 종속성, Lambda 계층 및 Lambda 확장의 바이너리를 확인해야 합니다.

### Graviton2를 위한 함수 빌드하기
Rust나 Go와 같은 컴파일된 언어의 경우 Arm을 지원하는 `provided.al2` [사용자 지정 런타임](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html)을 사용할 수 있습니다. [Lambda Runtime API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html)와 통신하는 바이너리를 제공하면 됩니다.

Go를 컴파일할 때는 `GOARCH`를 `arm64`로 설정합니다.
```
GOOS=linux GOARCH=arm64 go build
```
Rust를 컴파일 할때는, `target`을 설정하빈다..
```
cargo build --release -- target-cpu=neoverse-n1
```
일부 리눅스 배포판에 대한 파이썬 `pip`의 기본 버전는 오래되었습니다(<19.3). Graviton용으로 릴리즈된 바이너리 wheel 패키지를 설치하려면 다음을 사용하여 pip 설치를 업그레이드하십시오:
```
sudo python3 -m pip install --upgrade pip
````

컨테이너 이미지로 패키지화된 기능은 사용할 아키텍처(x86 또는 arm64)에 맞게 구축되어야 합니다. [AWS 제공 Lambda용 기본 이미지](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-images.html#runtimes-images-lp)에는 arm64 아키텍처 버전이 있습니다. arm64에 대한 컨테이너 이미지를 지정하려면 예를 들어 Node.js 14에 대해 arm64 특정 이미지 태그를 사용합니다:
```
public.ecr.aws/lambda/nodejs:14-arm64
public.ecr.aws/lambda/nodejs:latest-arm64
public.ecr.aws/lambda/nodejs:14.2021.10.01.16-arm64
```
Arm64 이미지는 [Docker Hub](https://hub.docker.com/u/amazon)에서도 사용할 수 있습니다.
AWS에서 제공한 Amazon Linux 2 이미지 외에도 임의 Linux 기본 이미지를 사용할 수도 있습니다. arm64를 지원하는 이미지에는 [Alpine](https://alpinelinux.org/) Linux 3.12.7 이상, [Debian](https://www.debian.org/) 10 및 11, Ubuntu 18.04 및 20.04이 있습니다. 지원되는 다른 리눅스 버전에 대한 자세한 내용과 자세한 내용은 [Graviton 기반 인스턴스에 사용할 수 있는 운영 체제](https://github.com/aws/aws-graviton-getting-started/blob/main/os.md#operating-systems-available-for-graviton-based-instances)를 참조하십시오.

[AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) 오픈 소스 프로젝트는 다양한 설정을 사용하여 기능을 실행하여 비용을 최소화하고 성능을 극대화할 수 있는 구성을 제안합니다. 이 도구를 사용하면 동일한 차트에서 두 결과를 비교하고 arm64 기반 가격을 통합할 수 있습니다. 이는 동일한 기능의 두 버전을 비교하는 데 유용합니다. 하나는 x86을 사용하고 다른 하나는 arm64를 사용합니다.

## 데모: Graviton2를 위한 Lambda 함수 빌드
데모 요구사항: 
* [AWS Serverless Application Model (AWS SAM)](https://aws.amazon.com/serverless/sam/)
* [Docker](https://docs.docker.com/get-docker/)

저장소를 복제하고 데모 디렉토리로 변경
```
git clone https://github.com/aws/aws-graviton-getting-started
cd aws-graviton-getting-started/aws-lambda/GravitonLambdaNumber
```
### Lambda 함수를 x86에서 arm64로 마이그레이션
이 데모에서는 x86 기반 워크스테이션을 사용하여 기존 Lambda 함수를 x86에서 arm64로 마이그레이션하는 방법을 보여 줍니다.
['/src/app.js'](/src/app.js)의 Node.js 함수 코드는 [axios](https://www.npmjs.com/package/axios) 클라이언트를 사용하여 서드파티 서비스인 [http://numbersapi.com/](http://numbersapi.com/)에 연결하여 대한 흥미로운 수치적 사실을 찾습니다. axios 라이브러리는 바이너리 파일이 아니기 때문에 컴파일 없이 Graviton2에서 원활하게 작동할 수 있습니다.

기존 애플리케이션은 Lambda 함수를 호출하고 수치적 사실을 검색하며 응답을 반환하는 API 엔드포인트로 구성됩니다.

AWS SAM을 사용하여 기존 x86 함수 버전을 빌드합니다.
```
sam build
```
![sam build](/aws-lambda/img/sambuild.png)

함수를 AWS 계정에 배포합니다:
```
sam deploy --stack-name GravitonLambdaNumber -g
```
초기 기본값을 수락하고 Y를 입력합니다: `LambdaNumberFunction may not have authorization defined, Is this okay? [y/N]: y`

![sam deploy -g](/aws-lambda/img/samdeploy-g.png)

AWS SAM은 인프라를 배포하고 APIBasePath를 출력합니다.

![ApiBasePath](/aws-lambda/img/ApiBasePath.png)

`curl`을 사용하여 날짜를 숫자로 하여 APIBasePath URL에 POST 요청을 합니다.
```
curl -X POST https://6ioqy4z9ee.execute-api.us-east-1.amazonaws.com -H 'Content-Type: application/json' -d '{ "number": "345", "type": "date" }'
```
Lambda 함수는 x64 프로세서 아키텍처와 날짜에 대한 팩트로 응답해야 합니다.

![curl x86](/aws-lambda/img/curlx86.png)

AWS SAM [template.yml](.GravitonLambdaNumber/template.yml) 파일에서 프로세서 아키텍처를 수정합니다.
아래를 
```
Architectures:
- x86_64
```        
아래로 변경합니다
```
Architectures:
- arm64
```        

arm64 아키텍처를 사용하여 기능을 구축하고 변경 사항을 클라우드에 배포합니다.
```
sam build
sam deploy
```
동일한 `curl` 명령을 사용하여 숫자를 날짜로 하여 APIBasePath URL에 POST 요청을 합니다.
```
curl -X POST https://6ioqy4z9ee.execute-api.us-east-1.amazonaws.com -H 'Content-Type: application/json' -d '{ "number": "345", "type": "date" }'
```
Lambda 함수는 `arm64` 프로세서 아키텍처와 날짜에 대한 팩트로 응답해야 한다.

![curl arm64](/aws-lambda/img/curlarm64.png)

함수가 x86에서 arm64로 원활하게 마이그레이션되었습니다.

### 바이너리 종속성이 있는 Lambda 함수 빌드 하기

이 함수는 바이너리 종속성이 없습니다. arm64에 대해 컴파일이 필요한 기능이 있는 경우 AWS SAM은 arm64 빌드 컨테이너 이미지를 사용하여 arm64에 대한 아티팩트를 생성할 수 있습니다.
이 기능은 두 가지 방식으로 작동합니다. x86 워크스테이션에서 arm64 특정 종속성을 구축하고 arm64 워크스테이션에서 x86 특정 종속성을 구축할 수 있습니다.

빌드 컨테이너를 사용하려면 `--use-container`를 지정하십시오.
```
sam build --use-container
```
![sam build --use-container](/aws-lambda/img/sambuildcontainer.png)

### 로컬 테스트
AWS SAM 또는 Docker를 기본적으로 사용하여 arm64 기능을 로컬로 테스트할 수 있습니다.

AWS SAM을 사용할 때 [`sam local invoke`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-invoke.html)를 사용하여 로컬에서 기능을 테스트하고 샘플 `event.json`을 전달할 수 있습니다.
```
sam local invoke LambdaNumberFunction -e ./test/event.json
```
![sam local invoke](/aws-lambda/img/samlocalinvoke.png)

### arm64 Lambda 함수를 컨테이너 이미지로 빌드하기
arm64 함수를 컨테이너 이미지로 빌드할 수 있습니다. [AWS SAM natively]([.src/Dockerfile])(https://aws.amazon.com/blogs/compute/using-container-image-support-for-aws-lambda-with-aws-sam/)를 사용하여 컨테이너 이미지를 작성할 수 있습니다.

AWS SAM 대신 Docker 네이티브 명령을 사용할 수도 있습니다.
Docker를 사용하여 이 함수의 컨테이너 이미지를 작성하려면 [Dockerfile](.src/Dockerfile)을 사용하고 'nodejs:12-arm64' 기본 이미지를 지정합니다.
```
FROM public.ecr.aws/lambda/nodejs:12-arm64
COPY app.js package*.json $LAMBDA_TASK_ROOT
RUN npm install
CMD [ "app.lambdaHandler" ]
```
컨테이너 이미지를 빌드합니다.
```
docker build -t dockernumber-arm ./src 
```
![docker build](/aws-lambda/img/dockerbuild.png)

컨테이너 이미지를 검사하여 `Architecture`를 확인합니다.
```
docker inspect dockernumber-arm | grep Architecture 
```
![docker inspect](/aws-lambda/img/dockerinspect.png)

`docker run`을 사용하여 로컬에서 기능을 테스트할 수 있습니다.
다른 터미널 창에서 Lambda 함수 도커 컨테이너 이미지를 실행합니다.
```
docker run -p 9000:8080 dockernumber-arm:latest
```
![docker run](/aws-lambda/img/dockerrun.png)

로컬 도커 엔드포인트를 사용하여 Lambda 함수를 호출하고 샘플 'event.json'을 전달합니다.
```
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d @./test/event.json
```
![docker run response](/aws-lambda/img/dockerrunresponse.png)

`도커 실행` 터미널 창에서 로컬 로그를 볼 수 있습니다.

로컬 이미지에서 Lambda 함수를 생성하려면 먼저 [Amazon Elastic Container Registry(ECR)](https://aws.amazon.com/ecr/) 저장소를 생성하고 ECR에 로그인합니다.
AWS 계정 번호 '123456789012'와 AWS Region 값을 본인 계정 정보로 바꿉니다.
```
aws ecr create-repository --repository-name dockernumber-arm --image-scanning-configuration scanOnPush=true
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```
태그를 지정하고 도커 이미지를 ECR에 푸시합니다.
```
docker tag dockernumber-arm:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/dockernumber-arm:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/dockernumber-arm:latest
```
그런 다음 AWS Management Console, AWS CLI 또는 기타 도구를 사용하여 컨테이너 이미지에서 Lambda 함수를 생성할 수 있습니다.
![Create function from image](/aws-lambda/img/createfunctionfromimage.png)

## x86과 arm64의 성능과 비용 비교
[AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) 오픈 소스 프로젝트를 사용하여 비용을 최소화하고 성능을 극대화하기 위한 구성을 제안할 수 있습니다. 이 도구를 사용하면 동일한 차트에서 두 결과를 비교하고 arm64 기반 가격을 통합할 수 있습니다. 이는 동일한 기능의 두 버전을 비교하는 데 유용합니다. 하나는 x86을 사용하고 다른 하나는 arm64를 사용합니다.

데모 애플리케이션은 소수를 계산합니다. AWS SAM [template.yml](template.yml) 파일에는 x86용과 arm64용 등 두 가지 Lambda 함수가 포함되어 있습니다. 두 함수 모두 [`./src/app.py`](./src/app.py)에서 동일한 Python 소스 코드를 사용하여 소수점을 계산합니다.

리포지토리가 이전 데모에서 복제되고 디렉토리로 변경되었는지 확인합니다.
```
cd ../PythonPrime
```
AWS SAM 빌드 컨테이너 이미지 내에 애플리케이션을 빌드하고 클라우드에 배포합니다.
기본 `sam deploy` 프롬프트를 사용합니다.
```
sam build --use-container
sam deploy --stack-name PythonPrime -g
```
두 Lambda 함수의 출력 값을 확인하세요:
![Prime Functions](/aws-lambda/img/primefunctions.png)

### AWS Lambda Power Tuning State Machine 배포
[https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:451282441545:applications~aws-lambda-power-tuning](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:451282441545:applications~aws-lambda-power-tuning)으로 이동하여 **Deploy*를 선택합니다.
*I acknowledge that this app creates custom IAM roles*을 선택하고 **Deploy**를 선택합니다,

구축 후 *Resources*에서 *powerTuningStateMachine*를 선택합니다.

**Start execution**을 선택합니다.
상태 기계 입력의 경우 AWS SAM Outputs에서 x86 Lambda 함수의 ARN을 지정하고 메모리 `powerValues`를 점검하도록 설정합니다. Lambda 기능 페이로드를 지정하여 각 메모리 구성에 대해 10번의 호출을 병렬로 실행합니다.
다음은 입력 예입니다.
```
{
	"lambdaARN": "arn:aws:lambda:us-east-1:123456789012:function:gravitonpythonprime-PythonPrimeX86Function-3Gge9WXmHObZ",
	"powerValues": [
		128,
		256,
		512,
		1024,
		1536,
		2048,
		3008,
		10240
	],
	"num": 10,
	"parallelInvocation": true,
	"payload": {
		"queryStringParameters": {
			"max": "1000000"
		}
	}
}
```
*Open in a new browser tab*를 선택하고 **Start execution**을 선택합니다.
Lambda 전력 튜닝 상태 시스템은 구성된 각 메모리 값에 대해 실행됩니다.
![PowerTuningStateMachine](/aws-lambda/img/powertuningstatemachine.png)

완료되면 *Execution event history* 마지막 단계, *ExecutionSucceeded*에는 시각화 URL이 포함되어 있습니다.
```
{
  "output": {
    "power": 512,
    "cost": 0.000050971200000000004,
    "duration": 6067.418333333332,
    "stateMachine": {
      "executionCost": 0.00035,
      "lambdaCost": 0.007068295500000001,
      "visualization": "https://lambda-power-tuning.show/#gAAAAQACAAQABgAIwAsAKA==;5njERiCGREZZm71F9rxARW12AkU66d5EDqzeRLFs30Q=;a4NdODSTXTjpyVU42k9ZOLixXDipans4V224ONx8nTk="
    }
  },
  "outputDetails": {
    "truncated": false
  }
}
```
Lambda 파워 튜닝 URL로 이동하여 x86 함수의 각 메모리 값에 대한 평균 *호출 시간(ms)* 및 *호출 비용(USD)*을 확인합니다.

![PowerTuning x86 results](/aws-lambda/img/powertuningx86results.png)

Step Functions 콘솔로 돌아가서 AWS SAM Outputs에서 arm64 Lambda 기능의 ARN을 지정하여 다른 state machine을 실행합니다.
완료되면 *Execution event history* 마지막 단계인 *ExecutionSucceeded*에서 시각화 URL을 복사합니다.

x86 결과를 보여주는 *lambda-power-tuning* 브라우저 탭에서 **Compare**를 선택합니다.
함수 1의 이름으로 **x86**를 입력하십시오.
함수 2의 이름으로 **arm64**를 입력합니다.
arm64 함수의 URL에 붙여넣고 **비교**를 선택합니다.

![PowerTuning Compare](/aws-lambda/img/powertuningcompare.png)

*x86*과 *arm64* 기능의 비교를 봅니다.

![PowerTuning Comparison](/aws-lambda/img/powertuningcomparison.png)

2048MB에서는 arm64 기능이 x86에서 실행되는 동일한 Lambda 기능보다 29% 더 빠르고 43% 더 저렴합니다!
전력 튜닝은 Lambda 기능에 대한 최적의 메모리 구성을 선택하는 데이터 기반 접근 방식을 제공합니다. 이를 통해 x86과 arm64를 비교할 수도 있고 arm64 Lambda 기능의 메모리 구성을 줄여 비용을 더욱 절감할 수도 있습니다.
