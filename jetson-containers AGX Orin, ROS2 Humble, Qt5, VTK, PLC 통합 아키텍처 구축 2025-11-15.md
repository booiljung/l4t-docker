# Jetson AGX Orin (Ubuntu 22.04) 환경의 jetson-containers를 활용한 ROS2 Humble, Qt5, VTK, PLC 통합 아키텍처 구축 가이드





## 섹션 1: 서론: 통합 아키텍처 설계





### 1.1. 보고서 목적 및 범위



본 보고서는 NVIDIA Jetson AGX Orin 1 플랫폼(Ubuntu 22.04/JetPack 6 3)에서 `dusty-nv/jetson-containers` 프레임워크 5를 기반으로, 단일 Docker 컨테이너 내에 복잡한 소프트웨어 스택을 통합하는 전문가 수준의 기술 절차를 제공한다.

이 스택은 로봇 공학(ROS2 Humble), 3D 시각화(VTK, Rviz), GUI(Qt5), 그리고 산업용 제어(PLC) 통신 라이브러리를 포함한다. 본 작업의 최종 목표는 이 모든 구성요소가 Jetson 하드웨어의 GPU 가속(CUDA/OpenGL 6)을 통해 원활하게 연동되는 고성능 개발 및 배포 환경을 구축하는 것이다.



### 1.2. 핵심 과제: 의존성 복잡성과 하드웨어 가속



Jetson 플랫폼에서의 소프트웨어 개발은 수많은 라이브러리와 얽힌 의존성으로 인해 종종 "늪에서 빌드하는 것"에 비유된다.8 `jetson-containers` 프레임워크는 머신 러닝(ML) 및 인공 지능(AI) 스택(예: PyTorch, Transformers)과 ROS 5에 대해 이 문제를 효과적으로 해결해주는 핵심 도구이다.8

하지만, 사용자의 요구사항 중 VTK(Visualization Toolkit)와 PLC(Programmable Logic Controller) 통신 라이브러리는 `jetson-containers`의 표준 패키지 목록에 포함되어 있지 않다.9 이는 단순한 `jetson-containers build...` 명령어 5만으로는 사용자의 요구사항을 충족할 수 없음을 의미한다.



### 1.3. 솔루션: 하이브리드 빌드 전략



본 보고서는 `jetson-containers`의 안정성과 사용자 정의의 유연성을 결합한 "하이브리드 빌드 전략"을 채택한다. 이 전략은 다음과 같은 단계로 구성된다.

1. **베이스 구축:** `jetson-containers`의 `build.sh` 스크립트를 사용하여 JetPack 6에 완벽하게 최적화된 `ros:humble-desktop` 베이스 이미지를 빌드한다.9 이 이미지는 L4T(Linux for Tegra) 환경에 맞게 소스에서 컴파일된 ROS2와 필수적인 Qt5 의존성을 이미 포함한다.13
2. **확장:** 이 베이스 이미지를 `FROM` 명령어로 하는 새로운 사용자 정의 `Dockerfile`을 작성한다.12
3. **통합:** 이 사용자 정의 `Dockerfile` 내에서 `apt` 또는 소스 코드 컴파일을 통해 누락된 VTK 및 PLC 라이브러리를 안전하게 추가한다.16

이 하이브리드 접근 방식은 `dusty-nv`가 이미 해결한 L4T, CUDA, ROS, OpenCV 간의 복잡한 의존성 10을 그대로 활용하면서, 사용자가 원하는 VTK 및 PLC 라이브러리를 안정적으로 추가할 수 있는 가장 강력하고 효율적인 방법이다. 만약 `ubuntu:22.04` 베이스 이미지부터 시작한다면, 이 모든 복잡한 통합 작업을 수동으로 재현해야 하며, 이는 극도로 어렵고 실패 확률이 높다.18



## 섹션 2: 1단계: 호스트 시스템 준비 (Jetson AGX Orin, JetPack 6)





### 2.1. 호스트 환경 검증



통합 작업을 시작하기 전에, 호스트 시스템이 다음 요구사항을 충족하는지 반드시 확인해야 한다.

- **하드웨어:** NVIDIA Jetson AGX Orin 개발자 키트 또는 상용 모듈.1
- **운영체제:** JetPack 6 (L4T R36.x).4 이는 사용자가 요구한 Ubuntu 22.04 호스트 환경을 제공한다.3 `jetson-containers` 리포지토리의 최신 버전은 JetPack 6를 명시적으로 지원하고 테스트한다.9



### 2.2. Docker 및 NVIDIA Container Runtime 설치



