# dusty-nv/jetson-containers 패키징 - 커스텀 APT 패키지 추가

2025-11-17, G25DR

`dusty-nv/jetson-containers` 리포지토리는 단순한 Dockerfile의 집합이 아니다.1 이것은 PyTorch, TensorFlow, ROS 등 수십 개의 AI 및 로보틱스 구성 요소를 JetPack-L4T 환경에 맞춰 모듈식으로 조합하고 빌드하는 정교한 컨테이너 빌드 시스템이다.2

따라서 '커스텀 `apt` 패키지 추가'라는 요구사항은 단일 `RUN apt-get install` 명령으로 해결되는 문제가 아니며, 이 모듈형 시스템의 아키텍처 내에서 '어느 계층'에 해당 종속성을 주입할 것인지 결정하는 엔지니어링적 선택이다.

이 보고서는 `jetson-containers` 환경에서 `apt` 패키지를 추가하는 세 가지 핵심 접근 방식을 제시한다. 각 방식은 재현성(Reproducibility), 빌드 시스템 통합성(Integration), 그리고 유지보수성(Maintainability) 관점에서 평가된다. 개발자는 자신의 요구 수준—단순 테스트, 애플리케이션 배포, 또는 플랫폼 재구축—에 따라 최적의 방법을 선택해야 한다.

### [표 1] APT 패키지 추가 전략 비교

| **접근 방식**                  | **재현성** | **빌드 시스템 통합성** | **권장 사용처**                                              |
| ------------------------------ | ---------- | ---------------------- | ------------------------------------------------------------ |
| **방법 1: `docker commit`**    | 없음       | 없음                   | 빠른 임시 테스트. 프로덕션 환경 절대 비권장.                 |
| **방법 2: 외부 Dockerfile**    | 높음       | 낮음 (의도적 분리)     | 최종 애플리케이션 및 관련 종속성 배포. (대부분의 사용자에게 권장) |
| **방법 3: 커스텀 패키지 정의** | 최상       | 높음 (플랫폼)          | 공통 시스템 종속성(예: `libcurl`) 또는 복잡한 `apt` 구성을 모듈화. (플랫폼 엔지니어링) |

------

## 1. 방법 1: 비권장 방식 - `docker commit`을 이용한 임시 수정

이 방식은 가장 빠르지만, `jetson-containers`의 핵심 철학을 정면으로 위배하므로 프로덕션 환경에는 절대 사용해서는 안 된다.

### 1.1. 프로세스

1. `jetson-containers run` 또는 표준 `docker run` 명령을 사용하여 베이스 컨테이너(예: `dustynv/l4t-pytorch`)를 대화형 셸(interactive shell) 모드로 시작한다.2
2. 실행 중인 컨테이너 내부 터미널에서 직접 `apt-get update` 및 `apt-get install -y <package_name>`을 실행하여 필요한 패키지를 설치한다.3
3. 컨테이너에서 `exit`로 빠져나온 후, 호스트 머신에서 `docker commit <container_id> <new_image_tag>` 명령을 실행한다. 이는 현재 컨테이너의 상태를 새로운 Docker 이미지로 저장(snapshot)하는 행위이다.4

### 1.2. 분석 및 비판

`jetson-containers`의 소유자(dusty_nv) 자신도 포럼에서 이 방식을 언급하기도 한다.4 하지만 이는 Docker 초보자를 위한 임시 해결책으로 제시된 것일 뿐이다. 그는 항상 "결국에는(Eventually) 당신의 변경 사항을 설치하는 자신만의 Dockerfile을 만들고 싶을 것"이라고 덧붙인다.4

`docker commit`의 치명적인 약점은 재현성(reproducibility)의 완전한 파괴에 있다.

