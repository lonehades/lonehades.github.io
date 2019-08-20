---
title: AI platform setting
description: docker 를 활용해서 Google Cloud Platform 등 다양한 환경에서 동일한 AI 개발 플랫폼 세팅
categories:
 - tutorial
tags: AI, GPU, GCP, Docker, Tensorflow, Deep learning, deepo
---

# 설치 요약
* GPU 서버 준비 또는 구글 GCP 가입 및 GPU 쿼터 얻기
* 우분투 18.04 설치
* NDVIA 그래픽 드라이버 설치
* NDVIA GPU 용 Docker 설치
* Tensorflow 또는 ufoym/deep 등의 AI 개발 컨테이너 이미지 가져오기
* 설치 후 서버 성능 측정


# 구글 클라우드 가입 및 GPU 쿼터 얻기
* 구글 클라우드 가입 및 기본적인 설정, GPU 쿼터 신청, VM 실행하기
 * 테리의 구글 클라우드로 딥러딩 시작하기:  https://www.youtube.com/watch?v=d4mz9YIf6Gc&list=PL0oFI08O71gKEXITQ7OG2SCCXkrtid7Fq&index=26&t=0s
 * mc.ai, GCP VM 에서 NVIDA GPU를 사용해보자:  https://mc.ai/gcp-vm%EC%97%90%EC%84%9C-nvidia-gpu%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EC%9E%90/
 * 위 mc.ai에서  구글 클라우드 설정 및 쿼터 신청에 더 자세한 내용 참조
* 가입하기: https://cloud.google.com/ 
* try free: $300 쿼터 기본 제공
* IAM&admin, Upgrade account, GPU 사용 신청 (2일) => 기본은 0
* 구글은 성능 순으로 K80<P100<V100 등의 GPU 가능
![](/assets/20190820/GPUK80P100V100.png)
 * 참고로 자체 보유하고 있는 서버: M40 25G x 2
* k80은 테스트용, v100 은 실제 러닝용으로 계획
* K80<M40<P100<V100
* 구글 쿼터 승인:
승인에 2일 정도 걸렸고, 급하다고 했더니 좀 더 빨리 해준 느낌. 위 블로그에선 $70 을 요청한다고 했는데 그냥 승인해줌.
승인신청시에 medical imaging superresolution 프로젝트를 위해 GPU 가 많이 필요하다 등으로 영문 요청 사항을 기재했음.GPU all regions => 16 으로 별도로 신청해서 승인 받음
Compute Engine 에서 VM Images 선택
CPU, GPU 설정 (GPU 쿼터가 없으면 인스탄스가 안만들어짐)
코딩 개발용 (저렴, Preemtible), 실제 훈련용 (고급) 으로 나누면 좋음
구글 드라이브 등을 활용하려면 Allow full access to all Cloud APIs 체크
주피터 등을 활용하려면 HTTP traffic, HTTPS traffic 모두 체크
Preemtibility on 하면 반값
초록색: VM 켜져 있음. 항상 과금.
회색: VM 현재 꺼져 있음. HDD/SDD 만 과금!
GCP 방화벽 규칙 생성: Jupyter Notebook 과 Tensorboard 를 사용하기 위해 포트 접근 허용네트워킹>VPC 네트워크>방화벽 규칙

# Docker를 활용해서 AI 플랫폼 구축


* 출처: UBUNTU 18.04 설치 #2-3 INSTALL TENSORFLOW WITH DOCKER: https://eungbean.github.io/2018/11/09/Ubuntu-Installation2-3/

##### OS 설치 또는 VM 설정

* Ubuntu 18.04 설치:  https://eungbean.github.io/2018/08/08/Ubuntu-Installation1/     
* Google Cloud Platform VM 설치시:  https://mc.ai/gcp-vm%EC%97%90%EC%84%9C-nvidia-gpu%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EC%9E%90/           

    * -네트워킹 > VPC 네트워크 > 방화벽 규칙     
    1) 이름: jupyter
    2) 대상 태그: jupyter                 
    3) 소스 IP 범위: 0.0.0.0/0                 
    4) 프로토콜 및 포트: tcp:8888                 
    
    1) 이름: tensorboard                 
    2) 대상 태그: tensorboard                 
    3) 소스 IP 범위: 0.0.0.0/0                 
    4) 프로토콜 및 포트: tcp:6006                 
    -VM 설치 예시                 
    1. 이름: v100–4ea (비싸진다. 옆에 시간당, 월당 예상 요금이 나옴)                 
    2. 지역: us-west1                 
    3. 영역: us-west-1-b                 
    4. (머신 유형 맞춤설정) 코어: 24 vCPU                 
    5. (머신 유형 맞춤설정) 메모리: 90 GB                 
    6. (머신 유형 맞춤설정) GPU 개수: 4                 
    7. (머신 유형 맞춤설정) GPU 유형: NVIDIA Tesla V100                 
    8. 부팅 디스크: 100GB SSD 영구 디스크 Ubuntu 18.04 LTS                 
    9. 방화벽: HTTP, HTTPS 트래픽 허용 선택                 
    10. (네트워킹) 네트워크 태그: jupyter, tensorboard (필수)                 
    11. (관리)선점 가능성: 사용안함 (선택사항)        

