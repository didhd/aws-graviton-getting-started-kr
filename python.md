# Graviton 에서의 Python

Python(Python)은 인터프리터, 고수준, 범용 프로그래밍 언어이며, arm64를 포함한 많은 운영 체제와 아키텍처에서 사용할 수 있습니다. _[위키피디아에서 더 읽기](https://en.wikipedia.org/wiki/Python_(programming_language))_

## 1. Python 패키지 설치

*pip*(Python의 표준 패키지 설치 프로그램)을 사용하면 [Python Package Index](https://pypi.org) 및 기타 인덱스에서 패키지를 가져옵니다. [Python Package Index](https://pypi.org)에서 바이너리 설치를 보장하기 위해, pip를 새 버전 \(>19.3\)으로 업데이트 하세요.

```
# To ensure an up-to-date pip version
sudo python3 -m pip install --upgrade pip
```

AWS는 사전에 컴파일된 패키지를 Graviton에서 사용할 수 있도록 적극적으로 노력하고 있다. 200개 이상의 인기 있는 Python 패키지의 현재 목록을 [Python wheel tester](https://geoffreyblake.github.io/arm64-python-wheel-tester/)에서 AL2 및 Ubuntu 의 Graviton 지원 상태를 확인할 수 있습니다.

*pip*가 사전에 컴파일된 패키지를 찾을 수 없는 경우 소스 코드에서 패키지를 자동으로 다운로드, 컴파일 및 빌드합니다.
일반적으로 소스코드에서 패키지를 설치하는 데 사전 빌드된 것보다 몇 분 정도 더 걸릴 수 있습니다. 일부 대형 패키지의 경우, 최대 20분 정도 걸릴 수 있습니다. 경우에 따라 종속성이 누락되어 컴파일이 실패할 수 있습니다. 
최신 버전이 아닌 wheel 설치를 시도하기 위해 Python 패키지를 원본에서 빌드하기 전에 다음을 시도해 보십시오 `python3 -m pip install --prefer-binary <package>`. 가끔 자동화된 패키지 빌더는 나중에 수정될 빌드의 고장으로 인해 모든 휠 없이 릴리스를 푸시하기도 합니다. 해당 옵션을 선택할 수 없는 경우 다음 지침을 따라 소스로부터 Python 패키지를 빌드하세요.

### 1.1 소스에서 Python 패키지를 설치하기 위한 필수 구성 요소

소스 코드에서 일반적인 Python 패키지를 설치하려면 다음과 같은 개발 도구를 설치해야 합니다:

**AmazonLinux2 또는 RedHat**:
```
sudo yum install "@Development tools" python3-pip python3-devel blas-devel gcc-gfortran lapack-devel
python3 -m pip install --user --upgrade pip
```

**Debian/Ubuntu**:
```
sudo apt update
sudo apt-get install build-essential python3-pip python3-dev libblas-dev gfortran liblapack-dev
python3 -m pip install --user --upgrade pip
```

모든 배포판에서, 설치하려는 Python 모듈에 따라 추가 컴파일 시간이 필요할 수 있습니다.

### 1.2 권장 버전

Graviton2 를 채택할 때는 가급적 최신 소프트웨어 버전을 사용하는 것이 좋으며 Python도 예외는 아닙니다.

Python 2.7은 2020년 1월 1일부터 EOL로, Graviton2로 이동하기 전에 Python 3.x 버전으로 업그레이드하는 것이 확실히 권장됩니다.

Python 3.6은 [2021년 12월 EOL](https://www.python.org/dev/peps/pep-0494/#2011)에 도달할 것이므로, Graviton2에 애플리케이션을 이식하기 시작할 때는 적어도 Python 3.7을 대상으로 하는 것이 좋습니다.

### 1.3 AL2 과 RHEL 8 에서의 Python

AL2와 RHEL 8은 각각 3.7과 3.6 버전의 오래된 Python을 기본적으로 배포합니다. Python 3.6은 [2021년 12월 이후](https://endoflife.date/python) EOL이고 Python 3.7은 [2023년 6월](https://endoflife.date/python) EOL이 됩니다. 따라서 일부 패키지 메인테이너들은 이미 [pypi.org](https://pypi.org)에 게시된 사전 제작 wheel을 생략함으로써 Python 3.6 및 3.7에 대한 지원을 중단하기 시작했습니다. 일부 패키지의 경우 패키지 관리자의 배포를 사용하여 기본 Python을 사용할 수 있습니다. 예를 들어 `numpy`는 더 이상 Python 3.6 wheel을 게시하지 않지만 패키지 관리자 `yum install python3-numpy`에서 설치할 수 있습니다.

또 다른 옵션은 기본 Python 패키지 대신 Python 3.8을 사용하는 것입니다. 다음 명령어로 Python 3.8 및 pip를 설치할 수 있습니다: `yum install python38-pip`. 그런 다음 `pip3 install numpy`를 사용하여 최신 버전의 패키지를 설치하세요. AL2에서는 Python 3.8 패키지를 사용하기 위해 `amazon-linux-extra enable python3`을 사용해야 합니다.

패키지 관리자가 배포하는 일반적인 Python 패키지는 다음과 같습니다:
1. python3-numpy
2. python3-markupsafe
3. python3-pillow

전체 목록 실행을 보려면: `yum search python3`


## 2. 과학 및 수치 애플리케이션 (NumPy, SciPy, BLAS, etc)

Python은 고성능을 달성하기 위해 네이티브 코드에 의존합니다. 과학 및 수치 애플리케이션을 위해 NumPy와 SciPy는 ATLAS, BLAS, BLIS, OpenBLAS와 같은 고성능 컴퓨팅 라이브러리에 대한 인터페이스를 제공합니다. 이러한 라이브러리에는 Graviton 프로세서용으로 조정된 코드가 포함되어 있습니다.

가능한 한 최신 소프트웨어 버전을 사용하는 것이 좋습니다. 최신 버전 마이그레이션이 가능하지 않은 경우, aarch64의 데이터 정밀도 및 정확성과 관련된 여러 수정 사항이 OpenBLAS에 들어갔기 때문에 적어도 권장되는 최소 버전인지 확인하십시오.

OpenBLAS:  >= v0.3.17
SciPy: >= v1.7.2
NumPy: >= 1.21.1

[SciPy>=1.5.3](https://pypi.org/project/scipy/1.5.3/#files)와 [NumPy>=1.19.0](https://pypi.org/project/numpy/1.19.0/#files)는 Aarch64용 바이너리 wheel 패키지를 제공하지만, 더 나은 성능이 필요하다면 최고의 성능 수치 라이브러리를 컴파일하는 것도 옵션이 될 수 있습니다. 이렇게 하시려면 다음 지침을 따르세요.

### 2.1 BLIS가 BLAS보다 더 빠를 수 있습니다

`pip3 install numpy scipy`로부터 설치된 SciPy 및 NumPy 바이너리는 OpenBLAS를 사용하도록 구성되어 있습니다. SciPy와 NumPy의 기본 구성은 설정하기 쉽고 테스트되었습니다.

일부 워크로드는 BLIS를 사용하면 이점을 얻을 수 있습니다. BLIS를 사용하여 SciPy 및 NumPy 워크로드를 벤치마킹하면 추가적인 성능 향상을 확인할 수 있습니다.


### 2.2 Ubuntu 및 Debian에 NumPy 및 SciPy와 BLIS 설치

우분투와 데비안 `apt install python3-numpy python3-scipy`는 BLAS와 LAPACK 라이브러리와 함께 NumPy와 SciPy를 설치할 것입니다. BLIS 및 OpenBLAS와 함께 SciPy 및 NumPy를 설치하려면:
```
sudo apt -y install python3-scipy python3-numpy libopenblas-dev libblis-dev
sudo update-alternatives --set libblas.so.3-aarch64-linux-gnu \
    /usr/lib/aarch64-linux-gnu/blis-openmp/libblas.so.3
```

사용 가능한 대체 항목 간에 전환하려면:

```
sudo update-alternatives --config libblas.so.3-aarch64-linux-gnu
sudo update-alternatives --config liblapack.so.3-aarch64-linux-gnu
```

### 2.3 Amazon Linux2(AL2) 및 RedHat에 NumPy 및 SciPy(BLIS 포함) 설치

arm64 AL2 및 Red Hat에서 BLIS를 사용하여 SciPy 및 NumPy를 빌드하기 위한 전제 조건:
```
# AL2/RedHat 설치 필수 구성 요소
sudo yum install "@Development tools" python3-pip python3-devel blas-devel gcc-gfortran

# BLIS 설치
git clone https://github.com/flame/blis $HOME/blis
cd $HOME/blis;  ./configure --enable-threading=openmp --enable-cblas --prefix=/usr cortexa57
make -j4;  sudo make install

# OpenBLAS 설치
git clone https://github.com/xianyi/OpenBLAS.git $HOME/OpenBLAS
cd $HOME/OpenBLAS
make -j4 BINARY=64 FC=gfortran USE_OPENMP=1 NUM_THREADS=64
sudo make PREFIX=/usr install
```

BLIS 및 OpenBLAS를 사용하여 NumPy 및 SciPy를 빌드하고 설치하려면:
```
git clone https://github.com/numpy/numpy/ $HOME/numpy
cd $HOME/numpy;  pip3 install .

git clone https://github.com/scipy/scipy/ $HOME/scipy
cd $HOME/scipy;  pip3 install .
```

NumPy와 SciPy는 빌드 시 BLIS 라이브러리의 존재를 감지하면 BLAS와 OpenBLAS의 동일한 기능보다 BLIS를 우선하여 사용합니다. LAPACK 기능을 제공하려면 OpenBLAS 또는 LAPACK 라이브러리를 BLIS와 함께 설치해야 합니다. 라이브러리 의존성을 변경하려면 numpy와 scipy를 빌드하기 전에 환경 변수 `NPY_BLAS_ORDER`와 `NPY_LAPACK_ORDER`를 설정할 수 있습니다. 기본값은 다음과 같습니다:
`NPY_BLAS_ORDER=mkl,blis,openblas,atlas,accelerate,blas` and
`NPY_LAPACK_ORDER=mkl,openblas,libflame,atlas,accelerate,lapack`.

### 2.4 NumPy 및 SciPy 설치 테스트

설치된 NumPy 및 SciPy가 BLIS 및 OpenBLAS로 구축되었는지 테스트합니다. 다음 명령은 기본 라이브러리 종속성을 보여줍니다:
```
python3 -c "import numpy as np; np.__config__.show()"
python3 -c "import scipy as sp; sp.__config__.show()"
```

Ubuntu와 Debian의 경우 이 명령어는 `update-alternative`에 의해 관리되는 심볼릭 링크인 `blas`와 `lapack`을 출력합니다.

### 2.5 멀티 스레딩을 통한 BLIS 및 OpenBLAS 성능 개선

OpenBLAS가 `USE_OPENMP=1`로 빌드되었을 때 OpenMP를 사용하여 연산을 병렬화합니다.
환경 변수 `OMP_NUM_THREADS`를 설정하여 최대 스레드 수를 지정할 수 있습니다. 
이 변수가 설정되지 않은 경우 기본적으로 단일 스레드를 사용합니다.

BLIS와 병렬성을 사용하려면 `--enable-threading=openmp`로 구성하고 환경 변수 `BLIS_NUM_THREADS`를 사용할 스레드 수로 설정해야 합니다. 기본값은 단일 스레드를 사용하는 것입니다.

### 2.6 Conda / Anaconda에서 Graviton 지원
Anaconda는 과학 계산을 위한 Python과 R 프로그래밍 언어의 배포판으로, 패키지 관리와 배포를 단순화하는 것을 목표로 합니다.

Anaconda는 [AWS Graviton2 2021년 5월 14일 지원](https://www.anaconda.com/blog/anaconda-aws-graviton2)을 발표했습니다.

전체 Anaconda 패키지 설치 프로그램을 설치하는 방법은 https://docs.anaconda.com/anaconda/install/graviton2/ 에서 확인할 수 있습니다.

Anaconda는 또한 [Miniconda](https://docs.conda.io/en/latest/miniconda.html)라는 가벼운 Anaconda의 부트스트랩 버전을 제공하는데, 버전에는 conda, Python, 의존성 패키지 및 pip, zlib 및 기타 몇 가지 유용한 패키지들이 포함되어 있습니다.

다음은 이 프로그램을 사용하여 Python 3.9를 위해 [numpy](https://numpy.org/) 및 [pandas](https://pandas.pydata.org/)을 설치하는 방법에 대한 예입니다.

첫 번째 단계는 conda를 설치하는 것입니다:
```
$ wget https://repo.anaconda.com/miniconda/Miniconda3-py39_4.10.3-Linux-aarch64.sh
$ chmod a+x chmod a+x Miniconda3-py39_4.10.3-Linux-aarch64.sh
$ ./Miniconda3-py39_4.10.3-Linux-aarch64.sh
```

설치가 완료되면 `conda` 명령을 사용하여 패키지를 직접 설치하거나 환경 정의 파일을 작성하여 해당 환경을 만들 수 있습니다.

다음은 [numpy](https://numpy.org/) 및 [pandas](https://pandas.pydata.org/) (`graviton-example.yml`)을 설치하는 예입니다.
```
name: graviton-example
dependencies:
  - numpy
  - pandas
```

다음 단계는 해당 정의에서 환경을 인스턴스화하는 것입니다:
```
$ conda env create -f graviton-example.yml
```

## 4. 머신 러닝 Python 패키지


### 4.1 PyTorch

```
pip install numpy
pip install torch torchvision
```
### 4.2 TensorFlow

```
pip install tensorflow-cpu-aws
```
### 4.3 DGL

Pytorch가 설치되어 있는지 확인하십시오. 설치되어 있지 않은 경우 [Pytorch 설치 단계](#41-pytorch)를 따르십시오.

**Ubuntu**에서:

[install from source](https://github.com/dmlc/dgl/blob/master/docs/source/install/index.rst#install-from-source) 지침을 따릅니다.


### 4.4 Sentencepiece

[Sentencepiece>=1.94 now has pre-compiled binary wheels available for Graviton](https://pypi.org/project/sentencepiece/0.1.94/#history).

### 4.5	Morfeusz

**Ubuntu**에서:

```
# download the source
wget http://download.sgjp.pl/morfeusz/20200913/morfeusz-src-20200913.tar.gz
tar -xf morfeusz-src-20200913.tar.gz
cd Morfeusz/
sudo apt install cmake zip build-essential autotools-dev \
    python3-stdeb python3-pip python3-all-dev python3-pyparsing devscripts \
    libcppunit-dev acl  default-jdk swig python3-all-dev python3-stdeb
export JAVA_TOOL_OPTIONS=-Dfile.encoding=UTF8
mkdir build
cd build
cmake ..
sudo make install
sudo ldconfig -v
sudo PYTHONPATH=/usr/local/lib/python make install-builder
```

마지막 명령(_make install-builder_)에 문제가 있는 경우 시도해 보십시오:
```
sudo PYTHONPATH=`which python3` make install-builder
```
