# NVIDIA AGX Orin (Ubuntu 22.04)을 위한 ROS 2 Humble, PCL, VTK 통합 Docker 이미지 구축 및 활용

## I. 서론: Jetson AGX Orin과 JetPack 6 기반 컨테이너화 전략

### 목표 정의

이 보고서의 목표는 NVIDIA Jetson AGX Orin 플랫폼, 특히 JetPack 6 (Ubuntu 22.04 기반) 환경에서 1, Robot Operating System (ROS) 2 Humble, Point Cloud Library (PCL), 그리고 Visualization Toolkit (VTK)를 완벽하게 통합하는 Docker 컨테이너를 구축하고 활용하는 전문가 수준의 가이드를 제공하는 것이다.

### 핵심 과제

AGX Orin에서의 컨테이너화는 단순한 Dockerfile 작성을 넘어서는 고유한 기술적 과제를 내포한다. 이 작업은 다음 네 가지 핵심 과제를 동시에 해결해야 한다.

1. **아키텍처 호환성:** AGX Orin은 `aarch64` (ARM64) 아키텍처를 사용한다.2 모든 기본 이미지와 라이브러리는 `linux/arm64`와 호환되어야 하며, x86_64용 이미지를 사용하려는 시도는 `exec format error`를 유발한다.4
2. **L4T(Linux4Tegra) 결합성:** NVIDIA Jetson 플랫폼은 호스트 OS의 커널 및 드라이버와 컨테이너가 특수한 방식으로 연동된다.5 GPU, CUDA, TensorRT, 멀티미디어 가속기(GStreamer, VPI)에 접근하려면 단순한 `ubuntu:22.04` 이미지가 아닌, 호스트와 결합되도록 설계된 NVIDIA L4T 기본 이미지가 필수적이다.6
3. **복잡한 의존성:** ROS 2 Humble 8, PCL 9, VTK 10 스택은 상호 간에 복잡한 라이브러리 의존성을 갖는다. 특히 PCL의 시각화 모듈과 VTK 간의 버전 비호환성은 예고 없는 세그멘테이션 오류(Segmentation Fault)를 유발하는 주된 원인이다.11
4. **GUI 애플리케이션 연동:** VTK 기반의 PCL 시각화 도구 및 ROS의 RViz2는 GUI 애플리케이션이다. 격리된 Docker 컨테이너 환경에서 호스트(또는 원격) X 서버로 GUI를 안정적으로 포워딩하는 것은(X11 Forwarding) 그 자체로 복잡한 설정(Configuration)을 요구한다.13

### JetPack 6의 변화

JetPack 5 (Ubuntu 20.04)와는 달리, JetPack 6 (Ubuntu 22.04)는 더 이상 Docker를 사전 설치하여 제공하지 않는다.15 이는 개발자가 AGX Orin을 수령한 직후, Docker 및 NVIDIA Container Toolkit의 설치와 구성을 수동으로 직접 수행해야 함을 의미한다. 이는 사용자가 마주하는 첫 번째 장애물이다.

### 보고서의 접근 방식

본 문서는 AGX Orin 환경에서 검증된 최상의 솔루션(커뮤니티 기반 스크립트 활용)을 먼저 제시한다. 이후, 심층적인 이해와 고도의 맞춤 설정(Customizing)을 필요로 하는 전문가를 위해, 수동으로 Dockerfile을 작성하고 이미지를 빌드하며 실행하는 모든 과정을 상세히 분석하고 지침을 제공한다.

## II. 1단계: 호스트 시스템(AGX Orin) 준비 - Docker 및 NVIDIA Container Toolkit 설치

JetPack 6 (L4T R36.x) 환경은 Ubuntu 22.04를 기반으로 하지만 1, `nvidia-docker2` 및 `docker-ce` 패키지가 기본적으로 누락되어 있다.15 따라서 컨테이너 작업을 시작하기 전, 호스트 시스템에 Docker와 NVIDIA Container Toolkit을 올바르게 설치하는 것이 선행되어야 한다.

### Docker-CE 설치 (버전 민감도)

최신 `docker-ce` 패키지를 무분별하게 설치할 경우, JetPack 6의 `nvidia-container-toolkit` 버전과 호환성 충돌을 일으키거나 `systemd` cgroup 드라이버 설정과 맞지 않아 Docker 데몬 시작 자체가 실패할 수 있다.16

이러한 현상은 `sudo systemctl restart docker` 명령이 `Job for docker.service failed because the control process exited with error code` 메시지와 함께 실패하는 것으로 관찰된다.17

이에 대한 가장 안정적인 해결책은 Jetson 커뮤니티에서 검증된 특정 버전을 명시하여 설치하는 것이다.

**권장 설치 절차:**

1. **Docker 공식 GPG 키 및 저장소 추가:**

   Bash

   ```
   $ sudo apt-get update
   $ sudo apt-get install ca-certificates curl
   $ sudo install -m 0755 -d /etc/apt/keyrings
   $ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
   $ sudo chmod a+r /etc/apt/keyrings/docker.asc
   $ echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   $ sudo apt-get update
   ```

   (상기 단계는 16에서 참조한 Docker 공식 문서를 따른 것이다.)

2. **검증된 특정 버전 설치:** 포럼 보고서 17에 따르면, 특정 이전 버전(예: `5:27.5.1`)이 JetPack 6.2 환경에서 안정적으로 작동함이 확인되었다. `docker-ce`의 최신 버전이 문제를 일으킬 경우, 다음과 같이 버전을 명시하여 다운그레이드 혹은 설치를 진행하라.

   Bash

   ```
   $ sudo apt-get install docker-ce=5:27.5.1-1~ubuntu.22.04~jammy \
                          docker-ce-cli=5:27.5.1-1~ubuntu.22.04~jammy \
                          docker-compose-plugin \
                          docker-buildx-plugin \
                          docker-ce-rootless-extras=5:27.5.1-1~ubuntu.22.04~jammy
   ```