##### python3 과 pip3 설치
```
      sudo apt update      
      sudo apt-get install gnome-tweak-tool      
      sudo apt update
      sudo apt install python3-pip      
      sudo apt install build-essential python3-dev  python3-setuptools      
      pip3 --version               # 설치 확인
```

##### Graphic Driver 설치:  https://www.linuxbabe.com/ubuntu/install-nvidia-driver-ubuntu-18-04
* ubuntu-drivers 가 실행이 안될때 (구글 클라우드)
  sudo apt install ubuntu-drivers-common
* GPU 종류와 CUDA 킷에 따라 적절한 그래픽 driver 를 설치 하는 것이 중요
* ufoym/deepo 는 18.04 에서 CUDA 10.0 이 필요해서 여기에 맞는 driver 를 검색:
  https://www.nvidia.com/Download/index.aspx?lang=en-us
  docker 에서 실제 설치된 버전을 확인해보니 CUDA 9.0 임
  root@tf3j:~# nvcc --version
  nvcc: NVIDIA (R) Cuda compiler driver            
   Copyright (c) 2005-2017 NVIDIA Corporation            
   Built on Fri_Sep__1_21:08:03_CDT_2017            
   Cuda compilation tools, release 9.0, V9.0.176
* tensorflow docker image 는 CUDA 9.0 이 필요함.
* 회사 서버 Tesla M40 은 위 두 경우 모두 nvidia driver 410.xxx 를 깔면 해결 됨.
  아래엔 Tesla GPU 의 경우 10.0은 >=384.111, <385.00 으로 되어 있지만 410 을 설치해도 잘됨.
  https://github.com/NVIDIA/nvidia-docker/wiki/CUDA
sudo add-apt-repository ppa:graphics-drivers/ppa
최신 드라이버 설치 법:
sudo ubuntu-drivers devices
sudo apt install nvidia-driver-version-number
(ex. sudo apt install nvidia-driver-410)		
sudo shutdown -r now
* 결론 회사 서버 M40 및 GCP는  nvidia-driver-410 으로 설치

##### Docker 설치:
* 초보를 위한 안내문 도커란 무엇인가?: https://subicura.com/2017/01/19/docker-guide-for-beginners-1.html
```
    #Update the apt package index: 
    sudo apt-get update
    #Install packages to allow apt to use a repository over HTTPS:
    sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
    #Add Docker’s official GPG key:
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    #Verify that you now have the key with the fingerprint: #9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88 
    sudo apt-key fingerprint 0EBFCD88
    #set up the stable repository sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
    #Update the apt package index:
    sudo apt-get update
    #Install the latest version of Docker CE
    sudo apt-get install docker-ce
    #Verify that Docker CE is installed correctly by running the hello-world image.
    sudo docker run hello-world
```

*  sudo 없이 사용하기 (ssh 재접속해야 적용)
```
   sudo usermod -aG docker $USER                      
   # 현재 접속중인 사용자에게 권한주기
   sudo usermod -aG docker your-user
   # your-user 사용자에게 권한주기           
```   

##### Nvidia-docker 설치                        
```
	# If you have nvidia-docker 1.0 이 설치된 경우만: we need to remove it and all existing GPU containers
    docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 
    docker ps -q -a -f volume={} | xargs -r 
    docker rm -f
    sudo apt-get purge -y nvidia-docker
    # Add the package repositories
    curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \sudo apt-key add -distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
    curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \sudo tee /etc/apt/sources.list.d/nvidia-docker.list
    sudo apt-get update
    # Install nvidia-docker2 and reload the Docker daemon configuration
    sudo apt-get install -y nvidia-docker2
    sudo pkill -SIGHUP dockerd
    # Test nvidia-smi with the latest official CUDA image
    sudo docker run --runtime=nvidia --rm nvidia/cuda:9.0-base nvidia-smi                                   
```
##### ufoym/deepo 컨테이너 이미지 당겨오기