1. **기술 부채(Technical Debt) 생성:** `apt` 패키지가 설치되었다는 이력(provenance)이 Dockerfile이라는 '소스 코드'에 남지 않는다.
2. **불투명한(Opaque) 레이어:** 설치된 패키지 목록, 버전, 이유는 오직 이미지를 '커밋'한 사람의 기억에만 의존하게 된다. 이는 명시적인 `Dockerfile`을 통해 '코드로써의 인프라(Infrastructure as Code)'를 구현하려는 `jetson-containers`의 목적과 정면으로 충돌한다.
3. **유지보수 불가:** 기반 `l4t-pytorch` 이미지가 업데이트되었을 때, 수동으로 설치한 `apt` 패키지를 동일하게 다시 설치하고 커밋하는 과정을 매번 반복해야 한다.

빠른 테스트 외의 목적으로는 이 방법을 사용하지 마라.

## 2. 방법 2: 권장 방식 (애플리케이션) - 외부 Dockerfile을 이용한 계층화

이 방식은 `jetson-containers`를 '플랫폼'으로 취급하고, 그 위에 사용자의 '애플리케이션'을 계층화(layering)하는 가장 표준적이고 권장되는 방법이다.

### 2.1. 아키텍처: 플랫폼과 애플리케이션의 분리

GitHub 이슈 스레드6에서 많은 사용자가 `jetson-containers`의 복잡한 빌드 스크립트(예: `build.sh --package-dirs` 6)를 파고들려다 혼란을 겪는다. 그러나 가장 성공적인 결론은 `jetson-containers`가 빌드한 이미지를 단순한 `FROM` 명령어로 사용하는 것이다.6

이는 `jetson-containers`가 의도한 '관심사의 분리(Separation of Concerns)'를 정확히 따르는 방식이다.

- **플랫폼 (`jetson-containers`):** CUDA, PyTorch, ROS 등 복잡한 AI/ML 스택을 제공한다.
- **애플리케이션 (사용자 Dockerfile):** 이 플랫폼 위에서 실행되며, 오직 애플리케이션 실행에 필요한 `apt` 패키지(예: `libcurl4-openssl-dev`, `vim`)와 Python 패키지만을 추가한다.

`apt` 패키지가 *오직* 당신의 최종 애플리케이션에만 필요하다면, 이 방식이 정답이다.

### 2.2. 단계별 실행

1. **베이스 이미지 확보:** `jetson-containers`를 사용하여 필요한 구성요소가 포함된 베이스 이미지를 빌드하거나 PULL한다. `autotag` 스크립트를 사용하면 현재 JetPack 버전에 맞는 태그를 자동으로 찾아준다.

   ```Bash
   # PULL 또는 BUILD를 통해 dustynv/l4t-pytorch:r36.2.0 같은 이미지를 로컬에 확보
   jetson-containers run $(autotag l4t-pytorch)
   ```

2. **외부 Dockerfile 생성:** `jetson-containers` 리포지토리 *외부*의 사용자 프로젝트 루트에 `Dockerfile`이라는 새 파일을 생성한다.

3. **Dockerfile 작성:**

   ```Dockerfile
   # 1. jetson-containers가 빌드한 베이스 이미지를 사용 
   ARG BASE_IMAGE=dustynv/l4t-pytorch:r36.2.0
   FROM ${BASE_IMAGE}
   
   # 2. 커스텀 APT 패키지 설치
   #    - DEBIAN_FRONTEND: 대화형 프롬프트(interactive prompt) 방지 
   #    - apt-utils: 'debconf' 경고 메시지 제거 
   #    - --no-install-recommends: 불필요한 종속성 설치 방지
   #    - rm -rf /var/lib/apt/lists/*: 이미지 레이어 크기 최적화 [8, 9]
   ENV DEBIAN_FRONTEND=noninteractive
   RUN apt-get update && \
       apt-get install -y --no-install-recommends \
           apt-utils \
           htop \
           vim \
           libcurl4-openssl-dev \
           libboost-python-dev \
       && rm -rf /var/lib/apt/lists/*
   
   # 3. 애플리케이션 코드 복사 및 Python 종속성 설치 
   WORKDIR /app
   COPY./requirements.txt./
   RUN pip3 install --no-cache-dir -r requirements.txt
   COPY..
   
   # 4. 컨테이너 실행 명령
   CMD ["python3", "my_app.py"]
   ```