3. **업데이트 방지:** 안정화된 버전이 의도치 않게 업데이트되는 것을 방지하기 위해 `apt-mark hold`를 설정하는 것을 강력히 권장한다.17

   Bash

   ```
   $ sudo apt-mark hold docker-ce docker-ce-cli docker-ce-rootless-extras
   ```

4. **대안 (스크립트 사용):** `jetsonhacks/install-docker` 저장소는 이 과정을 자동화하는 스크립트(`install_nvidia_docker.sh`)를 제공하며, 내부적으로 검증된 버전을 설치한다.15

### NVIDIA Container Toolkit 설치 및 구성

Docker 설치 후, AGX Orin의 GPU와 하드웨어 가속기에 접근하기 위해 NVIDIA Container Toolkit을 설치하고 구성해야 한다.

1. **Toolkit 설치:** JetPack 6은 `nvidia-container-toolkit`을 설치하기 위한 `apt` 저장소를 이미 포함하고 있다. `nvidia-jetpack` 메타 패키지를 설치했다면 필요한 구성 요소(예: `nvidia-container`)가 이미 설치되어 있을 수 있다.19 만약 누락되었다면, 다음을 통해 설치한다.

   Bash

   ```
   $ sudo apt-get update
   $ sudo apt-get install -y nvidia-container-toolkit
   ```

2. **핵심 구성 단계 (Runtime 설정):** 이것이 가장 중요한 단계이다. Docker가 NVIDIA 하드웨어를 인지하고 사용할 수 있도록 `nvidia` 런타임을 등록해야 한다.21

   Bash

   ```
   $ sudo nvidia-ctk runtime configure --runtime=docker
   ```

   이 명령 21은 `/etc/docker/daemon.json` 파일을 자동으로 수정하여 "nvidia"라는 커스텀 런타임을 등록하고 이를 기본값으로 설정한다.21 이 `nvidia` 런타임은 컨테이너 시작 시 `nvidia-container-runtime-hook`을 실행하여, 호스트의 GPU 드라이버, CUDA 라이브러리, Tegra 멀티미디어 API 등 5을 컨테이너 내부로 마운트하는 핵심 메커니즘으로 작동한다.23

3. **Docker 데몬 재시작:** 설정을 적용하기 위해 Docker 서비스를 재시작한다.

   Bash

   ```
   $ sudo systemctl restart docker
   ```

### 사용자 그룹 설정 (선택 사항)

매번 `sudo`를 입력하지 않고 Docker 명령을 사용하기 위해 현재 사용자를 `docker` 그룹에 추가한다.24

Bash

```
$ sudo usermod -aG docker $USER
$ newgrp docker
```

(이후 로그아웃 후 다시 로그인하면 `newgrp` 없이 영구 적용된다.)

## III. 방법 1: (권장) `dusty-nv/jetson-containers`를 사용한 자동 빌드

수동 Dockerfile 작성은 L4T 버전 불일치 6, PCL/VTK 의존성 문제 11, `aarch64` 컴파일 오류 등 수많은 함정을 내포하고 있다. `dusty-nv/jetson-containers` 26는 이 모든 복잡한 과정을 해결하기 위해 NVIDIA Jetson 커뮤니티에서 개발 및 유지보수하는 스크립트 및 Dockerfile의 집합이다.

이는 Jetson 생태계에서 사실상의 표준(de-facto standard)으로 통용되며, ROS 2 Humble 빌드를 포함한 대부분의 작업을 자동화한다.27

### `jetson-containers` 작동 방식

이 프레임워크의 핵심 기능은 단순히 Dockerfile을 제공하는 것이 아니라, **호스트 시스템의 L4T 버전을 자동으로 감지**하여 해당 버전에 정확히 일치하는 `BASE_IMAGE` 인수를 빌드 스크립트에 동적으로 주입하는 것이다.30

예를 들어, JetPack 6 (L4T R36.x) 호스트에서 빌드를 실행하면 `l4t-base:r36.x`를 기본 이미지로 자동 선택하고, JetPack 5 (L4T R35.x) 호스트에서는 `l4t-base:r35.x`를 선택한다. 이는 IV부에서 상세히 다룰 'L4T 버전 결합' 문제를 사용자가 인지하지 못하는 사이 완벽하게 자동 해결한다.

### 빌드 명령어

`dusty-nv/jetson-containers`를 사용하는 것이 가장 빠르고 안정적으로 ROS 2 Humble, PCL, VTK가 포함된 이미지를 얻는 방법이다.

1. **저장소 복제(Clone):**

   Bash

   ```
   $ git clone https://github.com/dusty-nv/jetson-containers
   $ cd jetson-containers
   ```

2. ROS 2 Humble (Desktop) 빌드 실행:

   ros-humble-desktop은 PCL과 VTK를 포함하는 pcl_ros 및 RViz2를 포함한다.

   - **구형 스크립트 방식:**

     Bash

     ```
     $./scripts/docker_build_ros.sh --distro humble --package desktop
     ```

     (31에서 이 명령어로 `desktop` 버전 빌드가 가능함이 언급된다.)

   - **신형 `jetson-containers` 도구 방식 (권장):**

     Bash

     ```
     $ jetson-containers build ros:humble-desktop
     ```

     (26에서 이 빌드 명령어가 PyTorch, Transformers, ROS 등 다양한 패키지를 조합할 수 있음을 보여준다.)