```
	- Jupyter Notebook 버전
	
	#jupyer가 지원되는 이미지 불러오기
    docker pull ufoym/deepo:all-py36-jupyter
    
    #공유 폴더만들기sudo mkdir /docker/data
    
    #Jupyter settingjupyter notebook --generate-config (노트북 실행시 패스워드가 필요한 경우)
    
    In [1]: from notebook.auth import passwd
    In [2]: passwd()
    Enter password:
    Verify password:
    Out[2]: 'sha1:f24어쩌고아주긴코드생성'
    
    -in config file: jupyter_notebook_config.py 편집
    
    c = get_config()
    c.NotebookApp.password='sha1:f24위에서생성한코드'
    
    -config 파일 복사
    cp .jupyter/jupyter_notebook_config.py data/
    
    #이미지 실행하기
    nvidia-docker run -it \
    -p 8888:8888 \
    -p 6006:6006 \
    -h tf3j \
    --name tf3j \-v /docker/data:/root/data \
    --ipc=host --rm ufoym/deepo:all-py36-jupyter \
    jupyter notebook --no-browser \
    --ip=0.0.0.0 \
    --allow-root \
    --config /root/data/jupyter_notebook_config.py \
    --NotebookApp.token= --notebook-dir='/root'
    
    #쥬피터 노트북으로 접속하기 (여러개의 쥬피터 노트북을 실행하려면 다른 포트로 바인딩 필요)http://서버외부IP:8888 (구글은 해당 VM 외부 접속 IP:8888)
    
    #실행중인 docker 컨테이너에 접속하기 (다른 터머널로)
    docker exec -it tf3j /bin/bash
    
    #CUDA 버전 확인                      
    nvcc --version
    
    #CUDNN 버전 확인 (ufoym/deepo 는 CUDNN 설치가 안되어 있음)
    cat /usr/local/cuda/include/cudnn.h | grep CUDNN_MAJOR -A 2

```

##### 설치 후 AI 서버 성능측정

* 출처:  https://hiseon.me/2018/06/23/tensorflow-benchmark/
* tensorflow 가 최신 버전이 아닌 경우 (nightly version) 로 가져와야 실행됨
![](http://assets/20190820/tensorflow_tag.png)
```
	docker pull tensorflow/tensorflow:nightly-gpu-py3-jupyter
    nvidia-docker run -it -v /docker/data:/root/data --rm tensorflow/tensorflow:nightly-gpu-py3-jupyter bash
    git clone https://github.com/tensorflow/benchmarks.git
```    
* GPU 2개로 resnet 152 테스트:
```
    python scripts/tf_cnn_benchmarks/tf_cnn_benchmarks.py --num_gpus=2 --nodistortions --model=resnet152 --batch_size=32 --num_warmup_batches=4 --forward_only=True
```    
* CPU 모드로 resnet 152 테스트:
```
	python scripts/tf_cnn_benchmarks/tf_cnn_benchmarks.py --device=cpu --nodistortions --model=resnet152 --data_format=NHWC --batch_size=32 --num_warmup_batches=4 --forward_only=True
```    
* 참고 테스트 데이터 ResNet152

|               AI 서버 환경        | 초당 처리능력 (image/sec) | 사용료    |
|----------------------------------|-------------------------|----------|
|Notebook X1 i7-6600U 2.6G (window)|            3.23         |  무료     |
|HC Server CPU Xeon E5-2650 2.2G   |            41.74        |  무료     |
|HC Server M40 GPU x2              |            260.81       |  무료     |
|GCP K80 x2                        |            115.10       |$0.469/시간|
|GCP V100  x4                      |            976.13       |$7.805/시간|

* window 에서 docker 설치 및 실행Git bash 에서 실행할때:
	winpty docker run -it --rm tensorflow/tensorflow:nightly-py3 bash

##### GPU 메모리 모니터링



* 메모리 때문에 주피터 커널이 죽는 경우가 발생.
* nvtop 으로 메모리 모니터링 및 관리
* 출처:    https://github.com/Syllo/nvtop#nvtop-build
```
	#docker 컨태이너로 접속
	apt-get update
	apt install cmake libncurses5-dev libncursesw5-dev git

	git clone https://github.com/Syllo/nvtop.git
	mkdir -p nvtop/build && cd nvtop/build
	cmake ..

	# If it errors with "Could NOT find NVML (missing: NVML_INCLUDE_DIRS)"
	# try the following command instead, otherwise skip to the build with make.
	cmake .. -DNVML_RETRIEVE_HEADER_ONLINE=True

	make
	make install # You may need sufficient permission for that (root)

	copy nvtop => /root/data
```