4. **이미지 빌드:** `jetson-containers` 빌드 스크립트가 *아닌*, 표준 `docker build` 명령으로 이 이미지를 빌드한다.

   ```Bash
   sudo docker build -t my-application:latest.
   ```

### 2.3. 결론

이 방식은 재현 가능하며, `jetson-containers`의 복잡한 내부 빌드 시스템을 알 필요가 없다. 대부분의 사용 사례에 이 방식을 사용하라.

## 3. 방법 3: 궁극의 방식 (종속성) - `jetson-containers` 커스텀 패키지 정의

이 방식은 추가하려는 `apt` 패키지 자체가 `opencv`나 `ros`처럼 재사용 가능한 '플랫폼 구성요소(component)'일 경우 사용하는 가장 강력하고 통합적인 방법이다.

### 3.1. 사용 시나리오

- 추가하려는 `apt` 패키지가 여러 다른 모듈(예: `pytorch`와 `transformers`)의 *공통 종속성*일 경우.
- `apt` 패키지 설치가 매우 복잡하여(예: 서드파티 PPA 추가, GPG 키 등록) 이 과정을 별도의 모듈로 분리하고 싶을 경우.
- `jetson-containers build...` 명령 한 줄로 모든 `apt` 종속성까지 포함된 이미지를 빌드하고 싶을 경우.

### 3.2. 핵심 원리: `apt` 설치를 패키지로 취급

`jetson-containers`의 빌드 로그를 분석하면 10, `cmake:apt` 또는 `build-essential` 같은 패키지 빌드 단계를 발견할 수 있다.

`packages/build/build-essential/Dockerfile`의 내용 9을 확인하면, 이 Dockerfile의 핵심은 `RUN... apt-get install -y... build-essential...` 9을 실행하는 것임을 알 수 있다.

이는 `jetson-containers`가 `apt` 패키지 설치 자체를 `pytorch`와 동일한 하나의 독립된 '패키지' 또는 '구성요소'로 취급함을 의미한다. 따라서 우리도 동일한 방식으로 커스텀 `apt` 패키지를 정의할 수 있다.

### 3.3. 단계별 실행: `my-apt-deps` 패키지 생성하기

1. `jetson-containers` 리포지토리 내에 새로운 패키지 디렉토리를 생성한다. (리포지토리를 포크(fork)하여 관리하는 것을 권장한다.)

   ```Bash
   cd /path/to/jetson-containers
   mkdir -p packages/my-apt-deps
   ```

2. `packages/my-apt-deps/Dockerfile` 파일을 생성하고 다음 내용을 작성한다. 이것은 `jetson-containers`의 패키지 Dockerfile 표준 양식을 따른다.

   Dockerfile

   ```
   # 모든 jetson-containers 패키지 Dockerfile의 표준 시작 
   ARG BASE_IMAGE
   FROM ${BASE_IMAGE}
   
   # 대화형 프롬프트 방지
   ENV DEBIAN_FRONTEND=noninteractive
   
   # 커스텀 APT 패키지 설치
   # 여기서 PPA 추가, 키 등록 등 복잡한 로직 수행 가능
   RUN apt-get update && \
       apt-get install -y --no-install-recommends \
           htop \
           vim \
           nmap \
           libboost-all-dev \
       && rm -rf /var/lib/apt/lists/*
   ```

3. *(선택 사항)* `packages/my-apt-deps/__config__.py` 파일을 생성하여 `pytorch`와 같은 다른 패키지에 대한 종속성을 명시적으로 정의할 수 있다. 하지만 단순 `apt` 설치는 Dockerfile만으로 충분하다.