### 잠재적 문제

이 프레임워크가 대부분의 문제를 해결하지만, 완벽하지 않을 수 있다. 일부 사용자들은 빌드 실패 32 또는 `jetson-inference`와 같은 특정 패키지와의 통합 시 데이터 파일 누락 문제를 보고하기도 한다.33 그럼에도 불구하고, 이는 수동으로 빌드할 때 마주칠 거대한 문제들에 비하면 훨씬 사소하며 해결하기 쉬운 수준이다.

## IV. 방법 2: (심층) 수동 Dockerfile 상세 구성



`jetson-containers` 프레임워크가 내부적으로 수행하는 작업을 이해하거나, 고도로 최적화되고 맞춤화된 이미지를 직접 제어하며 빌드해야 하는 전문가를 위해 수동 Dockerfile 작성법을 상세히 기술한다.



### 4.1. 기본 이미지 선택: `l4t-base`의 치명적 중요성



Dockerfile의 `FROM` 명령어는 AGX Orin 컨테이너 빌드의 성패를 가르는 첫 번째 관문이다.

잘못된 접근:

FROM ubuntu:22.04

aarch64 아키텍처용 Ubuntu 22.04 이미지를 사용하는 것은 논리적으로 보이나, 이는 반드시 실패한다. 이 이미지는 NVIDIA 드라이버 및 L4T 하드웨어 가속 라이브러리와의 연동 메커니즘을 전혀 갖추고 있지 않다. 컨테이너는 부팅될 수 있으나, GPU에 접근하려는 모든 시도는 no CUDA-capable device is detected 3 또는 CUDA driver version is insufficient 3 오류를 반환할 것이다.

올바른 접근:

FROM nvcr.io/nvidia/l4t-base:r36.x.x

반드시 NVIDIA NGC(NVIDIA GPU Cloud)에서 제공하는 공식 L4T 기본 이미지를 사용해야 한다.5

L4T 버전 결합 메커니즘:

l4t-base 컨테이너 5는 의도적으로 경량화되어 있다. 이 이미지 안에는 CUDA, cuDNN, TensorRT 라이브러리가 포함되어 있지 않다. 대신, 2단계에서 설정한 nvidia 런타임 23이 컨테이너가 시작될 때 호스트 시스템의 /usr/lib/aarch64-linux-gnu/tegra/ 및 기타 경로에 존재하는 L4T 관련 라이브러리(CUDA, cuDNN, VPI, GStreamer, Vulkan 등) 5를 컨테이너 내부로 주입(바인드 마운트)하는 것을 전제로 설계되었다.

이것이 **호스트의 L4T 버전과 컨테이너의 L4T 태그가 정확히 일치해야 하는** 이유이다.6 R36.x (JetPack 6) 호스트에서 R35.x (JetPack 5) 컨테이너를 실행하면 라이브러리 불일치로 인해 실패한다.7

테이블 1: JetPack/L4T 버전과 L4T 컨테이너 태그 매핑

다음은 사용자의 AGX Orin 호스트 환경에 맞는 정확한 기본 이미지 태그를 선택하기 위한 매핑 테이블이다.

| **호스트 JetPack 버전** | **호스트 L4T 버전** | **호스트 Ubuntu 버전** | **권장 l4t-base 태그 (aarch64)**                   |
| ----------------------- | ------------------- | ---------------------- | -------------------------------------------------- |
| **JetPack 6.x**         | **L4T R36.x**       | **22.04**              | `nvcr.io/nvidia/l4t-base:r36.3.0` 36, `r36.2.0` 5  |
| JetPack 5.x             | L4T R35.x           | 20.04                  | `nvcr.io/nvidia/l4t-base:r35.4.1` 38, `r35.1.0` 39 |
| JetPack 4.x             | L4T R32.x           | 18.04                  | `nvcr.io/nvidia/l4t-base:r32.7.1` 6, `r32.5.0` 30  |

(참고: `l4t-jetpack` 이미지는 `l4t-base`에 CUDA, cuDNN, TensorRT 런타임이 미리 설치된 버전으로40 `l4t-base`를 기반으로 빌드할 수도 있다.)



### 4.2. ROS 2 Humble 설치 계층



기본 이미지를 `l4t-base:r36.x` (Ubuntu 22.04 기반 37)로 선택한 후, ROS 2 Humble을 설치하는 표준 절차를 따른다.

Dockerfile

```
# (예시) JetPack 6.x 호스트용
FROM nvcr.io/nvidia/l4t-base:r36.2.0

# 1. 환경 변수 설정 (비대화형 설치)
ENV DEBIAN_FRONTEND=noninteractive

# 2. 로케일(Locale) 설정 (UTF-8 필수)
RUN apt-get update && \
    apt-get install -y locales && \
    locale-gen en_US.UTF-8 && \
    update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
ENV LANG=en_US.UTF-8
# [8, 53]

# 3. ROS 2 GPG 키 및 저장소 추가
RUN apt-get update && \
    apt-get install -y curl gnupg lsb-release && \
    curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | tee /etc/apt/sources.list.d/ros2.list > /dev/null
# [8, 54]

# 4. ROS 2 Humble 설치
RUN apt-get update && \
    apt-get upgrade -y
# 의 경고: ROS 2 설치 전 시스템 업그레이드(특히 systemd, udev)가 중요함.
```



### 4.3. PCL 및 VTK 설치: 의존성 문제와 치명적 오류 해결책



사용자가 요청한 PCL과 VTK는 `ros-humble-desktop-full` 메타 패키지를 통해 가장 간단하게 설치할 수 있다.

암묵적 의존성 해결:

ros-humble-desktop-full은 ros-humble-pcl-ros 41를 포함하며, 이는 다시 libpcl 라이브러리를 의존성으로 갖는다.9 또한, PCL의 시각화 모듈 및 RViz2는 VTK 라이브러리를 의존성으로 포함한다.10

Dockerfile

```
# 4.3.1. ros-humble-desktop-full 설치 (PCL, VTK 자동 포함)
RUN apt-get install -y ros-humble-desktop-full python3-colcon-common-extensions
# [25, 54]

# 4.3.2. rosdep 초기화
RUN apt-get install -y python3-rosdep && \
    rosdep init && \
    rosdep update
```

치명적인 잠재적 오류 (PCL/VTK 세그폴트):

단순히 apt-get으로 설치하는 것은 PCL 1.12.1 (또는 이전 버전)에서 PCLVisualizer를 사용할 때, VTK 9.x와의 API/ABI 비호환성으로 인해 발생하는 심각한 세그멘테이션 오류(Segfault)의 위험을 안고 있다.11

- **문제:** PCL 1.12.1 + VTK 9.x 환경에서 `PCLVisualizer::spinOnce()` 또는 관련 시각화 함수 호출 시 즉시 세그폴트 발생.11
- **원인:** VTK 라이브러리의 ABI 불안정성 12 또는 PCL의 VTK 관련 코드 버그.
- **해결책:** 이 문제는 **PCL 1.13.0** 이상 버전에서 해결되었다.11

전문가적 조치 (Actionable Warning):

ros-humble-desktop-full 설치 후, 컨테이너 내에서 apt show libpcl-dev 또는 pcl_viewer 명령을 실행하여 설치된 PCL 버전을 확인해야 한다. 만약 PCL 버전이 1.13.0 미만이라면, apt로 설치된 버전을 apt-get remove로 제거하고, Dockerfile 내에서 PCL 1.13.0 이상 버전을 소스에서 직접 빌드하는 과정을 추가해야 한다. 이는 VTK 관련 세그폴트를 피하기 위한 필수적인 조치이다.



### 4.4. 환경 설정 및 정리



이미지 용량을 줄이고 컨테이너 실행을 용이하게 하기 위해 정리 및 환경 설정 계층을 추가한다.

Dockerfile

```
# 4.4.1. APT 캐시 정리
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# [25]

# 4.4.2. ROS 환경 설정
# [54]
ENV ROS_DISTRO=humble
ENV ROS_PYTHON_VERSION=3
ENV ROS_VERSION=2

# 4.4.3. Entrypoint 설정 (컨테이너 시작 시 자동 소싱)
COPY./ros_entrypoint.sh /
RUN chmod +x /ros_entrypoint.sh
ENTRYPOINT ["/ros_entrypoint.sh"]
CMD ["bash"]
```

`ros_entrypoint.sh` 파일의 내용은 다음과 같아야 한다 25:

Bash

```
#!/bin/bash
set -e

# ROS 2 환경 설정
source /opt/ros/humble/setup.bash

# 전달된 명령 실행
exec "$@"
```



## V. 3단계: Docker 이미지 빌드 전략 (On-Device vs. Cross-Compile)



작성된 Dockerfile을 이미지로 빌드하는 전략은 개발 환경에 따라 나뉜다.



### 5.1. 온-디바이스 빌드 (On-Device Build)



가장 간단한 방법은 AGX Orin 장치 상에서 직접 `docker build` 명령을 실행하는 것이다.

Bash

```
# AGX Orin 터미널에서 실행
$ docker build -t my-ros-humble-pcl-image.
```

- **장점:** 설정이 가장 간단하며 `aarch64` 아키텍처 문제를 신경 쓸 필요가 없다. AGX Orin의 컴파일 속도는 상당히 빠르다.27
- **단점:** ROS 2, PCL, VTK 전체를 소스에서 빌드할 경우(특히 PCL 세그폴트 문제 해결 시), 상당한 시간과 리소스(RAM, 저장 공간)를 소모한다.



### 5.2. x86_64 호스트에서의 크로스 컴파일



개발 속도 향상 및 CI/CD 파이프라인 연동을 위해, 고성능 x86_64 개발 머신에서 `linux/arm64` 타겟 이미지를 빌드(크로스 컴파일)할 수 있다.2

- **필요성:** AGX Orin의 리소스를 아끼고, x86 호스트의 강력한 CPU 성능을 활용하여 빌드 시간을 단축한다.44

- **1단계: QEMU 및 binfmt 설정:** x86 호스트가 ARM64 바이너리를 에뮬레이션할 수 있도록 `qemu-user-static` 및 `binfmt-support`를 설치해야 한다.4

  Bash

  ```
  $ sudo apt-get install qemu binfmt-support qemu-user-static
  $ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
  ```

- **2단계: `docker buildx` 설정:** Docker의 다중 아키텍처 빌드 도구인 `buildx`를 생성하고 사용하도록 설정한다.

  Bash

  ```
  $ docker buildx create --name arm_builder --use
  $ docker buildx inspect --bootstrap
  ```

- **3단계: 빌드 실행:** `--platform linux/arm64` 플래그를 명시하여 이미지를 빌드한다.45 `--load`는 빌드된 이미지를 로컬 Docker 데몬으로 가져오기 위함이다.

  Bash

  ```
  $ docker buildx build --platform linux/arm64 \
    -t my-ros-humble-pcl-image:latest \
    --load.
  ```

- **`exec format error`:** 이 과정에서 `exec format error` 2가 발생한다면, 이는 QEMU 에뮬레이션 설정이(1단계) 올바르게 작동하지 않거나 `buildx`가 아닌 일반 `docker build`를 사용하여 아키텍처가 불일치했음을 의미한다.