JetPack 6 릴리스부터 Docker는 더 이상 운영체제에 기본 설치되어 있지 않다.20 이는 JetPack 5 22 환경에서 마이그레이션하는 사용자들이 직면하는 주요 변경 사항이다. NVIDIA는 Jetson Linux BSP와 AI 스택을 분리하여 4 컨테이너 런타임을 선택적으로 설치하도록 정책을 변경했다.

따라서, `jetson-containers` 리포지토리를 클론하기 전에 다음 명령어를 통해 Docker와 NVIDIA 런타임을 수동으로 설치 및 구성해야 한다.21

1. NVIDIA 컨테이너 관련 라이브러리 설치:

   Bash

   ```
   sudo apt update
   sudo apt install -y nvidia-container
   ```

2. Docker 공식 설치 스크립트를 사용하여 Docker 엔진 설치:

   Bash

   ```
   curl https://get.docker.com | sh && sudo systemctl --now enable docker
   ```

3. Docker가 NVIDIA 런타임을 사용하도록 등록:

   Bash

   ```
   sudo nvidia-ctk runtime configure --runtime=docker
   ```

4. Docker 서비스 재시작 및 사용자 권한 설정 (재로그인 필요):

   Bash

   ```
   sudo systemctl restart docker
   sudo usermod -aG docker $USER
   newgrp docker
   ```

   21



### 2.3. NVIDIA 런타임을 기본값으로 설정



컨테이너 *실행* 시뿐만 아니라 `docker build` *빌드* 과정 중에도 NVIDIA GPU 리소스(CUDA, cuDNN 라이브러리)에 접근할 수 있도록 24 `nvidia` 런타임을 Docker 데몬의 기본값으로 설정해야 한다. 이는 `l4t-base` 컨테이너가 호스트의 CUDA/cuDNN 헤더와 라이브러리를 마운트하여 25 의존성을 해결하는 핵심 메커니즘이다.27

`/etc/docker/daemon.json` 파일을 생성하거나 수정하여 다음 내용을 적용하라.21

JSON

```
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "args":
        }
    }
}
```

파일 수정 후, 변경 사항을 적용하기 위해 Docker 데몬을 다시 시작해야 한다.

Bash

```
sudo systemctl restart docker
```



### 2.4. jetson-containers 프레임워크 설치



이제 `dusty-nv`의 `jetson-containers` 리포지토리를 클론하고 관련 파이썬 도구들을 설치한다.5

Bash

```
git clone --depth=1 https://github.com/dusty-nv/jetson-containers
cd jetson-containers
pip3 install -r requirements.txt
bash jetson-containers/install.sh
```

`install.sh` 스크립트는 `jetson-containers`, `autotag` 등 L4T 버전에 맞는 컨테이너를 자동으로 찾고 빌드/실행하는 헬퍼 스크립트들을 시스템 `PATH`에 추가한다.9



## 섹션 3: 2단계: 코어 ROS2 Humble + Qt5 베이스 구축





### 3.1. jetson-containers 빌드 시스템 활용



`jetson-containers` 프레임워크는 `ros:humble-desktop` 패키지를 공식적으로 제공하며 5, 이는 JetPack 6(L4T R36.x) 및 Ubuntu 22.04와 완벽하게 호환된다.28



### 3.2. Qt5 의존성 해결



사용자의 요구사항 중 "Qt5"는 별도의 설치 단계가 필요하지 않다. `ros:humble-desktop` 패키지는 ROS2의 핵심 시각화 도구인 `rviz2`를 포함하며 30, `rviz2`는 GUI 툴킷으로 Qt5에 대한 강력한 의존성을 갖는다.32

따라서 `jetson-containers`를 통해 `ros:humble-desktop`을 빌드하는 과정에서 `libqt5-core`를 포함한 모든 필수 Qt5 라이브러리가 `apt` 의존성으로 자동 설치 및 구성된다. 사용자가 Qt5를 수동으로 설치하려 시도할 경우, 오히려 시스템 라이브러리와의 버전 충돌 33을 야기할 수 있으므로 절대 시도해서는 안 된다. Qt5 요구사항은 `ros:humble-desktop`을 빌드함으로써 자동으로 충족된다.



### 3.3. 베이스 이미지 빌드 실행



`jetson-containers` 디렉토리에서 다음 명령을 실행하여, 본 보고서의 하이브리드 전략에 사용할 로컬 베이스 이미지를 생성한다.

Bash

```
jetson-containers build --name=jetson-ros-vtk-plc-base ros:humble-desktop
```