4. 이제 `jetson-containers build` 명령에 새로 만든 패키지(`my-apt-deps`)를 다른 패키지들과 함께 나열한다.

   Bash

   ```
   # l4t-pytorch와 transformers, 그리고 우리가 만든 my-apt-deps를 포함하는
   # 'my-custom-image'라는 이름의 새 컨테이너를 빌드하라.
   jetson-containers build --name=my-custom-image \
                                 l4t-pytorch \
                                 transformers \
                                 my-apt-deps
   ```

### 3.4. 결과

`jetson-containers` 빌드 시스템은 종속성 트리를 분석하여, `l4t-pytorch`와 `transformers`를 빌드하는 과정에 `my-apt-deps` Dockerfile을 삽입하여 실행한다. 이제 `htop`, `vim` 등은 애플리케이션 계층이 아닌 '플랫폼'의 일부가 된다.

`--package-dirs` 인수 6는 `packages/` 디렉토리 *외부에* 있는 패키지 정의를 빌드 시스템에 알려주는 데 사용되지만, 리포지토리를 포크하여 `packages/` 내부에 패키지를 생성하는 것이 더 명시적이고 관리가 용이하다.

## 4. 고급 주제: `apt` 패키지 관리를 위한 MLOps 최적화

`apt` 패키지를 반복적으로 빌드하는 CI/CD 또는 MLOps 환경에서는 설치 속도와 유연성이 중요하다.

### 4.1. 로컬 APT 캐시 서버 활용 (성능 최적화)

`jetson-containers` 리포지토리 루트에는 `docker-compose.yaml` 파일이 존재한다.2 이 파일의 커밋 로그는 "Add local PyPI/APT compose stack" 2을 언급한다.

이는 `apt-cacher-ng`와 로컬 PyPI 서버를 Docker 컨테이너로 실행하여, `apt-get install` 이나 `pip install` 시 패키지 다운로드를 호스트에 캐시하기 위함이다.

```Bash
# jetson-containers 루트 디렉토리에서 실행
docker-compose up -d
```

이 캐시 서버가 실행 중이면, `jetson-containers` 빌드 스크립트는 이를 자동으로 감지하여 사용한다. 동일한 `apt` 패키지를 수십 번 설치해야 하는 CI/CD 파이프라인에서 빌드 시간을 극적으로 단축시킬 수 있다.

### 4.2. 빌드 인수(Build ARGs)를 통한 동적 `apt` 설치

`jetson-containers`는 `--build-arg` 플래그13를 Dockerfile 내부로 전달하는 `--build-flags` 옵션을 지원한다. '방법 3'의 Dockerfile을 수정하여 설치할 패키지 목록을 동적으로 주입할 수 있다.

**Dockerfile (`packages/my-apt-deps/Dockerfile`):**

```Dockerfile
ARG BASE_IMAGE
FROM ${BASE_IMAGE}

# 설치할 패키지 목록을 빌드 인수로 받음
ARG EXTRA_APT_PACKAGES=""

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && \
    apt-get install -y --no-install-recommends ${EXTRA_APT_PACKAGES} \
    && rm -rf /var/lib/apt/lists/*
```

**빌드 명령:**

```Bash
jetson-containers build my-package \
    --build-flags="--build-arg EXTRA_APT_PACKAGES='htop vim nmap'"
```

이 방식은 단일 커스텀 패키지 Dockerfile을 유지하면서도 다양한 `apt` 패키지 조합을 테스트할 수 있게 한다.

## 5. 일반적인 `apt` 관련 빌드 오류 및 문제 해결

### 오류 1: `apt-get update` 실패 (404 Not Found, E:...)