## VI. 4단계: 컨테이너 실행 및 활용 (GPU 및 GUI 완벽 연동)



빌드된 이미지를 사용하여 AGX Orin에서 ROS 2 Humble, PCL, VTK의 모든 기능을 활용하는 것은 `docker run` 명령어에 정확한 플래그 조합을 사용하는 것에 달려있다.



### 6.1. GPU 가속 실행 (필수)



컨테이너 내부에서 CUDA, TensorRT 및 AGX Orin GPU에 접근하려면 `nvidia` 런타임을 명시해야 한다.

- **핵심 플래그:** `docker run` 명령어에 `--runtime nvidia` 22 플래그를 반드시 포함해야 한다.
- 이 플래그는 2단계에서 `daemon.json`에 설정한 `nvidia` 런타임을 사용하도록 Docker에 지시하며, L4T 라이브러리 주입(IV-4.1 참조)을 트리거한다.5
- `--gpus all` 47 플래그는 `nvidia` 런타임이 AGX Orin의 모든 가용 GPU 리소스를 컨테이너에 노출하도록 지시한다.
- **결론:** **`--runtime nvidia --gpus all`** 두 플래그를 함께 사용하는 것이 가장 확실하고 호환성이 높다.



### 6.2. GUI 애플리케이션 실행 (VTK, PCLVisualizer, RViz2)



VTK, PCL 시각화, RViz2는 모두 X11 서버를 필요로 하는 GUI 애플리케이션이다.

- **호스트 설정 (필수):** 컨테이너를 실행하기 *전*, AGX Orin의 호스트 터미널(로컬 데스크톱 또는 SSH 세션)에서 X 서버에 대한 접근 권한을 개방해야 한다.

  Bash

  ```
  # 호스트 터미널에서 실행
  $ xhost +
  ```

  (더 안전한 방법: `$ xhost +local:docker` 48)

- **Docker Run 플래그 (GUI 3종 세트):** X11 포워딩을 위해 다음 플래그가 필수적이다.14

  1. `-e DISPLAY=$DISPLAY`: 호스트의 `DISPLAY` 환경 변수를 컨테이너로 그대로 전달한다.14
  2. `-v /tmp/.X11-unix:/tmp/.X11-unix`: X11 통신을 위한 유닉스 소켓 파일을 컨테이너 내부에 마운트한다.14



### 6.3. 고급 GUI 설정 (원격 SSH 및 RViz2 가속)



원격 PC에서 `ssh -X`를 통해 AGX Orin에 접속하여 GUI를 실행할 때, 치명적인 네트워킹 함정이 존재한다.

- **`ssh -X`의 함정:** `ssh -X user@agx-orin`으로 접속 시, 호스트의 `DISPLAY` 환경 변수는 `:0`이 아닌 `localhost:10.0`과 같이 설정된다.13
- **문제 발생:** 이 `DISPLAY=localhost:10.0`이 컨테이너로 전달되면, 컨테이너는 *컨테이너 내부의* `localhost` (즉, 자기 자신)에서 X 서버를 찾으려 시도한다. 호스트의 X 서버 터널에 접근할 수 없으므로 `Error: Can't open display: localhost:10.0` 13 오류가 발생한다.
- **해결책 1 (권장):** `docker run`에 `--net=host` 플래그를 추가한다.14 이는 컨테이너가 호스트의 네트워크 네임스페이스를 공유하도록 하여, 컨테이너 내부의 `localhost:10.0`이 호스트의 X11 터널을 올바르게 가리키도록 한다. (보안상 단점이 있으나 가장 간단하다.)
- **해결책 2 (고급):** `socat`을 사용하여 호스트의 유닉스 소켓(`X10`)을 TCP 포트로 프록시한다.51

Qt/RViz2 특화 플래그:

RViz2와 같은 Qt 기반 애플리케이션은 추가 환경 변수가 필요할 수 있다.

- `-e QT_X11_NO_MITSHM=1`: X 서버와의 공유 메모리(MIT-SHM) 사용을 비활성화하여 Qt 관련 X11 오류를 방지한다.51

- `-e __NV_PRIME_RENDER_OFFLOAD=1`

- -e __GLX_VENDOR_LIBRARY_NAME=nvidia

  위 두 변수는 컨테이너 내에서 RViz2가 하드웨어 가속(GPU)을 사용하도록 강제하여 성능을 향상시키는 데 필요할 수 있다.52



### 6.4. 통합 실행 명령어 예시



지금까지의 모든 요소를 결합하여 사용 시나리오별 `docker run` 명령어 플래그를 요약한다.

테이블 2: 사용 시나리오별 docker run 명령어 플래그 요약

다음은 사용자의 목표에 따라 필요한 플래그를 조합하기 위한 "레시피"이다.

| **목표**                    | **필수 플래그**                                              | **설명**                                                     |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **CLI 전용**                | `-it --rm`                                                   | 대화형 터미널을 실행하고, 종료 시 컨테이너를 삭제한다.       |
| **GPU 가속 (CUDA)**         | `[CLI] + --runtime nvidia --gpus all`                        | NVIDIA 런타임을 사용해 모든 GPU를 컨테이너에 노출한다.22     |
| **GUI (로컬 데스크톱)**     | `[GPU] + -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix` | 호스트의 X11 소켓과 `DISPLAY` 변수를 공유한다.14 (호스트에서 `xhost +` 실행 49). |
| **GUI (원격 SSH `ssh -X`)** | `[GPU] + --net=host -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix` | `--net=host`를 추가하여 SSH X11 포워딩(`localhost:10.0`) 문제를 해결한다.13 |
| **GUI (RViz/PCL)**          | `[GUI (원격 또는 로컬)] + -e QT_X11_NO_MITSHM=1 -e __NV_PRIME_RENDER_OFFLOAD=1 -e __GLX_VENDOR_LIBRARY_NAME=nvidia` | Qt/RViz/VTK의 안정성 및 GPU 가속을 위한 환경 변수를 추가한다.51 |