이 명령어 9는 내부적으로 `autotag` 5를 호출하여 현재 호스트의 L4T 버전(예: `r36.4.3`)에 맞는 `l4t-jetpack` 베이스 이미지를 선택한다. 그 후, ROS2 Humble을 소스에서 컴파일하고 13 필요한 모든 의존성을 설치하여 `jetson-ros-vtk-plc-base:r36.4.3` (또는 호스트 L4T 버전에 맞는 태그)라는 이름의 로컬 Docker 이미지를 생성한다.



## 섹션 4: 3단계: VTK 및 PLC 통합을 위한 사용자 정의 Dockerfile 작성





### 4.1. 사용자 정의 Dockerfile의 필요성



`jetson-containers` 프레임워크는 `packages` 디렉토리에 사전 정의되지 않은 임의의 `apt` 패키지 14나 소스 코드를 `build.sh` 명령줄 9을 통해 직접 추가하는 표준화된 방법을 제공하지 않는다.

`jetson-containers`의 진정한 강력함은 확장성에 있으며, 프레임워크는 사용자가 `jetson-containers`가 빌드한 이미지를 `FROM`으로 사용하여 12 자신만의 `Dockerfile`을 만들어 시스템을 확장할 것을 권장한다.14

본 섹션에서는 2단계에서 생성한 `jetson-ros-vtk-plc-base` 이미지를 기반으로 하여 누락된 VTK와 PLC 라이브러리를 추가하는 전체 `Dockerfile`을 작성한다.



### 4.2. 주석이 포함된 통합 Dockerfile



다음 `Dockerfile`은 `jetson-ros-vtk-plc-base` (2단계에서 빌드됨)를 기반으로 VTK 9와 Modbus TCP/RTU 라이브러리(`libmodbus`)를 추가한다. EtherNet/IP (`libplctag`)를 위한 설치 스크립트는 주석 처리된 형태로 선택적 옵션으로 제공한다.

Dockerfile

```
# 1단계: 2단계에서 'jetson-containers build'로 생성한 베이스 이미지를 사용한다.
# 이 태그(r36.4.3)는 호스트의 JetPack/L4T 버전에 정확히 일치해야 한다.
# 'docker images | grep jetson-ros-vtk-plc-base' 명령으로 정확한 태그를 확인하라.
ARG BASE_IMAGE=jetson-ros-vtk-plc-base:r36.4.3
FROM ${BASE_IMAGE}

# 2단계: apt가 대화형 프롬프트를 표시하지 않도록 환경 변수를 설정한다.
ENV DEBIAN_FRONTEND=noninteractive

# 3단계: apt 패키지 목록을 업데이트하고 VTK 및 PLC (libmodbus) 라이브러리를 설치한다.
# 'apt-utils'는 'debconf' 관련 경고[64]를 제거하기 위해 포함한다.
# 'libvtk9-dev'와 'python3-vtk9'는 Ubuntu 22.04의 표준 패키지이다.[16, 41]
# 이는 시스템의 Python 3.10 및 ROS의 Qt5와 완벽하게 호환된다.[32, 37]
# 'libmodbus-dev'는 Modbus TCP/RTU 통신을 위한 표준 라이브러리이다.
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-utils \
    libvtk9-dev \
    python3-vtk9 \
    libmodbus-dev \
# 4단계: apt 캐시를 정리하여 최종 이미지 크기를 최적화한다.
&& apt-get clean && rm -rf /var/lib/apt/lists/*

# 5단계: (선택 사항 - EtherNet/IP 'libplctag'를 소스에서 빌드)
# Modbus 대신 또는 Modbus와 함께 EtherNet/IP (Allen-Bradley) 지원이 필요한 경우,
# 위 3단계의 'libmodbus-dev'를 제거하거나, 이 블록의 주석을 해제하여 함께 설치한다.
# 'libplctag'는 apt 리포지토리에 없으므로 소스에서 빌드해야 한다.
#
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     cmake git build-essential \
# && cd /tmp \
# && git clone --depth=1 https://github.com/libplctag/libplctag.git \
# && mkdir -p libplctag/build && cd libplctag/build \
# && cmake.. -DBUILD_SHARED_LIBS=ON \
# && make -j$(nproc) install \
# && ldconfig \
# && cd / && rm -rf /tmp/libplctag \
# && apt-get purge -y cmake git build-essential && apt-get autoremove -y \
# && apt-get clean && rm -rf /var/lib/apt/lists/*

# 6단계: ROS2 환경을 기본으로 설정한다.
# 베이스 이미지는 이미 /ros_entrypoint.sh를 ENTRYPOINT로 사용하고 있다.[65]
# 대화형 bash 세션의 편의를 위해.bashrc에도 ROS 환경 설정을 추가한다.
RUN echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
```



## 섹션 5: 4단계: 시각화 스택 심층 통합 (VTK 및 Qt)