- **증상:** `apt-get update` 실행 중 특정 리포지토리를 찾지 못하거나 404 오류 발생.15
- **원인:** 1. 컨테이너의 DNS 또는 네트워크 설정 오류. 2. 호스트의 JetPack 버전과 컨테이너의 `l4t-base` 태그가 불일치하여 존재하지 않는 `apt` 리포지토리를 참조함. 3. NVIDIA 또는 Ubuntu의 `apt` 리포지토리가 일시적으로 다운됨.15
- **해결:**
  1. `jetson-containers` 빌드 시 기본값인 `--network=host` 9 옵션이 사용되는지 확인하라.
  2. Dockerfile에서 `apt-get update` 전에 `apt-get clean && rm -rf /var/lib/apt/lists/*`를 먼저 실행하여 오래된 캐시를 완전히 삭제하라.
  3. `autotag` 스크립트를 사용하거나 베이스 이미지의 L4T 태그(예: `r35.4.1`)가 호스트 JetPack 버전과 일치하는지 확인하라.

### 오류 2: OCI 런타임 오류 (Shim, Seccomp)

- **증상:** 빌드 또는 실행 시 `failed to create shim: OCI runtime create failed` 8, `error adding seccomp filter rule... permission denied` 8 또는 `nvidia-container-cli: mount error` 17 로그 발생.
- **원인:** 이는 Dockerfile 내부의 `apt` 명령 문제가 아니라, 호스트 시스템의 Docker 데몬 또는 `nvidia-container-runtime` 18 설정이 손상되었을 때 발생한다. 특히 호스트에서 `sudo apt-get upgrade`를 실행한 후 `nvidia-container-runtime`과 Docker 버전 간의 호환성이 깨졌을 때 자주 발생한다.16
- **해결:**
  1. NVIDIA 공식 지침 17에 따라 `nvidia-docker2` 및 관련 패키지를 재설치하라.
  2. `sudo systemctl restart docker` 명령으로 Docker 데몬을 재시작하라.3
  3. 문제가 지속되면 JetPack을 재설치(reflash)하는 것이 가장 확실하고 빠른 해결책이다.8

### 오류 3: `debconf: delaying package configuration, since apt-utils is not installed`

- **증상:** `apt-get install` 중 위와 같은 경고 메시지가 출력됨.11
- **원인:** 이는 오류가 아닌 경고이다. `apt-utils` 패키지가 없어 `apt` 설치 과정 중 발생할 수 있는 대화형 프롬프트를 처리할 수 없다는 의미이다.
- **해결:**
  1. Dockerfile 상단에 `ENV DEBIAN_FRONTEND=noninteractive` 8를 선언하여 모든 대화형 프롬프트를 원천 차단하라.
  2. `apt-get install -y apt-utils`를 다른 패키지 목록에 함께 추가하여 경고 메시지를 제거하라.

## 결론: 시나리오별 최종 권고

`dusty-nv/jetson-containers`에 `apt` 패키지를 추가하는 방법은 목적에 따라 명확히 구분되어야 한다.

1. **빠른 테스트 및 디버깅:** '방법 2 (외부 Dockerfile)'를 사용하라. `docker commit`보다 빠르고 표준적이며 재현 가능하다.
2. **단일 애플리케이션 배포:** '방법 2 (외부 Dockerfile)'가 정답이다. 플랫폼(jetson-containers)과 애플리케이션(사용자 Dockerfile)을 명확히 분리하여 유지보수성을 극대화하라.
3. **새로운 기반 플랫폼 구축:** `libboost`나 `libcurl`처럼 여러 하위 모듈(pytorch, ros)에서 공통으로 요구하는 시스템 라이브러리를 추가해야 하는가? 이때는 '방법 3 (커스텀 패키지 정의)'을 사용하라. `packages/` 디렉토리에 `my-common-deps` 같은 모듈을 정의하고, `jetson-containers build`로 통합 빌드하라.
4. **CI/CD 및 반복 빌드:** '방법 3'과 '고급 주제 (로컬 APT 캐시)' 2를 조합하여 빌드 속도를 최적화하고 재현성을 극대화하라.