**최종 실행 명령어 예시 (원격 PC에서 `ssh -X`로 AGX Orin에 접속하여 RViz2 실행):**

1. **호스트 터미널 (AGX Orin):**

   Bash

   ```
   $ xhost +
   ```

2. **호스트 터미널 (AGX Orin, `ssh -X` 세션 내부):**

   Bash

   ```
   $ docker run -it --rm \
     --runtime nvidia \
     --gpus all \
     --net=host \
     -e DISPLAY=$DISPLAY \
     -e QT_X11_NO_MITSHM=1 \
     -e __NV_PRIME_RENDER_OFFLOAD=1 \
     -e __GLX_VENDOR_LIBRARY_NAME=nvidia \
     -v /tmp/.X11-unix:/tmp/.X11-unix \
     -v /home/nvidia/ros2_ws:/root/ros2_ws \
     my-ros-humble-pcl-image:latest \
     bash
   
   # 컨테이너 내부
   root@agx-orin:/# source /opt/ros/humble/setup.bash
   root@agx-orin:/# rviz2
   ```



## VII. 문제 해결 (Troubleshooting)



오류: `docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock...` 17

- **원인:** Docker 데몬이 실행 중이 아니거나, `docker-ce` 버전이 JetPack 6와 호환되지 않는다.
- **해결:** `systemctl status docker`로 상태를 확인한다. 데몬이 죽어있다면, II부에서 설명한 대로 `docker-ce` 버전을 검증된 버전(예: 5:27.5.1)으로 다운그레이드한다.17

오류: `exec /...: exec format error` 2

- **원인:** 아키텍처 불일치. x86_64 (`amd64`) 이미지를 `aarch64` (AGX Orin)에서 실행하려 했다.
- **해결:** Dockerfile의 `FROM` 명령어가 `linux/arm64` 또는 `aarch64` 아키텍처(예: `l4t-base`)인지 확인한다. x86 호스트에서 빌드 시 `docker buildx --platform linux/arm64`를 사용했는지 확인한다.45

오류: `cudaErrorInsufficientDriver: CUDA driver version is insufficient...` 3

- **원인:** L4T 버전 결합 문제. 컨테이너의 CUDA 툴킷 버전이 호스트의 L4T 드라이버 버전과 호환되지 않는다.
- **해결:** Dockerfile의 `FROM` 이미지가 호스트의 JetPack/L4T 버전과 정확히 일치하는지 **테이블 1**을 참조하여 확인한다.6

오류: `Error: Can't open display: :0` 또는 `localhost:10.0` 13

- **원인:** X11 포워딩 실패.
- **해결:** 다음을 순서대로 확인한다.
  1. 호스트에서 `xhost +` 49를 실행했는가?
  2. `docker run`에 `-e DISPLAY=$DISPLAY`와 `-v /tmp/.X11-unix:/tmp/.X11-unix` 플래그 50가 포함되었는가?
  3. `ssh -X`를 사용 중인가? 그렇다면 `DISPLAY` 변수가 `localhost:10.0`인지 확인하고, `--net=host` 14 플래그를 추가했는가?

오류: PCL/VTK 시각화 노드 실행 시 세그멘테이션 오류 (Segfault) 11

- **원인:** PCL 1.13 미만 버전과 VTK 9.x 간의 알려진 버그.11
- **해결:** IV-4.3부의 '전문가적 조치'를 따른다. 컨테이너 내 PCL 버전을 확인한다. 1.13.0 미만일 경우, `apt` 버전을 제거하고 Dockerfile 내에서 PCL 1.13.0 이상 버전을 소스에서 직접 빌드해야 한다.



## VIII. 결론 및 전문가 제언



NVIDIA AGX Orin (JetPack 6) 환경에서 ROS 2 Humble, PCL, VTK 스택을 컨테이너화하는 작업은, 호스트의 L4T 드라이버와 컨테이너가 긴밀하게 결합하는 NVIDIA의 하이브리드 런타임 모델 5에 대한 정확한 이해를 요구하는 고도의 기술적 과제이다.

이 과정에서 개발자가 직면하는 가장 큰 함정은 다음 세 가지로 요약된다.

1. **JetPack 6의 수동 Docker 설치:** JP6는 Docker를 포함하지 않으며 15, 최신 `docker-ce` 버전은 호환성 문제를 일으킬 수 있으므로 특정 버전의 설치가 강제된다.17
2. **L4T 버전 결합:** `nvcr.io/nvidia/l4t-base` 이미지는 호스트의 드라이버를 주입받는 방식 5으로 작동하므로, 호스트 L4T 태그와 컨테이너 태그의 엄격한 버전 일치(예: R36.x 호스트 -> R36.x 태그)가 필수적이다.6
3. **PCL/VTK 시각화 버그:** `apt`를 통한 표준 설치는 PCL 1.13.0 미만 버전을 설치할 수 있으며, 이는 VTK 시각화 시 세그멘테이션 오류를 유발할 수 있다.11

최종 권고:

이러한 복잡성을 90% 이상 줄여주는 가장 효율적이고 검증된 전문가적 접근 방식은 dusty-nv/jetson-containers 프레임워크 26를 사용하는 것이다. 이 프레임워크는 L4T 버전 결합 문제를 자동으로 해결해준다.30