### 5.1. Qt5 통합 (ROS 의존성)



앞서 3.2절에서 분석했듯이, Qt5 통합은 사용자가 명시적으로 수행할 작업이 아니다. `ros:humble-desktop` 30을 설치하면 `rviz2` 32가 설치되고, `rviz2`는 `apt` 의존성 관리를 통해 호환되는 `libqt5-*` 패키지들을 자동으로 가져온다. 이 아키텍처는 안정성을 보장한다.



### 5.2. VTK 통합 (Apt-based 전략)



`Dockerfile`에서 VTK를 통합하는 방법은 시스템 안정성에 지대한 영향을 미친다. `pip install vtk` 35 명령을 사용하는 것은 권장되지 않는다. `pip`로 설치된 VTK 휠(wheel)은 Ubuntu 22.04의 기본 Python 3.10 36과 호환성 문제를 일으킬 수 있으며 37, 더 심각하게는 ROS/Rviz가 의존하는 시스템 `libOpenGL` 및 `libQt5` 라이브러리와 `pip` 패키지에 포함된 라이브러리 간의 충돌 33을 야기할 수 있다.

가장 안정적이고 호환성이 보장되는 "시스템 통합" 방식은 Ubuntu 22.04 (Jammy) 배포판이 공식적으로 제공하는 `apt` 패키지를 사용하는 것이다. `Dockerfile`에 명시된 `libvtk9-dev` (C++ 개발 파일) 41와 `python3-vtk9` (Python 바인딩) 16는 Ubuntu 22.04의 Python 3.10 및 시스템 Qt5 라이브러리와 상호 호환되도록 빌드 및 테스트되었다. 이 방식은 ROS2 스택과의 완벽한 호환성을 보장한다.



## 섹션 6: 5단계: 산업용 통신 스택 통합 (PLC 라이브러리)





### 6.1. 프로토콜 분석 및 라이브러리 선정



사용자의 "PLC" 요구사항은 프로토콜을 명시하지 않았다. 로보틱스 및 산업 자동화 환경에서 가장 보편적인 두 가지 오픈소스 PLC 통신 프로토콜은 Modbus와 EtherNet/IP이다.

- **Modbus (TCP/RTU):** Siemens, Schneider Electric 등 광범위한 디바이스에서 사용되는 경량 프로토콜이다. `libmodbus` 42가 C/C++ 표준 라이브러리이다.
- **EtherNet/IP (CIP):** 주로 Rockwell Automation (Allen-Bradley) PLC에서 사용되는 CIP(Common Industrial Protocol) 기반 프로토콜이다. `libplctag` 44가 널리 사용되는 오픈소스 C 라이브러리이다.



### 6.2. 통합 전략 비교



이 두 라이브러리는 Ubuntu 22.04에서의 배포 용이성이 다르다. `libmodbus`는 `apt`를 통해 즉시 설치할 수 있지만, `libplctag`는 소스 코드 빌드가 필요하다. 이는 `apt-cache search` 46를 통해 `libplctag` 관련 패키지가 공식 리포지토리에 존재하지 않음을 확인함으로써 알 수 있다.49

다음 표는 두 라이브러리의 통합 전략을 요약한다.

**Table 1: PLC 통신 라이브러리 선택 및 통합 전략**

| **프로토콜**      | **대상 PLC (예)**        | **C/C++ 라이브러리** | **Ubuntu 22.04 apt 지원**   | **Dockerfile 통합 방법**                      |
| ----------------- | ------------------------ | -------------------- | --------------------------- | --------------------------------------------- |
| Modbus TCP/RTU    | Siemens, Schneider       | `libmodbus` 42       | **예** (`libmodbus-dev`) 17 | `RUN apt-get install -y libmodbus-dev`        |
| EtherNet/IP (CIP) | Rockwell (Allen-Bradley) | `libplctag` 44       | **아니요** 46               | 소스 빌드 (`git`, `cmake`, `make install`) 51 |



### 6.3. Dockerfile 구현



섹션 4.2의 `Dockerfile`은 이 분석을 기반으로, 가장 접근성이 좋은 `libmodbus-dev`를 기본 설치하도록 구현되었다.17 `libplctag`가 필요한 사용자를 위해 주석 처리된 소스 빌드 51 스크립트를 선택적 옵션으로 포함했다.



## 섹션 7: 6단계: 최종 이미지 빌드 및 GUI 가속 실행





### 7.1. 사용자 정의 이미지 빌드



`jetson-containers` 스크립트가 아닌 15 표준 `docker build` 명령을 사용하여 섹션 4.2에서 작성한 `Dockerfile`로 최종 이미지를 빌드한다.