어떠한 경우에도 '방법 1 (`docker commit`)' 4은 재현성을 파괴하고 기술 부채를 생성하므로, 최종 워크플로우에 포함시키지 마라.

#### **Works cited**

1. Dustin Franklin dusty-nv - GitHub, accessed November 16, 2025, https://github.com/dusty-nv
2. dusty-nv/jetson-containers: Machine Learning Containers for NVIDIA Jetson and JetPack-L4T - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers
3. Connect Create® 3 to NVIDIA® Jetson™ and set up ROS 2 Galactic, accessed November 16, 2025, https://iroboteducation.github.io/create3_docs/setup/jetson/
4. How to install additional packages in the docker container? - NVIDIA Developer Forums, accessed November 16, 2025, https://forums.developer.nvidia.com/t/how-to-install-additional-packages-in-the-docker-container/271324
5. Installing python packages #925 - dusty-nv/jetson-inference - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-inference/issues/925
6. How do I build a custom container? · Issue #350 · dusty-nv/jetson-containers - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers/issues/350
7. Help needed with custom docker image based on dustynv/ros:humble-ros-base-l4t-r32.7.1 · Issue #539 · dusty-nv/jetson-containers - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers/issues/539
8. Running the docker/build.sh doesn't works - Jetson Xavier NX - NVIDIA Developer Forums, accessed November 16, 2025, https://forums.developer.nvidia.com/t/running-the-docker-build-sh-doesnt-works/220462
9. cannot build containers, python_install.sh fails (Orin AGX 64GB, JetPack 6.1) · Issue #654 · dusty-nv/jetson-containers - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers/issues/654
10. Jetson-containers keep exiting with error code but not sure what it means - Page 2, accessed November 16, 2025, https://forums.developer.nvidia.com/t/jetson-containers-keep-exiting-with-error-code-but-not-sure-what-it-means/320624?page=2
11. "jetson-containers build deepstream" on a jetson orin nano with jetpack 6.2 fails to build · Issue #1117 · dusty-nv/jetson-containers - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers/issues/1117
12. I think i misunderstood what "jetson-containers run/build" can achieve vs what i expected, accessed November 16, 2025, https://forums.developer.nvidia.com/t/i-think-i-misunderstood-what-jetson-containers-run-build-can-achieve-vs-what-i-expected/323014
13. Issue #1007 · dusty-nv/jetson-containers - opencv build fails - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers/issues/1007
14. Unable to build custom Docker · Issue #480 · dusty-nv/jetson-containers - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers/issues/480
15. Unable to install various packages from jetson common apt repositories, accessed November 16, 2025, https://forums.developer.nvidia.com/t/unable-to-install-various-packages-from-jetson-common-apt-repositories/337319
16. Docker containers won't run after recent apt-get upgrade - Jetson Nano, accessed November 16, 2025, https://forums.developer.nvidia.com/t/docker-containers-wont-run-after-recent-apt-get-upgrade/194369
17. running the docker/build.sh doesn't works · Issue #1426 · dusty-nv/jetson-inference - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-inference/issues/1426
18. Installing docker container runtime for jetson nano - NVIDIA Developer Forums, accessed November 16, 2025, https://forums.developer.nvidia.com/t/installing-docker-container-runtime-for-jetson-nano/203424
19. NVIDIA Container Runtime on Jetson (Beta) — Cloud Native Products documentation, accessed November 16, 2025, https://nvidia.github.io/container-wiki/toolkit/jetson.html
20. Docker fails to create container after upgrading docker on Jetpack 4.9 · Issue #108 · dusty-nv/jetson-containers - GitHub, accessed November 16, 2025, https://github.com/dusty-nv/jetson-containers/issues/108
21. Combining Jetson Containers - NVIDIA Developer Forums, accessed November 16, 2025, https://forums.developer.nvidia.com/t/combining-jetson-containers/292883