수동 Dockerfile 작성은 이 프레임워크가 해결하지 못하는 특수한 의존성이나 고도의 최적화가 필요한 경우에만 시도해야 한다.

어떤 방법으로 이미지를 빌드하든, GPU 가속(`--runtime nvidia`) 22과 GUI 포워딩(`-e DISPLAY`, `-v /tmp/.X11-unix`, `--net=host`) 14을 위한 정확한 `docker run` 플래그 조합(본문 **테이블 2** 참조)을 사용하는 것이 AGX Orin에서 ROS 2, PCL, VTK의 모든 성능을 활용하는 마지막 열쇠가 될 것이다.

#### **Works cited**

1. JetPack SDK 6.0 - NVIDIA Developer, accessed November 14, 2025, https://developer.nvidia.com/embedded/jetpack-sdk-60
2. Building ARM64-based Docker containers for NVIDIA Jetson devices on an x86-based host., accessed November 14, 2025, https://medium.com/@Smartcow_ai/building-arm64-based-docker-containers-for-nvidia-jetson-devices-on-an-x86-based-host-d72cfa535786
3. Docker image for Jetson AGX Orin with CUDA environment - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/docker-image-for-jetson-agx-orin-with-cuda-environment/293561
4. Cross Compilation of docker images between amd64 and arm64 architecture - DeepStream SDK - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/cross-compilation-of-docker-images-between-amd64-and-arm64-architecture/332095
5. NVIDIA L4T Base - NGC Catalog, accessed November 14, 2025, https://catalog.ngc.nvidia.com/orgs/nvidia/containers/l4t-base
6. Run image based off Ubuntu 22.04 on R32.7.2 / Where is l4t:r32.7.2 - Jetson Xavier NX, accessed November 14, 2025, https://forums.developer.nvidia.com/t/run-image-based-off-ubuntu-22-04-on-r32-7-2-where-is-l4t-r32-7-2/242129
7. Host [jetpack 6.x ubuntu 22.04] docker image [ubuntu18.04] opengl and rviz not work in docker - Jetson AGX Orin - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/host-jetpack-6-x-ubuntu-22-04-docker-image-ubuntu18-04-opengl-and-rviz-not-work-in-docker/316231
8. Ubuntu (deb packages) — ROS 2 Documentation: Humble documentation, accessed November 14, 2025, https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html
9. AUR (en) - pcl - Arch Linux, accessed November 14, 2025, https://aur.archlinux.org/packages/pcl?O=100
10. Pcl_ros with PCL 1.8 (ROS Lunar and newer) means linking to 139 VTK dynamic libraries, accessed November 14, 2025, https://discourse.openrobotics.org/t/pcl-ros-with-pcl-1-8-ros-lunar-and-newer-means-linking-to-139-vtk-dynamic-libraries/4433
11. Segmentation fault in PCL viewer - c++ - Stack Overflow, accessed November 14, 2025, https://stackoverflow.com/questions/77751213/segmentation-fault-in-pcl-viewer
12. Linking Error with ROS PCL and Drake: Different VTK versions - Stack Overflow, accessed November 14, 2025, https://stackoverflow.com/questions/73981130/linking-error-with-ros-pcl-and-drake-different-vtk-versions
13. X11 Forwarding with with Docker containers - Jetson Orin Nano - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/x11-forwarding-with-with-docker-containers/328241
14. ROS2: rviz in docker container - Robotics Stack Exchange, accessed November 14, 2025, https://robotics.stackexchange.com/questions/88607/ros2-rviz-in-docker-container
15. Docker Setup On Jetson Orin - Includes JetPack 6 Docker fix - YouTube, accessed November 14, 2025, https://www.youtube.com/watch?v=d2I_wjJTekw
16. Error with “Nvidia Container Runtime with Docker Integration” on AGX Orin with JP6.2, accessed November 14, 2025, https://forums.developer.nvidia.com/t/error-with-nvidia-container-runtime-with-docker-integration-on-agx-orin-with-jp6-2/324558
17. NVDIA docker container error in jetson agx orin dev kit 64gb -- jetpack 6.2, accessed November 14, 2025, https://forums.developer.nvidia.com/t/nvdia-docker-container-error-in-jetson-agx-orin-dev-kit-64gb-jetpack-6-2/324725
18. Docker Setup on JetPack 6 - Jetson Orin - JetsonHacks, accessed November 14, 2025, https://jetsonhacks.com/2025/02/24/docker-setup-on-jetpack-6-jetson-orin/
19. Getting Started with Jetson AGX Orin Developer Kit, accessed November 14, 2025, https://developer.nvidia.com/embedded/learn/get-started-jetson-agx-orin-devkit
20. How to Install and Configure JetPack SDK - NVIDIA Docs Hub, accessed November 14, 2025, https://docs.nvidia.com/jetson/archives/jetpack-archived/jetpack-60/install-setup/index.html
21. Installing the NVIDIA Container Toolkit, accessed November 14, 2025, https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
22. GPU not accessible in a custom Docker container - Jetson AGX Orin, accessed November 14, 2025, https://forums.developer.nvidia.com/t/gpu-not-accessible-in-a-custom-docker-container/309099
23. NVIDIA Container Runtime on Jetson (Beta) — Cloud Native Products documentation, accessed November 14, 2025, https://nvidia.github.io/container-wiki/toolkit/jetson.html
24. Setup ROS 2 with VSCode and Docker [community-contributed], accessed November 14, 2025, https://docs.ros.org/en/humble/How-To-Guides/Setup-ROS-2-with-VSCode-and-Docker-Container.html
25. The Complete Beginner's Guide to Docker for ROS 2 Deployment (2025) - Robotair, accessed November 14, 2025, https://blog.robotair.io/the-complete-beginners-guide-to-using-docker-for-ros-2-deployment-2025-edition-0f259ca8b378
26. dusty-nv/jetson-containers: Machine Learning Containers for NVIDIA Jetson and JetPack-L4T - GitHub, accessed November 14, 2025, https://github.com/dusty-nv/jetson-containers
27. Advice wanted for ROS 2 development of AGX Orin, accessed November 14, 2025, https://forums.developer.nvidia.com/t/advice-wanted-for-ros-2-development-of-agx-orin/251354
28. Install ros 2 humble on jetson orin : r/ROS - Reddit, accessed November 14, 2025, https://www.reddit.com/r/ROS/comments/14ipjm3/install_ros_2_humble_on_jetson_orin/
29. Proper dev toolchain for ROS2 Humble in dusty's pre-built docker image on a remote Xavier NX - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/proper-dev-toolchain-for-ros2-humble-in-dustys-pre-built-docker-image-on-a-remote-xavier-nx/229400
30. Dockerfile for ros humble #244 - dusty-nv/jetson-containers - GitHub, accessed November 14, 2025, https://github.com/dusty-nv/jetson-containers/issues/244
31. Trying to install ros2 humble desktop version in Jetson Xavier via Docker #195 - GitHub, accessed November 14, 2025, https://github.com/dusty-nv/jetson-containers/issues/195
32. Dockerfile for ROS2 Humble running on Jetson Orin Nano with Ubuntu 22.04?, accessed November 14, 2025, https://forums.developer.nvidia.com/t/dockerfile-for-ros2-humble-running-on-jetson-orin-nano-with-ubuntu-22-04/349594
33. Dockerfile for Ros-Humble-desktop with PyTorch · Issue #247 · dusty-nv/jetson-containers, accessed November 14, 2025, https://github.com/dusty-nv/jetson-containers/issues/247
34. JetPack 6.0: Cannot run a base CUDA Docker container - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/jetpack-6-0-cannot-run-a-base-cuda-docker-container/322397
35. How to install nvidia-l4t-core in docker? - Jetson AGX Orin, accessed November 14, 2025, https://forums.developer.nvidia.com/t/how-to-install-nvidia-l4t-core-in-docker/329942
36. Image Layer Details - dustynv/ros:humble-ros-base-l4t-r36.3.0 | Docker Hub, accessed November 14, 2025, https://hub.docker.com/layers/dustynv/ros/humble-ros-base-l4t-r36.3.0/images/sha256-15b670796051ee3be829a559862432c5bafbdcbc93f80555dc6741a90a0f9116
37. Dockerfiles and workflows to build various docker containers for jetson nano devices - GitHub, accessed November 14, 2025, https://github.com/KalanaRatnayake/jetson-docker
38. Why does jetpack >6 use ubuntu:22.04 as base image instead of nvcr.io/nvidia/l4t-jetpack:r36.X.X · Issue #883 · dusty-nv/jetson-containers - GitHub, accessed November 14, 2025, https://github.com/dusty-nv/jetson-containers/issues/883
39. How to build and run ROS2 Humble on Jetson AGX Xavier · Issue #226 - GitHub, accessed November 14, 2025, https://github.com/dusty-nv/jetson-containers/issues/226
40. NVIDIA docker container l4t-base:r36.3.0 not released - Jetson Nano, accessed November 14, 2025, https://forums.developer.nvidia.com/t/nvidia-docker-container-l4t-base-r36-3-0-not-released/303852
41. pcl_ros: Humble 2.7.3 documentation - ROS documentation, accessed November 14, 2025, https://docs.ros.org/en/humble/p/pcl_ros/
42. pcl libraries and vtk cannot be compiled together with cmakelists · Issue #6212 - GitHub, accessed November 14, 2025, https://github.com/PointCloudLibrary/pcl/issues/6212
43. Complete Dockerfile ros2-humble-desktop - ROS Answers archive, accessed November 14, 2025, https://answers.ros.org/question/416582/
44. Running and Building ARM Docker Containers on x86 - Stereolabs, accessed November 14, 2025, https://www.stereolabs.com/docs/docker/building-arm-container-on-x86
45. Multi-platform builds - Docker Docs, accessed November 14, 2025, https://docs.docker.com/build/building/multi-platform/
46. Cross-compile Deepstream Docker Image for Jetson on x86_64 - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/cross-compile-deepstream-docker-image-for-jetson-on-x86-64/313057
47. Running Docker Containers Directly on NVIDIA DRIVE AGX Orin | NVIDIA Technical Blog, accessed November 14, 2025, https://developer.nvidia.com/blog/running-docker-containers-directly-on-nvidia-drive-agx-orin/
48. Trying out ROS2 Humble Hawksbill using Docker | by RoboFoundry | Medium, accessed November 14, 2025, https://robofoundry.medium.com/trying-out-ros2-humble-hawksbill-using-docker-4490bc88c926
49. X11 forwarding using docker desktop for linux, accessed November 14, 2025, https://forums.docker.com/t/x11-forwarding-using-docker-desktop-for-linux/137266
50. How to show ros rviz in docker · Issue #239 · microsoft/wslg - GitHub, accessed November 14, 2025, https://github.com/microsoft/wslg/issues/239
51. Run Rviz from remote docker using X11 - Stack Overflow, accessed November 14, 2025, https://stackoverflow.com/questions/61283781/run-rviz-from-remote-docker-using-x11
52. How to use Rviz2 with GPU in jetson docker container? - NVIDIA Developer Forums, accessed November 14, 2025, https://forums.developer.nvidia.com/t/how-to-use-rviz2-with-gpu-in-jetson-docker-container/343087