1. `Dockerfile`을 저장할 디렉토리를 생성한다.

   Bash

   ```
   mkdir -p ~/my_robot_container
   cd ~/my_robot_container
   ```

2. 선호하는 편집기(예: `nano`)를 사용하여 `Dockerfile`이라는 이름의 파일을 생성하고 섹션 4.2의 코드를 붙여넣는다. `ARG BASE_IMAGE`의 태그가 2단계에서 빌드한 이미지의 태그(예: `:r36.4.3`)와 일치하는지 반드시 확인하라.

3. `docker build` 명령을 실행하여 이미지를 빌드한다.

   Bash

   ```
   docker build -t ros-humble-vtk-plc:r36.
   ```



### 7.2. GUI 가속 실행 (권장 방식: `jetson-containers run`)



컨테이너 내부에서 Rviz2, Qt, VTK와 같은 GUI 애플리케이션을 실행하려면 `--runtime nvidia`, `DISPLAY` 환경 변수, `/tmp/.X11-unix` 소켓 마운트, `xhost` 권한 설정 등 복잡한 `docker run` 인수 52가 필요하다. 이 인수를 수동으로 관리하는 것은 번거롭고 오류를 유발하기 쉽다.56

`jetson-containers run` 스크립트 (또는 `jetson-containers` 별칭) 5는 이 모든 복잡한 인수를 자동으로 처리하도록 설계되었다.9 이 스크립트와 `autotag` 유틸리티는 레지스트리의 이미지뿐만 아니라 로컬에 빌드된 사용자 정의 이미지도 인식하고 실행할 수 있다.9 이는 사용자가 빌드한 커스텀 이미지를 실행하는 가장 전문적이고 견고한 방법이다.

**실행 절차:**

1. (호스트 시스템 재부팅 시 1회) 호스트 터미널에서 X 서버 접근 권한을 부여한다.

   Bash

   ```
   sudo xhost +si:localuser:root
   ```

   53

2. `autotag`를 사용하여 방금 빌드한 커스텀 이미지를 대화형 `bash` 셸로 실행한다.

   Bash

   ```
   jetson-containers run --rm -it $(autotag ros-humble-vtk-plc:r36) bash
   ```

   9

3. (선택적) `autotag`가 이미지를 올바르게 찾지 못하는 경우 57, 이미지 태그를 직접 명시적으로 지정하여 실행할 수 있다.58

   Bash

   ```
   jetson-containers run --rm -it ros-humble-vtk-plc:r36 bash
   ```



### 7.3. 수동 `docker run` 명령어 (참조용)



디버깅 또는 특정 옵션 수정이 필요한 경우, `jetson-containers run` 스크립트가 내부적으로 실행하는 전체 `docker run` 명령어는 다음과 유사하다.52

Bash

```
# 이 명령어는 'jetson-containers run' 스크립트가 자동으로 처리해주는
# 복잡한 인수를 보여주기 위한 참조용이다.
sudo docker run -it --rm \
    --runtime nvidia \
    --network host \
    --privileged \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix/:/tmp/.X11-unix \
    -v /tmp/.docker.xauth:/tmp/.docker.xauth \
    -e XAUTHORITY=/tmp/.docker.xauth \
    --device /dev \
    --volume /dev/bus/usb:/dev/bus/usb \
    ros-humble-vtk-plc:r36 \
    bash
```



## 섹션 8: 7단계: 통합 검증



컨테이너가 성공적으로 실행되고 `bash` 프롬프트가 나타나면, 모든 구성요소가 올바르게 작동하는지 검증한다.



### 8.1. 검증 1: ROS2, Qt5, GPU (OpenGL) 가속



이 테스트는 ROS2, Qt5, X11 포워딩, OpenGL GPU 가속 6이 모두 동시에 작동하는지 확인하는 가장 확실한 방법이다.

1. 컨테이너 내부에서 ROS2 환경을 활성화한다.

   Bash

   ```
   source /opt/ros/humble/setup.bash
   ```

2. Rviz2를 실행한다.

   Bash

   ```
   rviz2
   ```

- **예상 결과:** 호스트 데스크톱에 Rviz2 GUI 창이 성공적으로 나타나야 한다. "Global Options"의 "Fixed Frame"을 "map" (임의의 값)으로 설정했을 때 3D 뷰 영역에 격자(Grid)가 오류 없이 렌더링되어야 한다.
- **문제 해결:** 만약 `rviz2` 실행 시 `OgreWindow` 관련 오류 61가 발생하며 비정상 종료된다면, 이는 Rviz의 3D 렌더링 엔진(Ogre)이 GPU 가속 OpenGL 컨텍스트를 생성하지 못했음을 의미한다.
  - **진단:** 컨테이너 내부에서 `glxinfo | grep "OpenGL renderer"` 명령을 실행한다.
  - **정상:** 결과가 `NVIDIA Corporation...` (예: `NVIDIA Orin`)을 포함해야 한다.
  - **오류:** 결과가 `llvmpipe` 또는 `swrast` 62를 표시한다면, 이는 GPU 가속이 활성화되지 않은 것이다. 이 경우, 섹션 2.3의 `daemon.json` 설정과 섹션 7.2의 `jetson-containers run` 명령어가 올바르게 실행되었는지 확인해야 한다.



### 8.2. 검증 2: VTK (Python 및 C++)



1. **Python 바인딩 검증:** `python3-vtk9` 16가 올바르게 설치되었는지 확인한다.

   Bash

   ```
   python3 -c "import vtk; v = vtk.vtkVersion(); print(f'VTK Version: {v.GetVTKVersion()}')"
   ```

   - **예상 결과:** Ubuntu 22.04의 `python3-vtk9` 버전에 해당하는 버전 문자열(예: `9.1.0`)이 출력된다.37

2. **C++ 라이브러리 검증:** `libvtk9-dev` 41가 올바르게 설치되었는지 확인한다.

   Bash

   ```
   ldconfig -p | grep libvtkCommonCore-9.1
   ```

   - **예상 결과:** `libvtkCommonCore-9.1.so` 라이브러리 경로가 출력된다.



### 8.3. 검증 3: PLC 라이브러리



1. **Modbus 검증 (설치한 경우):**

   Bash

   ```
   ldconfig -p | grep libmodbus
   ```

   - **예상 결과:** `libmodbus.so` 17 라이브러리 경로가 출력된다.

2. **EtherNet/IP 검증 (설치한 경우):**

   Bash

   ```
   ldconfig -p | grep libplctag
   ```

   - **예상 결과:** `libplctag.so` 44 라이브러리 경로가 출력된다.



## 섹션 9: 결론





### 9.1. 보고서 요약



본 보고서는 Jetson AGX Orin (JP6/Ubuntu 22.04)에서 `jetson-containers` 프레임워크를 활용하여 ROS2 Humble, Qt5, VTK, PLC 라이브러리를 성공적으로 통합하는 "하이브리드 빌드 전략"을 수립하고 실행했다.

핵심 성공 요인은 `jetson-containers` 5를 사용하여 L4T에 최적화된 ROS2+Qt5 베이스를 먼저 구축하고, 이후 `apt` 패키지 호환성을 신중하게 분석하여 16 VTK와 PLC를 추가하는 사용자 정의 `Dockerfile` 12을 사용한 것이다. 또한, `jetson-containers run` 9 스크립트를 활용하여 복잡한 GUI/GPU 가속 실행을 자동화하는 전문가 수준의 워크플로우를 제시했다.



### 9.2. 향후 단계: 프로덕션 배포



본 보고서에서 생성된 이미지는 C++ 헤더(`-dev` 패키지 41)와 빌드 도구(`cmake`, `git` 51)를 포함하는 *개발용(development)* 이미지이다.

프로덕션 환경에 배포할 때는 이미지 크기를 최적화하고 보안을 강화하는 것이 필수적이다. 이를 위해 `Dockerfile`을 다단계 빌드(multi-stage build)로 리팩토링해야 한다. 첫 번째 단계(build stage)에서 소스 코드를 컴파일하고(`libplctag` 등) 필요한 `-dev` 패키지를 설치한 뒤, 두 번째 단계(final stage)에서는 첫 번째 단계에서 생성된 런타임 바이너리 및 라이브러리(예: `libvtk9.1`, `python3-vtk9`, `libmodbus.so.5`, `libplctag.so.*`)만을 복사하여 최소한의 런타임 이미지를 생성해야 한다.

개발 중 `docker commit` 14을 사용하여 변경 사항을 임시 저장할 수 있으나, 이는 재현성을 떨어뜨리므로 장기적으로는 본 보고서에서 제시한 `Dockerfile`을 지속적으로 개선하여 버전 관리하는 것이 바람직하다.

#### **Works cited**

1. Jetson AGX Orin for Next-Gen Robotics - NVIDIA, accessed November 15, 2025, https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-orin/
2. Running NVILA on Nvidia Jetson AGX Orin 64GB Development Kit with TinyChat, accessed November 15, 2025, https://faun.pub/running-nvila-on-nvidia-jetson-agx-orin-64gb-development-kit-with-tinychat-d442ff56814b
3. Install Ubuntu on NVIDIA Jetson, accessed November 15, 2025, https://ubuntu.com/download/nvidia-jetson
4. JetPack SDK - NVIDIA Developer, accessed November 15, 2025, https://developer.nvidia.com/embedded/jetpack-sdk-62
5. yunyao01/jetson-containers - Gitee, accessed November 15, 2025, https://gitee.com/yunyao01/jetson-containers
6. dusty-nv/ros_deep_learning: Deep learning inference nodes for ROS / ROS2 with support for NVIDIA Jetson and TensorRT - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/ros_deep_learning
7. Error with using TAO model · Issue #1623 · dusty-nv/jetson-inference - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-inference/issues/1623
8. Use These! Jetson Docker Containers Tutorial - JetsonHacks, accessed November 15, 2025, https://jetsonhacks.com/2023/09/04/use-these-jetson-docker-containers-tutorial/
9. dusty-nv/jetson-containers: Machine Learning Containers ... - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers
10. ROS2 Docker Containers for NVIDIA Jetson, accessed November 15, 2025, https://nvidia-ai-iot.github.io/ros2_jetson/ros2-jetson-dockers/
11. github.com, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/tree/master/packages
12. Combining Jetson Containers - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/combining-jetson-containers/292883
13. Using ROS2-Humble on a Jetson with Hardware acceleration - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/using-ros2-humble-on-a-jetson-with-hardware-acceleration/334990
14. How to install additional packages in the docker container? - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/how-to-install-additional-packages-in-the-docker-container/271324
15. How do I build a custom container? · Issue #350 · dusty-nv/jetson-containers - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/350
16. Getting Started - VTK documentation, accessed November 15, 2025, https://docs.vtk.org/en/latest/getting_started/index.html
17. libmodbus-dev : Jammy (22.04) : Ubuntu - Launchpad, accessed November 15, 2025, https://launchpad.net/ubuntu/jammy/+package/libmodbus-dev
18. Install ros 2 humble on jetson orin : r/ROS - Reddit, accessed November 15, 2025, https://www.reddit.com/r/ROS/comments/14ipjm3/install_ros_2_humble_on_jetson_orin/
19. Jetpack 6.1/6.2 Support · Issue #825 · dusty-nv/jetson-containers - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/825
20. Docker Setup On Jetson Orin - Includes JetPack 6 Docker fix - YouTube, accessed November 15, 2025, https://www.youtube.com/watch?v=d2I_wjJTekw
21. SSD + Docker - NVIDIA Jetson AI Lab, accessed November 15, 2025, https://www.jetson-ai-lab.com/tips_ssd-docker.html
22. Support for ROS2 Iron Irwini #279 - dusty-nv/jetson-containers - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/279
23. Connect Create® 3 to NVIDIA® Jetson™ and set up ROS 2 Galactic, accessed November 15, 2025, https://iroboteducation.github.io/create3_docs/setup/jetson/
24. Jetson-inference docker file - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/jetson-inference-docker-file/169336
25. Nvidia l4t-base missing cublas_v2.h - Jetson Xavier NX, accessed November 15, 2025, https://forums.developer.nvidia.com/t/nvidia-l4t-base-missing-cublas-v2-h/174582
26. L4t-base with cuDNN - Jetson TX2 - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/l4t-base-with-cudnn/220911
27. cudnn + tensorrt container · Issue #3 · dusty-nv/jetson-containers - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/3
28. ROS2 Nodes for Generative AI - NVIDIA Jetson AI Lab, accessed November 15, 2025, https://www.jetson-ai-lab.com/ros.html
29. Intro: Install and Setup ROS2 Humble - ROS2 Tutorial 1 - YouTube, accessed November 15, 2025, https://www.youtube.com/watch?v=0aPbWsyENA8
30. Ubuntu (deb packages) — ROS 2 Documentation: Humble documentation, accessed November 15, 2025, https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html
31. Installing desktop without updating Ubuntu 22.04 can break a desktop system. · Issue #1272, accessed November 15, 2025, https://github.com/ros2/ros2/issues/1272
32. Cannot install ROS2 humble on Ubuntu 22.04 (qt5 dependancies issue) - ROS Answers, accessed November 15, 2025, https://answers.ros.org/question/412321/
33. How can I fix an apparent Qt4 / Qt5 conflict? - Stack Overflow, accessed November 15, 2025, https://stackoverflow.com/questions/42955301/how-can-i-fix-an-apparent-qt4-qt5-conflict
34. ROS2 humble, not able to install common ros binaries? · Issue #309 · dusty-nv/jetson-containers - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/309
35. Setting up VTK, with python3.8 and Ubuntu 20.04 - Stack Overflow, accessed November 15, 2025, https://stackoverflow.com/questions/61754702/setting-up-vtk-with-python3-8-and-ubuntu-20-04
36. Will 22.04 LTS ship with Python 3.10? : r/Ubuntu - Reddit, accessed November 15, 2025, https://www.reddit.com/r/Ubuntu/comments/sehecb/will_2204_lts_ship_with_python_310/
37. Unable to build vtk 8 on ubuntu 22.04 - Support - VTK Discourse, accessed November 15, 2025, https://discourse.vtk.org/t/unable-to-build-vtk-8-on-ubuntu-22-04/14453
38. Initialization failed for vtkWebCore, not compatible with vtkmodules.vtkCommonCore, accessed November 15, 2025, https://discourse.vtk.org/t/initialization-failed-for-vtkwebcore-not-compatible-with-vtkmodules-vtkcommoncore/13423
39. Installation troubleshooting — ROS 2 Documentation: Humble documentation, accessed November 15, 2025, https://docs.ros.org/en/humble/How-To-Guides/Installation-Troubleshooting.html
40. resolve potentially conflicting python-vtk rosdep rules · Issue #10752 · ros/rosdistro - GitHub, accessed November 15, 2025, https://github.com/ros/rosdistro/issues/10752
41. vtk9 package : Ubuntu - Launchpad, accessed November 15, 2025, https://launchpad.net/ubuntu/+source/vtk9
42. tku-iarc/Hiwin_libmodbus - GitHub, accessed November 15, 2025, https://github.com/tku-iarc/Hiwin_libmodbus
43. C++ implementation of Modbus for ROS2 - GitHub, accessed November 15, 2025, https://github.com/openvmp/modbus
44. libplctag | libplctag - a library for PLC communication, accessed November 15, 2025, https://libplctag.github.io/
45. PLCIO Linux/Unix to PLC API Library, accessed November 15, 2025, https://www.ctiplcio.com/
46. query the APT cache - Ubuntu Manpage, accessed November 15, 2025, https://manpages.ubuntu.com/manpages/noble/man8/apt-cache.8.html
47. apt - How do I search for a package? - Ask Ubuntu, accessed November 15, 2025, https://askubuntu.com/questions/1181334/how-do-i-search-for-a-package
48. How to Search Packages With Apt Search Command - KodeKloud, accessed November 15, 2025, https://kodekloud.com/blog/apt-search-command/
49. apt-cache search : command not found - Ask Ubuntu, accessed November 15, 2025, https://askubuntu.com/questions/1447517/apt-cache-search-command-not-found
50. Getting started - libmodbus, accessed November 15, 2025, https://libmodbus.org/getting_started/
51. libplctag/libplctag: This C library provides a portable and simple API for accessing Allen-Bradley and Modbus PLC data over Ethernet. - GitHub, accessed November 15, 2025, https://github.com/libplctag/libplctag
52. Your First Jetson Container | NVIDIA Developer, accessed November 15, 2025, https://developer.nvidia.com/embedded/learn/tutorials/jetson-container
53. How to run GUI from docker file · Issue #36 · dusty-nv/jetson-containers - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/36
54. Start a Docker container at startup in jetson nano - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/start-a-docker-container-at-startup-in-jetson-nano/265760
55. Jetson Containers - Building DeepStream Images - Code Pyre, accessed November 15, 2025, https://codepyre.com/2019/07/building-deepstream-images/
56. /tmp/.docker.xauth got destroyed when computer shutdown · Issue #212 · dusty-nv/jetson-containers - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/212
57. Cannot run containers using jetson-containers run · Issue #787 - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-containers/issues/787
58. Jetson Container - model download - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/jetson-container-model-download/277730
59. How can I create my custom containers for Jetson Nano - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/how-can-i-create-my-custom-containers-for-jetson-nano/174053
60. I think i misunderstood what "jetson-containers run/build" can achieve vs what i expected, accessed November 15, 2025, https://forums.developer.nvidia.com/t/i-think-i-misunderstood-what-jetson-containers-run-build-can-achieve-vs-what-i-expected/323014
61. How to use Rviz2 with GPU in jetson docker container? - NVIDIA Developer Forums, accessed November 15, 2025, https://forums.developer.nvidia.com/t/how-to-use-rviz2-with-gpu-in-jetson-docker-container/343087
62. using VTK from within a Docker container on OS X (Intel & ARM), accessed November 15, 2025, https://discourse.vtk.org/t/using-vtk-from-within-a-docker-container-on-os-x-intel-arm/9375
63. Installing python packages #925 - dusty-nv/jetson-inference - GitHub, accessed November 15, 2025, https://github.com/dusty-nv/jetson-inference/issues/925



