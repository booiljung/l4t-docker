# Docker Buildx - AMD64에서 Jetson까지 네이티브 및 크로스 플랫폼 빌드

2025-11-16, G25DR





## I. `docker buildx` 및 BuildKit 아키텍처 심층 분석





### `docker buildx`란 무엇인가: 기존 `docker build`를 넘어서



`docker buildx`는 Docker CLI(Command Line Interface)의 공식 플러그인으로, Docker의 차세대 빌드 엔진인 BuildKit의 모든 고급 기능을 사용자에게 제공합니다.1 이는 기존의 `docker build` 명령을 확장하여, 멀티 플랫폼 이미지 빌드(예: `linux/amd64` 및 `linux/arm64` 동시 지원) 4, 빌드 단계의 병렬 실행 5, 그리고 향상된 캐시 관리 2와 같은 강력한 기능들을 지원합니다. `buildx`는 사용자가 상호작용하는 CLI 도구이며, 실제 빌드 작업은 백그라운드에서 실행되는 `buildkitd` 데몬 컨테이너에 의해 수행됩니다.6



### 핵심 엔진, BuildKit: 병렬 처리, 고급 캐싱 및 드라이버 모델



BuildKit은 Moby 프로젝트의 일부로 개발된 차세대 빌드 엔진으로, 기존 Docker 빌더 대비 월등한 성능, 효율성 및 유연성을 제공합니다.2 BuildKit의 핵심 아키텍처적 이점은 다음과 같습니다.

- **병렬 스테이지 실행:** Dockerfile 내에서 서로 의존성이 없는 빌드 단계(stage)들을 동시에 병렬로 처리하여 전체 빌드 시간을 획기적으로 단축시킵니다.5
- **고급 캐싱:** 빌드 캐시를 원격 레지스트리로 내보내거나(`--cache-to`) 8, 로컬 파일 시스템에 저장하여 9 CI/CD 파이프라인의 여러 단계 또는 여러 개발자 간에 캐시를 효율적으로 공유할 수 있습니다.
- **드라이버 모델:** BuildKit이 실행될 백엔드 환경을 '드라이버(driver)'라는 개념으로 추상화합니다.7 이를 통해 사용자는 `docker-container` (전용 컨테이너 사용), `kubernetes` (Kubernetes 클러스터 사용), `remote` (원격 BuildKit 데몬 연결) 등 다양한 실행 환경을 선택할 수 있습니다.7



### `buildx` 사용의 명확한 이점: 왜 네이티브 빌드에도 `buildx`를 사용해야 하는가



최신 Docker 버전(19.03 이상)에서 `docker build` 명령은 내부적으로 `docker buildx build`의 별칭(alias)으로 작동하는 경우가 많습니다.12 하지만 여기에는 사용자가 반드시 인지해야 할 결정적인 차이가 존재합니다. `docker build` 명령은 기본적으로 `default`라는 이름의 빌더 인스턴스를 사용하며, 이 인스턴스는 `docker`라는 *레거시 드라이버*를 사용하도록 설정되어 있습니다.7

이 `docker` 드라이버는 Docker 데몬에 내장된 BuildKit 라이브러리를 사용하지만, `buildx`의 핵심 기능인 멀티 플랫폼 빌드나 고급 캐시 내보내기(`--cache-to`) 같은 고급 기능들을 **지원하지 않습니다**.8 따라서 사용자가 Jetson용 이미지 빌드(시나리오 1)와 같은 `buildx`의 진정한 잠재력을 활용하기 위해서는, `default` 빌더가 아닌 `docker-container` 드라이버를 사용하는 새로운 빌더 인스턴스를 명시적으로 생성하고 활성화해야 합니다.

더 나아가, 사용자의 두 번째 시나리오인 네이티브 `linux/amd64` 빌드 역시 `docker-container` 드라이버 기반의 `buildx`를 사용함으로써 큰 이점을 얻을 수 있습니다. BuildKit의 병렬 실행 기능 2은 크로스 컴파일에만 국한되지 않습니다. 예를 들어, 멀티 스테이지 Dockerfile에서 `build-stage`와 `test-stage`가 서로 독립적이라면, BuildKit은 이 두 단계를 동시에 실행하여 네이티브 빌드의 속도를 향상시킵니다.

결론적으로, Jetson 빌드(시나리오 1)뿐만 아니라 네이티브 AMD64 빌드(시나리오 2)의 성능과 효율성을 모두 최적화하기 위해서는, 기존의 `docker build` 워크플로우를 `docker buildx build` (새 빌더 사용)로 완전히 대체하는 것이 강력히 권장됩니다.



## II. 핵심 환경 설정: 멀티 플랫폼 빌더 및 QEMU 에뮬레이션



`docker buildx`를 사용하여 사용자의 두 시나리오, 특히 Jetson용 크로스 컴파일을 수행하기 위해서는 두 가지 핵심 설정이 선행되어야 합니다.



### 빌더(Builder) 인스턴스 이해: 드라이버 비교 (`docker` vs. `docker-container`)



빌더는 BuildKit 데몬이 실행되는 격리된 환경입니다.7 `docker buildx ls` 명령을 사용하여 현재 사용 가능한 빌더 인스턴스 목록을 확인할 수 있으며, 목록에서 `*` 표시는 현재 CLI 세션에서 활성화된(selected) 빌더를 나타냅니다.3

- **`docker` 드라이버 (기본값):** `default` 빌더가 사용하는 드라이버입니다. Docker 데몬에 번들된 BuildKit 라이브러리를 사용하며 7, 별도 설정이 필요 없는 대신 멀티 플랫폼 빌드 및 캐시 내보내기가 불가능합니다.8
- **`docker-container` 드라이버:** BuildKit 데몬을 전용 Docker 컨테이너(보통 `buildkitd`라는 이름으로 실행됨)로 실행합니다.6 이 방식은 멀티 플랫폼 빌드, 고급 캐싱 등 모든 BuildKit 기능을 완벽하게 지원합니다. `docker buildx create` 명령 실행 시 `--driver` 플래그를 생략하면 이 `docker-container` 드라이버가 기본값으로 사용됩니다.10

사용자가 요청한 Jetson 빌드를 포함한 모든 고급 빌드 기능을 사용하려면 `docker-container` 드라이버를 사용하는 새 빌더 생성이 필수적입니다.



#### 테이블 1: 빌더 드라이버 비교 (`docker` vs. `docker-container`)



| **기능**                         | **docker 드라이버 (default 빌더)**     | **docker-container 드라이버 (신규 빌더)** |
| -------------------------------- | -------------------------------------- | ----------------------------------------- |
| **백엔드**                       | Docker 데몬 내장 BuildKit 라이브러리 7 | 전용 `buildkitd` 컨테이너 7               |
| **멀티 플랫폼 빌드**             | **불가** 13                            | **가능** 13                               |
| **`--platform` 지정**            | 단일 네이티브 플랫폼만 가능            | 다중 플랫폼 지정 가능                     |
| **`--push` (멀티 플랫폼)**       | 불가                                   | 가능                                      |
| **`--load` (멀티 플랫폼)**       | 해당 없음                              | **불가** (지원 안 됨) 14                  |
| **캐시 내보내기 (`--cache-to`)** | **불가** 8                             | **가능**                                  |
| **권장 사용 사례**               | 간단한 로컬 네이티브 빌드              | **모든 CI/CD 및 멀티 플랫폼 빌드**        |



### 단계별 설정 (1): `docker-container` 드라이버를 사용한 신규 빌더 생성, 검사 및 활성화



다음은 Ubuntu 22.04 호스트에서 `docker-container` 드라이버를 사용하는 새 빌더를 설정하는 3단계 프로세스입니다.

1. **빌더 생성:** `mybuilder`라는 이름의 새 빌더 인스턴스를 생성합니다.4

   Bash

   ```
   $ docker buildx create --name mybuilder --driver docker-container
   ```

   - `--name`: 빌더 인스턴스의 고유한 이름을 지정합니다.11
   - `--driver docker-container`: `docker-container` 드라이버 사용을 명시합니다. (현재 기본값이지만 명시하는 것이 좋습니다).10

2. **빌더 활성화 (Use):** 생성한 `mybuilder`를 현재 CLI 세션의 기본 빌더로 설정합니다.4

   Bash

   ```
   $ docker buildx use mybuilder
   ```

   - *팁:* `docker buildx create --name mybuilder --use`와 같이 `--use` 플래그를 함께 사용하면 생성과 활성화를 동시에 수행할 수 있습니다.4

3. **부트스트랩 및 검사:** 빌더 컨테이너를 강제로 시작(부트스트랩)하고, 빌더가 지원하는 플랫폼 목록을 확인합니다.

   Bash

   ```
   $ docker buildx inspect --bootstrap
   ```

   - `--bootstrap`: 빌더 인스턴스가 아직 실행 중이 아닐 경우, 이 명령을 통해 BuildKit 데몬 컨테이너를 시작합니다.11
   - 명령 실행 후 출력되는 `Platforms` 필드를 주의 깊게 확인해야 합니다. QEMU 설정이 올바르게 완료되었다면, 이 목록에 `linux/amd64`, `linux/arm64`, `linux/arm/v7` 등이 포함되어 있어야 합니다.7



### 단계별 설정 (2): Ubuntu 22.04에서 QEMU 및 `binfmt_misc` 활성화



사용자의 AMD64(x86_64) CPU는 Jetson의 ARM64 아키텍처용 바이너리를 직접 실행할 수 없습니다.13 `docker buildx build`가 `linux/arm64` 이미지를 빌드하는 동안 `RUN apt-get update`와 같은 명령을 실행해야 할 때, QEMU라는 에뮬레이터가 이 ARM64 명령을 실시간으로 AMD64 명령으로 번역해주는 과정이 필요합니다.

`binfmt_misc`는 리눅스 커널의 기능으로, 특정 형식(예: ARM64 ELF 바이너리)의 파일을 실행하려 할 때, 커널이 자동으로 지정된 에뮬레이터(QEMU)를 호출하도록 등록해주는 메커니즘입니다.15

Ubuntu 22.04 호스트에서 가장 안정적으로 QEMU 에뮬레이션을 설정하는 방법은 **두 가지 단계**를 모두 수행하는 것입니다.

1. QEMU 바이너리 및 커널 지원 설치:

   먼저, 호스트 시스템에 QEMU 에뮬레이션 실행 파일(qemu-aarch64-static 등)과 binfmt_misc 커널 모듈 지원을 설치합니다.

   Bash

   ```
   $ sudo apt-get update
   $ sudo apt-get install -y qemu-user-static binfmt-support
   ```

   17

2. Docker를 위한 binfmt_misc 등록:

   다음으로, Docker 컨테이너가 이 에뮬레이터를 투명하게 사용할 수 있도록 커널의 binfmt_misc 핸들러를 등록하고 활성화합니다.

   Bash

   ```
   $ docker run --privileged --rm tonistiigi/binfmt --install all
   ```

   15 (또는 `multiarch/qemu-user-static --reset -p yes` 20 이미지도 동일한 기능을 수행합니다).

   - *경고:* 이 명령은 `--privileged` 플래그를 요구하며 호스트의 커널 설정을 변경합니다. `root` 권한이 필요할 수 있으며, 권한 부족 시 `operation not permitted` 오류가 발생할 수 있습니다.22

3. 설정 확인:

   aarch64(ARM64) 핸들러가 F (Fix Binary) 플래그와 함께 올바르게 등록되었는지 확인합니다. 이 F 플래그는 커널이 컨테이너 내부에서도 에뮬레이션을 투명하게 처리할 수 있도록 보장하는 중요한 설정입니다.15

   Bash

   ```
   $ cat /proc/sys/fs/binfmt_misc/qemu-aarch64
   ```

   출력 결과에 `flags: OCF` (또는 `flags:` 목록에 `F`가 포함)가 표시되어야 합니다.17



## III. 시나리오 1: Jetson (linux/arm64) 크로스 컴파일 빌드





### 요구사항 분석



첫 번째 시나리오는 AMD64 호스트에서 Jetson 플랫폼(NVIDIA ARM64)용 이미지를 빌드하는 것입니다. 이는 `docker buildx build` 명령에 `linux/arm64` 플랫폼을 타겟팅하도록 지시하는 것을 의미합니다.



### 기본 실행



II 섹션의 `mybuilder` 활성화 및 QEMU 설정이 완료되었다면, `--platform linux/arm64` 플래그를 추가하여 Jetson용 빌드를 시작할 수 있습니다.1 빌드된 결과물을 처리하는 방식에는 두 가지 주요 전략이 있으며, 각각 명확한 사용 사례와 한계를 가집니다.



### 빌드 결과물 처리 전략 (1): `--push`를 사용한 레지스트리 배포



이 전략은 크로스 컴파일된 이미지를 빌드함과 동시에 Docker Hub, AWS ECR, GCR 또는 사설 레지스트리로 즉시 푸시합니다.

- `--push` 옵션은 `--output=type=registry`의 축약형(shorthand)입니다.1

- **명령 예시:**

  Bash

  ```
  $ docker buildx build --platform linux/arm64 \
      -t your-registry/jetson-app:latest \
      --push.
  ```

  1

- 워크플로우 분석:

  크로스 컴파일 워크플로우에서 --push가 표준인 이유가 있습니다. linux/arm64 아키텍처로 빌드된 이미지는 AMD64 호스트에서 docker run으로 실행할 수 없습니다. 실행 시도 시, 호스트 CPU가 바이너리 형식을 이해할 수 없어 exec format error가 발생하게 됩니다.28 따라서 빌드된 이미지를 로컬 docker images 목록으로 가져오는 것(--load)은 런타임 테스트에 아무런 이점이 없습니다.

  이 이미지의 유일한 소비자는 타겟 아키텍처(즉, `linux/arm64`를 실행하는 Jetson 장치)입니다. Jetson 장치가 이 이미지에 액세스하는 유일하고 표준적인 방법은 레지스트리에서 `docker pull` 명령을 실행하는 것입니다.31 그러므로 AMD64 호스트에서의 크로스 컴파일 빌드의 논리적인 귀결은 항상 레지스트리로의 `--push`가 됩니다.



### 빌드 결과물 처리 전략 (2): `--load`의 한계와 단일 플랫폼 로드



`--load` 옵션은 `--output=type=docker`의 축약형으로 9, 빌드 결과를 로컬 Docker 데몬의 이미지 저장소로 로드하여 `docker images` 목록에서 볼 수 있게 합니다.

- 단일 *비-네이티브* 플랫폼(예: `--platform linux/arm64`)을 빌드할 때 `--load`를 사용하는 것 자체는 기술적으로 가능합니다.33

- **치명적인 한계:**

  1. **로컬 실행 불가:** 위에서 언급했듯이, `docker images` 목록에 `linux/arm64` 이미지가 로드되더라도, `docker run`으로 실행하면 `exec format error`가 발생합니다.29
  2. **멀티 플랫폼 로드 불가:** *여러* 플랫폼(예: `linux/amd64,linux/arm64`)을 동시에 빌드할 때 `--load` 옵션을 사용하는 것은 명시적으로 **지원되지 않습니다**.9

  이 지원 중단은 `docker-container` 드라이버의 작동 방식 및 Docker 데몬의 이미지 저장소 한계와 관련이 있습니다.15 멀티 플랫폼 빌드의 결과물은 단일 이미지가 아니라, 여러 아키텍처별 이미지를 가리키는 *매니페스트 리스트(manifest list)*입니다. `docker-container` 드라이버는 이 매니페스트 리스트를 생성할 수 있지만, Docker 데몬의 *기본* 이미지 저장소(classic image store)는 이 매니페스트 리스트 구조를 저장하도록 설계되지 않았습니다. 따라서 `--load`는 단일 플랫폼 이미지만 로컬 데몬으로 가져올 수 있습니다. (참고: Docker 데몬의 이미지 저장소를 `containerd`로 전환하면 이 제한이 해제될 수 있으나, 이는 Ubuntu 22.04의 기본 설정이 아닙니다 15).



### 특수 고려 사항: Jetson 이미지와 CUDA 종속성 처리



사용자의 "Jetson 플랫폼" 쿼리는 단순한 `linux/arm64` 빌드 이상의 복잡성을 내포합니다. Jetson 장치는 NVIDIA CUDA 기술에 크게 의존하며, 관련 Docker 이미지는 특별한 처리가 필요합니다.35

표준 NVIDIA Jetson 베이스 이미지(예: `nvcr.io/nvidia/l4t-base`)는 런타임 시 `nvidia-container-toolkit`을 통해 호스트(Jetson 장치)의 CUDA 드라이버와 라이브러리가 컨테이너 내부로 *주입*될 것을 전제로 합니다.35

`docker buildx build`를 실행하는 AMD64 호스트(또는 CI/CD 서버)에는 이러한 런타임 주입 메커니즘이 없습니다. 따라서 `Dockerfile` 내의 `RUN` 명령이 CUDA 컴파일러(`nvcc`)나 `libcublas.so` 같은 라이브러리를 필요로 하면(예: C++ 애플리케이션 컴파일), 해당 파일을 찾을 수 없어 빌드가 실패합니다.35

이 문제를 해결하기 위한 표준 패턴은 **2단계(multi-stage) 빌드**를 사용하는 것입니다 35:

1. **`build` 스테이지:** `l4t-base` 이미지와 같은 Jetson 베이스 이미지에서 시작하되, `apt-get install`을 통해 `cuda-compiler-10-2`, `libcufft-10-2`, `libcublas-dev-10-2` 등 *모든* 빌드 타임 CUDA 종속성을 명시적으로 설치합니다.35 이 스테이지는 애플리케이션 컴파일을 위해 필요한 모든 도구를 포함하므로 매우 무거워집니다.
2. **`final` 스테이지:** 다시 *깨끗한* `l4t-base` 런타임 이미지를 `FROM` 명령으로 사용합니다.
3. **`COPY`:** `build` 스테이지에서 컴파일된 최종 애플리케이션 바이너리(들)만 `final` 스테이지로 복사합니다. (예: `COPY --from=build /app/my_cuda_app /app/`)

이 접근 방식은 `build` 스테이지에서 QEMU 에뮬레이션을 통해 모든 CUDA 종속성을 해결하여 컴파일을 성공시키고, `final` 스테이지는 경량화된 상태를 유지하며 런타임 시 Jetson 호스트의 CUDA 주입에 올바르게 의존하도록 보장합니다.35



## IV. 시나리오 2: 네이티브 (linux/amd64) 빌드





### 요구사항 분석



두 번째 시나리오는 AMD64 호스트에서 호스트 자체(AMD64)에서 실행될 이미지를 빌드하는 것입니다. 이는 `docker buildx build`의 가장 기본적인 사용 사례입니다.



### `buildx`를 사용한 네이티브 빌드



I 섹션에서 분석했듯이, 네이티브 빌드라 할지라도 `docker-container` 드라이버를 사용하는 `buildx`는 BuildKit의 병렬 처리 및 고급 캐싱 이점을 제공하므로 2 사용이 권장됩니다.

플랫폼 명시의 중요성:

네이티브 빌드임에도 불구하고 --platform linux/amd64 플래그를 명시적으로 지정하는 것이 모범 사례입니다.15

- 플랫폼을 명시하지 않으면 `TARGETPLATFORM`과 같은 Dockerfile 내 변수들이 호스트의 `BUILDPLATFORM` 값으로 기본 설정됩니다.13
- `--platform linux/amd64`를 명시적으로 설정하면, Dockerfile이 `ARG TARGETPLATFORM` 변수(VI 섹션 참조)를 사용하더라도 빌드 동작이 항상 예측 가능하게 보장됩니다.
- 이는 나중에 M1/M2 Mac(ARM64) 환경에서 AMD64 이미지를 빌드해야 하는 경우 36처럼 개발 환경이 변경될 때 발생할 수 있는 실수를 방지해 줍니다.



### 빌드 결과물 처리: `--load`를 사용한 즉각적인 로컬 이미지 로드



네이티브 빌드의 주된 목적은 빌드 직후 로컬 환경에서 `docker run`을 통해 이미지를 테스트하는 것입니다.

이 시나리오에서는 `--load` 옵션(즉, `--output=type=docker` 9)이 가장 적합하고 효율적인 선택입니다. `docker-container` 드라이버는 빌드 결과(단일 플랫폼 이미지)를 로컬 Docker 데몬의 이미지 저장소로 성공적으로 전송할 수 있습니다.

빌드가 완료되면 이미지는 즉시 `docker images` 목록에 나타나며, `docker run` 명령으로 바로 사용할 수 있습니다.32

- **명령 예시:** (`mybuilder`가 활성화된 상태)

  Bash

  ```
  # 네이티브 빌드 후 즉시 로컬 데몬으로 로드
  $ docker buildx build --platform linux/amd64 \
      -t my-native-app:latest \
      --load.
  ```

이는 Jetson(시나리오 1)의 `--push` 중심 워크플로우와 명확히 대비되는, 네이티브 빌드(시나리오 2)의 핵심 워크플로우입니다.



## V. 통합 전략: 단일 명령어로 멀티 아키텍처 매니페스트 빌드



`docker buildx`의 가장 강력한 기능은 사용자의 두 시나리오(AMD64, ARM64)를 *단일 명령*으로 통합하여, 두 아키텍처를 모두 지원하는 단일 이미지 태그를 생성하는 것입니다.15



### 멀티 플랫폼 빌드의 정의



`--platform` 플래그에 쉼표(`,`)로 구분된 아키텍처 목록을 제공하여, `linux/amd64`와 `linux/arm64` 이미지를 동시에 빌드합니다.1

- **실행 (명령 예시):**

  Bash

  ```
  $ docker buildx build --platform linux/amd64,linux/arm64 \
      -t your-registry/multi-arch-app:latest \
      --push.
  ```



### 결과물: 매니페스트 리스트 (Manifest List)



이 명령을 실행할 때 `--load` 옵션은 사용할 수 없으며 14, 반드시 `--push` 옵션을 사용해야 합니다. 이 명령은 로컬에 이미지를 생성하지 않습니다. 대신, 레지스트리에서 다음과 같은 작업이 수행됩니다.

1. `buildx`는 Dockerfile을 각 플랫폼에 대해 한 번씩, 총 두 번 빌드합니다.13 `linux/amd64`용으로 한 번(네이티브), `linux/arm64`용으로 또 한 번(QEMU 에뮬레이션 사용).
2. 두 개의 플랫폼별 이미지가 빌드되어 레지스트리로 푸시됩니다 (내부적으로는 고유한 digest로 저장됩니다).
3. `buildx`는 사용자가 지정한 `your-registry/multi-arch-app:latest`라는 이름으로 **매니페스트 리스트(Manifest List)**라는 특수 객체를 생성하여 레지스트리에 푸시합니다.38
4. 이 매니페스트 리스트는 "이 태그는 `linux/amd64` 이미지를 가리키는 포인터와 `linux/arm64` 이미지를 가리키는 포인터를 포함한다"고 선언하는 일종의 '목차' 또는 '포인터' 역할을 합니다.

이후, 사용자의 AMD64 서버가 `docker pull your-registry/multi-arch-app:latest`를 실행하면, Docker 데몬은 매니페스트 리스트를 확인하고 자신의 아키텍처에 맞는 `linux/amd64` 이미지를 자동으로 다운로드합니다. 반면, Jetson 장치가 *동일한 태그*를 `pull`하면, `linux/arm64` 이미지를 자동으로 받게 됩니다.15



#### 테이블 2: `buildx build` 핵심 출력 옵션 비교



| **출력 옵션**             | **축약 대상 (--output)** | **지원 플랫폼**      | **결과물 위치**                       | **주요 사용 사례 (사용자 쿼리 기준)**                        |
| ------------------------- | ------------------------ | -------------------- | ------------------------------------- | ------------------------------------------------------------ |
| **`--load`**              | `type=docker` 9          | **단일 플랫폼만** 9  | 로컬 Docker 데몬 (`docker images`) 32 | **시나리오 2 (네이티브 AMD64 빌드)** 및 로컬 테스트          |
| **`--push`**              | `type=registry` 9        | **다중 플랫폼 지원** | 원격 레지스트리 8                     | **시나리오 1 (Jetson 크로스 컴파일)** 및 **통합 빌드 (V 섹션)** |
| **`--output type=local`** | 없음 39                  | 다중 플랫폼 지원 39  | 로컬 파일 시스템 디렉토리 32          | 이미지 대신 컴파일된 바이너리(아티팩트)만 추출할 때          |
| **`--output type=oci`**   | 없음 32                  | 다중 플랫폼 지원     | 로컬 OCI 이미지 레이아웃              | 타사 도구로 이미지를 스캔하거나 수동으로 전송할 때           |



## VI. 고급 Dockerfile 작성: 크로스 컴파일 최적화를 위한 아키텍처 인식



단순히 `buildx`를 사용하는 것을 넘어, `Dockerfile` 자체를 아키텍처를 인식하도록 작성하면 빌드 속도를 극적으로 향상시키고 이미지 크기를 최적화할 수 있습니다.



### 멀티 스테이지 빌드를 활용한 이미지 경량화



모든 프로덕션 Dockerfile의 표준 모범 사례는 멀티 스테이지 빌드를 사용하는 것입니다.5

- `AS build` (또는 `AS builder`) 스테이지에서 `golang`, `node`, `maven`과 같은 무거운 SDK 이미지와 모든 컴파일 도구, 개발용 종속성을 설치하고 애플리케이션을 컴파일합니다.
- `final` 스테이지는 `alpine`, `debian-slim` 또는 `scratch`와 같은 초경량 베이스 이미지에서 시작합니다.
- `COPY --from=build...` 명령을 사용하여 `build` 스테이지에서 생성된 최종 실행 파일이나 아티팩트만 `final` 스테이지로 복사합니다.41
- III 섹션에서 다룬 Jetson CUDA 빌드 35는 이 패턴을 필수적으로 사용해야 하는 대표적인 예입니다.



### BuildKit 자동 변수 활용: `TARGETPLATFORM`, `TARGETARCH`, `BUILDPLATFORM`



`docker buildx build` 명령을 실행할 때, BuildKit은 `--platform` 플래그 값을 기반으로 Dockerfile 내에서 `ARG`로 사용할 수 있는 여러 변수들을 자동으로 정의합니다.13

- **`TARGETPLATFORM`:** 빌드의 *최종* 타겟 플랫폼 (예: `linux/arm64`).43
- **`TARGETOS`**, **`TARGETARCH`**, **`TARGETVARIANT`:** `TARGETPLATFORM`을 세분화한 값 (예: `linux`, `arm64`, `v8`).43
- **`BUILDPLATFORM`:** 빌드 명령이 *실행되는* 호스트의 플랫폼 (사용자의 경우 항상 `linux/amd64`가 됨).13

*주의:* 이 변수들은 전역(global) 범위에 정의됩니다. Dockerfile의 `RUN` 명령어와 같은 스테이지 내부에서 이 변수들을 사용하려면, `FROM` 줄 다음에 `ARG TARGETPLATFORM` 또는 `ARG TARGETARCH`와 같이 다시 선언하여 로컬 범위로 가져와야 합니다.13



#### 테이블 3: BuildKit 자동 ARG 변수 (예시)



(실행 명령: `docker buildx build --platform linux/arm64/v8...` / 실행 호스트: AMD64)

| **변수명**           | **범위** | **값 (예시)**    | **설명**                                  |
| -------------------- | -------- | ---------------- | ----------------------------------------- |
| **`TARGETPLATFORM`** | Global   | `linux/arm64/v8` | `--platform` 플래그로 지정된 최종 타겟 43 |
| **`TARGETOS`**       | Global   | `linux`          | 타겟 OS 43                                |
| **`TARGETARCH`**     | Global   | `arm64`          | 타겟 아키텍처 43                          |
| **`TARGETVARIANT`**  | Global   | `v8`             | 타겟 아키텍처 변종 43                     |
| **`BUILDPLATFORM`**  | Global   | `linux/amd64`    | 빌드가 실행되는 호스트의 플랫폼 13        |



### 사례 연구: 아키텍처에 따라 다른 바이너리 다운로드



`Dockerfile` 내에서 `RUN uname -m`을 사용하여 아키텍처를 확인하려는 시도는 실패할 수 있습니다.44 QEMU 에뮬레이션 하에서는 `aarch64`가 반환될 수 있지만 15, 아래의 '빠른 빌드' 패턴에서는 호스트 아키텍처(`x86_64`)가 반환되어 로직이 깨집니다. `ARG TARGETPLATFORM` 변수가 아키텍처를 확인하는 유일하게 신뢰할 수 있는 방법입니다.

다음은 `TARGETPLATFORM` 변수와 `case` 문을 사용하여 타겟 아키텍처에 맞는 `tini` 바이너리를 조건부로 다운로드하는 예시입니다.44

Dockerfile

```
#...
ARG TINI_VERSION=v0.19.0
ARG TARGETPLATFORM
RUN case ${TARGETPLATFORM} in \
         "linux/amd64")  TINI_ARCH=amd64  ;; \
         "linux/arm64")  TINI_ARCH=arm64  ;; \
         "linux/arm/v7") TINI_ARCH=armhf  ;; \
         *) echo "Unsupported architecture: ${TARGETPLATFORM}"; exit 1 ;; \
    esac \
 && wget -q https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-static-${TINI_ARCH} -O /tini \
 && chmod +x /tini
#...
```



### 에뮬레이션(느림) vs. 크로스 컴파일(빠름): 네이티브 빌드 스테이지에서 타겟 바이너리 컴파일하기



문제점:

II 섹션에서 설정한 QEMU 에뮬레이션은 작동은 하지만 매우 느립니다.15 특히 apt-get install, npm install, 또는 go build와 같이 CPU 집약적인 컴파일 작업을 linux/arm64 베이스 이미지에서 실행할 때, QEMU의 오버헤드로 인해 빌드 시간이 네이티브 빌드 대비 수십 배까지 증가할 수 있습니다.

솔루션 (Native Cross-Compilation):

Go, Rust와 같은 최신 언어들은 AMD64 아키텍처에서 실행되는 컴파일러가 ARM64와 같은 다른 아키텍처용 바이너리를 생성(크로스 컴파일)하는 기능을 네이티브로 지원합니다.13 이 기능을 활용하면 QEMU 에뮬레이션을 완전히 회피할 수 있습니다.

"빠른 빌드" Dockerfile 패턴 (Go 예제) 13:

Dockerfile

```
# syntax=docker/dockerfile:1

# 1. 'build' 스테이지는 호스트(AMD64)의 네이티브 플랫폼($BUILDPLATFORM)에서 실행
#    이렇게 하면 'go build' 명령이 QEMU 없이 네이티브 속도로 실행됨
FROM --platform=$BUILDPLATFORM golang:1.19-alpine AS build
WORKDIR /src
COPY..

# 2. 타겟 아키텍처 정보를 ARG로 선언하여 로컬 범위로 가져옴
ARG TARGETOS
ARG TARGETARCH

# 3. 네이티브(AMD64) Go 컴파일러가 타겟(ARM64) 바이너리를 생성하도록
#    환경변수 GOOS, GOARCH를 설정
RUN --mount=type=cache,target=/root/.cache/go-build \
    GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/my-app.

# 4. 'final' 스테이지는 타겟 플랫폼(ARM64)의 경량 베이스 이미지를 사용
#    (이 스테이지는 QEMU 하에서 실행되지만, CPU 집약적 작업이 없음)
FROM alpine:latest
WORKDIR /app

# 5. 'build' 스테이지에서 크로스 컴파일된 바이너리만 복사
COPY --from=build /out/my-app.

ENTRYPOINT ["/my-app"]
```

이 Dockerfile을 `docker buildx build --platform linux/amd64,linux/arm64 --push`로 실행하면, BuildKit은 `build` 스테이지를 호스트(AMD64)에서 네이티브 속도로 실행합니다. `RUN go build` 단계는 `TARGETOS=linux`, `TARGETARCH=arm64` 환경 변수를 받아 QEMU 에뮬레이션 없이 ARM64용 바이너리를 *크로스 컴파일*합니다.13 `final` 스테이지(ARM64)는 QEMU 하에서 실행되지만, 단순 `COPY` 작업만 수행하므로 속도 저하가 거의 없습니다.

그 결과, Jetson(ARM64)용 이미지를 빌드하는 속도가 QEMU 에뮬레이션에만 의존할 때보다 극적으로 향상됩니다.



## VII. 핵심 문제 해결(Troubleshooting) 및 모범 사례





### 가장 일반적인 오류: `exec format error`



이 오류는 `docker buildx` 사용자가 가장 흔하게 마주치는 문제이며, 거의 항상 아키텍처 불일치를 의미합니다.

- 발생 원인 1: 실행 환경 불일치 (가장 흔함) 30

  docker buildx build --platform linux/arm64 --load 명령으로 linux/arm64 이미지를 빌드한 후 33, 이 이미지를 로컬 AMD64 호스트에서 docker run 하려고 시도할 때 발생합니다.30 호스트 CPU가 컨테이너 내부의 arm64 바이너리를 실행할 수 없기 때문입니다.28

  - **해결:** 이는 오류가 아니라 당연한 결과입니다. `linux/arm64` 이미지는 Jetson 장치에서 실행해야 합니다.

- 발생 원인 2: QEMU 설정 실패

  II 섹션의 QEMU 및 binfmt_misc 설정이 올바르게 완료되지 않았을 경우, docker buildx build... 명령의 RUN 단계(예: RUN apt-get update)에서 이 오류가 발생할 수 있습니다.18

  - **해결:** II 섹션의 `apt install qemu-user-static...` 및 `docker run --privileged... binfmt...` 명령을 다시 실행하고, `binfmt_misc` 핸들러가 `F` 플래그와 함께 올바르게 등록되었는지 확인합니다.18

- 발생 원인 3: 손상된 Shebang (스크립트 헤더) 30

  플랫폼과 무관하게, 실행하려는 스크립트 파일의 첫 줄(shebang)이 손상된 경우에도 이 오류가 발생할 수 있습니다.30

  - *오류 예시:* `#! /bin/sh` ( `#!` 뒤에 공백이 있음)
  - *올바른 예시:* `#!/bin/sh`
  - **해결:** 스크립트 파일의 shebang이 `#!`로 시작하고 올바른 인터프리터 경로를 가리키는지 확인합니다.



### 성능 문제: ARM64 빌드가 현저하게 느린 이유



- **원인:** QEMU를 통한 CPU 에뮬레이션은 본질적으로 매우 느립니다.15 `RUN` 명령어로 실행되는 모든 프로세스(패키지 설치, 컴파일, 압축 해제 등)가 이 에뮬레이션의 영향을 받아 네이티브 속도보다 10배에서 100배까지 느려질 수 있습니다.48

- **해결 1 (Dockerfile 최적화):** VI 섹션의 "빠른 빌드" 크로스 컴파일 패턴 13을 적용하여, CPU 집약적인 작업을 QEMU 에뮬레이션이 아닌 네이티브 `build` 스테이지로 옮깁니다.

- **해결 2 (네이티브 노드 추가):** 프로덕션 CI/CD 환경에서 QEMU를 완전히 배제하는 가장 좋은 방법입니다. `docker buildx create` 명령은 여러 빌드 노드를 단일 빌더에 추가하는 기능을 지원합니다.11

  1. AMD64 호스트(빌드 컨트롤러)에서 `docker buildx create --name multi-driver`로 빌더를 생성합니다.

  2. 네이티브 ARM64 시스템(예: 실제 Jetson 장치 또는 AWS Graviton 인스턴스)을 `ssh` 노드로 빌더에 추가합니다:

     Bash

     ```
     $ docker buildx create --name multi-driver --append --node arm-node \
         --platform linux/arm64 ssh://user@arm-host
     ```

  3. 이 `multi-driver` 빌더를 사용하여 `... --platform linux/amd64,linux/arm64...` 빌드를 실행하면, BuildKit이 `linux/amd64` 빌드는 로컬(AMD64)에서, `linux/arm64` 빌드는 `arm-node`(ARM64)로 작업을 자동 분배하여 *두 작업 모두 네이티브 속도*로 실행합니다.54



### 고급 출력: 이미지가 아닌 로컬 파일 시스템으로 아티팩트 내보내기



`--output type=local,dest=<path>` 옵션을 사용하면, 빌드된 이미지를 레지스트리나 로컬 데몬으로 보내는 대신, 이미지의 파일 시스템(또는 멀티 스테이지의 경우 최종 `COPY` 대상)을 호스트의 로컬 디렉토리로 직접 내보낼 수 있습니다.32

- **사용 사례:** Jetson에서 실행할 `my-app` 바이너리만 필요하고, 전체 Docker 이미지는 필요하지 않을 때 유용합니다.

- **명령 예시:** (VI 섹션의 "빠른 빌드" Dockerfile 사용 가정)

  Bash

  ```
  $ docker buildx build --platform linux/arm64 \
      --output type=local,dest=./jetson_binary.
  ```

- 이 명령은 AMD64 호스트에서 네이티브 속도로 `linux/arm64` 바이너리를 크로스 컴파일한 다음, 최종 이미지의 파일 시스템을 `./jetson_binary` 디렉토리에 저장합니다.39



## VIII. 전문가 권장 사항 및 요약



`docker buildx`는 최신 Docker 환경의 핵심이며, 특히 사용자가 요청한 Jetson(ARM64)과 네이티브(AMD64) 빌드 시나리오를 모두 효율적으로 처리하는 데 필수적입니다.



### 상황별 최적의 빌드 전략 요약



1. **초기 환경 설정 (최초 1회):**

   Bash

   ```
   # 1. QEMU 에뮬레이터 설치 (Ubuntu 22.04 기준)
   $ sudo apt-get update
   $ sudo apt-get install -y qemu-user-static binfmt-support
   
   # 2. binfmt_misc 핸들러 등록
   $ docker run --privileged --rm tonistiigi/binfmt --install all
   
   # 3. docker-container 드라이버 빌더 생성 및 활성화
   $ docker buildx create --name mybuilder --driver docker-container --use
   $ docker buildx inspect --bootstrap
   ```

   16

2. **시나리오 1 (Jetson/ARM64 빌드 후 레지스트리 배포):**

   Bash

   ```
   $ docker buildx build --platform linux/arm64 \
       -t your-registry/jetson-app:latest \
       --push.
   ```

3. **시나리오 2 (네이티브/AMD64 빌드 후 로컬 테스트):**

   Bash

   ```
   $ docker buildx build --platform linux/amd64 \
       -t my-native-app:latest \
       --load.
   ```

4. **통합 (두 플랫폼 동시 빌드 및 단일 태그로 배포):**

   Bash

   ```
   $ docker buildx build --platform linux/amd64,linux/arm64 \
       -t your-registry/multi-arch-app:latest \
       --push.
   ```

5. **고급 (Jetson용 컴파일된 바이너리만 추출):**

   Bash

   ```
   $ docker buildx build --platform linux/arm64 \
       --output type=local,dest=./jetson_artifacts.
   ```



### Dockerfile 황금률



`buildx`를 최대한 활용하기 위해 다음 Dockerfile 작성 원칙을 준수하는 것이 좋습니다.

1. **항상 멀티 스테이지 빌드를 사용하십시오:** `build` 스테이지와 `final` 스테이지를 분리하여 최종 이미지 크기를 최소화하고 보안을 강화해야 합니다.41
2. **`RUN uname -m`을 절대 신뢰하지 마십시오:** 아키텍처별 조건부 로직이 필요하다면, `RUN` 명령 대신 **항상 `ARG TARGETPLATFORM` 또는 `ARG TARGETARCH` 변수를 사용**해야 합니다.13
3. **QEMU 에뮬레이션을 최소화하십시오:** `linux/arm64` 빌드가 현저하게 느리다면 47, VI 섹션의 "빠른 빌드"(네이티브 크로스 컴파일) 패턴 13을 적용하여 Dockerfile을 리팩토링해야 합니다.
4. **Jetson CUDA 빌드는 특수하게 처리하십시오:** CUDA에 의존하는 Jetson 애플리케이션은 III 섹션에서 설명한 2단계(multi-stage) 빌드 패턴(빌드 스테이지에 CUDA dev 패키지 설치)을 필수적으로 적용해야 합니다.35

#### **Works cited**

1. How to Rapidly Build Multi-Architecture Images with Buildx | Docker, accessed November 16, 2025, https://www.docker.com/blog/how-to-rapidly-build-multi-architecture-images-with-buildx/
2. Understanding Docker Buildx & Builder: Modern Docker Builds with BuildKit - DEV Community, accessed November 16, 2025, https://dev.to/lovestaco/understanding-docker-buildx-builder-modern-docker-builds-with-buildkit-1hd
3. Docker buildx로 멀티 아키텍처 플랫폼 image 만들기(2) - 팔만코딩경, accessed November 16, 2025, https://80000coding.oopy.io/54dc871d-30c9-46cb-b609-2e8831541b5e
4. Docker Buildx Explained: Cross-Platform Builds Made Easy | by Mihir Popat - Medium, accessed November 16, 2025, https://mihirpopat.medium.com/docker-buildx-explained-cross-platform-builds-made-easy-c1b75e44ec6b
5. From Clean Dockerfiles to Cloud Build Offload: Unlocking Docker’s Modern Build Stack, accessed November 16, 2025, https://didourebai.medium.com/from-clean-dockerfiles-to-cloud-build-offload-unlocking-dockers-modern-build-stack-98fbb75365b8
6. Docker 멀티 플랫폼 빌드 - Docker Buildx - velog, accessed November 16, 2025, [https://velog.io/@dongs52/Docker-%EB%A9%80%ED%8B%B0-%ED%94%8C%EB%9E%AB%ED%8F%BC-%EB%B9%8C%EB%93%9C-Docker-Buildx](https://velog.io/@dongs52/Docker-멀티-플랫폼-빌드-Docker-Buildx)
7. Builders | Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/builders/
8. Registry cache - Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/cache/backends/registry/
9. docker buildx build | Docker Docs, accessed November 16, 2025, https://docs.docker.com/reference/cli/docker/buildx/build/
10. Manage builders | Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/builders/manage/
11. docker buildx create - Docker Docs, accessed November 16, 2025, https://docs.docker.com/reference/cli/docker/buildx/create/
12. Difference between docker buildx build and docker build for multi arch images, accessed November 16, 2025, https://stackoverflow.com/questions/78897082/difference-between-docker-buildx-build-and-docker-build-for-multi-arch-images
13. Faster Multi-Platform Builds: Dockerfile Cross-Compilation Guide ..., accessed November 16, 2025, https://www.docker.com/blog/faster-multi-platform-builds-dockerfile-cross-compilation-guide/
14. Usage | Docker Docs, accessed November 16, 2025, https://docs.docker.com/build-cloud/usage/
15. Multi-platform | Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/building/multi-platform/
16. Docker buildx로 멀티 플랫폼 이미지 빌드하기 - Taehun's Blog, accessed November 16, 2025, https://blog.taehun.dev/docker-buildx-
17. Building ARM/Jetson image on x86 linux machine using docker buildx | by pranjal Doshi, accessed November 16, 2025, https://medium.com/@pranjaldoshi96/building-arm-jetson-image-on-x86-linux-machine-using-docker-buildx-752293ce9c90
18. rust - docker buildx "exec user process caused: exec format error ..., accessed November 16, 2025, https://stackoverflow.com/questions/65365797/docker-buildx-exec-user-process-caused-exec-format-error
19. What does running the multiarch/qemu-user-static does before building a container?, accessed November 16, 2025, https://stackoverflow.com/questions/72444103/what-does-running-the-multiarch-qemu-user-static-does-before-building-a-containe
20. Build linux/arm64 docker image on linux/amd64 host - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/70757791/build-linux-arm64-docker-image-on-linux-amd64-host
21. Cannot build multi-platform images with docker buildx - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/60080264/docker-cannot-build-multi-platform-images-with-docker-buildx
22. Unable to use Action in Self Hosted Linux Containers · Issue #67 · docker/setup-qemu-action - GitHub, accessed November 16, 2025, https://github.com/docker/setup-qemu-action/issues/67
23. Building Multi-Arch Images for Arm and x86 with Docker Desktop, accessed November 16, 2025, https://www.docker.com/blog/multi-arch-images/
24. How to set Architecture for docker build to arm64? - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/47809904/how-to-set-architecture-for-docker-build-to-arm64
25. GitHub Action to build and push Docker images with Buildx, accessed November 16, 2025, https://github.com/docker/build-push-action
26. [Docker] Docker Buildx를 통한 Multi-architecture 이미지 빌드(x86, ARM) - 김징어의 Devlog, accessed November 16, 2025, https://kimjingo.tistory.com/115
27. How to build docker image for multiple platforms with cross-compile? - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/73978929/how-to-build-docker-image-for-multiple-platforms-with-cross-compile
28. Fixing 'exec format error' when Launching Docker Container in Kubernetes, accessed November 16, 2025, https://nodevops.nl/Fix-exec-format-error-Docker-Container-in-Kubernetes/
29. MacOS Ventura: buildx not solving "exec /bin/sh: exec format error" problems - Docker Hub, accessed November 16, 2025, https://forums.docker.com/t/macos-ventura-buildx-not-solving-exec-bin-sh-exec-format-error-problems/132444
30. Docker Build and Run Platforms and Resolving exec /usr/bin/sh ..., accessed November 16, 2025, https://www.baeldung.com/ops/docker-build-run-format-error
31. How to specify where to push using docker buildx build command - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/73933469/how-to-specify-where-to-push-using-docker-buildx-build-command
32. Exporters - Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/exporters/
33. Using buildx and buildkit and specifying --platform linux/arm64/v8 produces an image with no variant set · Issue #4039 - GitHub, accessed November 16, 2025, https://github.com/moby/buildkit/issues/4039
34. Building FROM scratch for cross compilation using buildx - Docker Community Forums, accessed November 16, 2025, https://forums.docker.com/t/building-from-scratch-for-cross-compilation-using-buildx/117134
35. Building docker image in CI with CUDA runnable on Jetson - Jetson ..., accessed November 16, 2025, https://forums.developer.nvidia.com/t/building-docker-image-in-ci-with-cuda-runnable-on-jetson/221643
36. What is --platform linux/amd64 doing on my Mac m1? : r/docker - Reddit, accessed November 16, 2025, https://www.reddit.com/r/docker/comments/1bvbq8b/what_is_platform_linuxamd64_doing_on_my_mac_m1/
37. Docker: Build multi platform container images with BuildX - YouTube, accessed November 16, 2025, https://www.youtube.com/watch?v=iWMvP9eKslg
38. docker buildx build 명령어를 사용하여 이미지 빌드 및 관리 방법 - LabEx, accessed November 16, 2025, https://labex.io/ko/tutorials/docker-how-to-use-docker-buildx-build-command-to-build-and-manage-images-555045
39. Local and tar exporters - Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/exporters/local-tar/
40. Building best practices - Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/building/best-practices/
41. Docker Build and Buildx best practices for optimized builds | Blog - Northflank, accessed November 16, 2025, https://northflank.com/blog/docker-build-and-buildx-best-practices-for-optimized-builds
42. Writing Effective Dockerfiles: Best Practices, accessed November 16, 2025, https://arnab-k.medium.com/writing-effective-dockerfiles-best-practices-6bb25d3a5fa8
43. Build variables - Docker Docs, accessed November 16, 2025, https://docs.docker.com/build/building/variables/
44. BretFisher/multi-platform-docker-build: Using BuildKit and ... - GitHub, accessed November 16, 2025, https://github.com/BretFisher/multi-platform-docker-build
45. Use Dockerfile ARG to determine FROM platform - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/71608637/use-dockerfile-arg-to-determine-from-platform
46. How can I get the Docker target platform inside the Build Environment / Dockerfile, accessed November 16, 2025, https://devops.stackexchange.com/questions/13446/how-can-i-get-the-docker-target-platform-inside-the-build-environment-dockerfi
47. Faster Docker builds for Arm without emulation - Depot.dev, accessed November 16, 2025, https://depot.dev/blog/docker-arm
48. Docker multi platform builds extremely slow for ARM64 on Gitlab CI - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/73938529/docker-multi-platform-builds-extremely-slow-for-arm64-on-gitlab-ci
49. buildx much slower than legacy on arm64 (Raspberry Pi) · Issue #2143 - GitHub, accessed November 16, 2025, https://github.com/docker/buildx/issues/2143
50. Docker Buildx taking so much time for arm64 - General, accessed November 16, 2025, https://forums.docker.com/t/docker-buildx-taking-so-much-time-for-arm64/128407
51. How to make my pipelines faster for arm64 architecture? : r/docker - Reddit, accessed November 16, 2025, https://www.reddit.com/r/docker/comments/1erp7t3/how_to_make_my_pipelines_faster_for_arm64/
52. accessed November 16, 2025, [https://nodevops.nl/Fix-exec-format-error-Docker-Container-in-Kubernetes/#:~:text=How%20to%20Fix%20the%20Error,both%20Mac%20and%20Linux%20architectures.](https://nodevops.nl/Fix-exec-format-error-Docker-Container-in-Kubernetes/#:~:text=How to Fix the Error,both Mac and Linux architectures.)
53. Docker : exec /usr/bin/sh: exec format error - GeeksforGeeks, accessed November 16, 2025, https://www.geeksforgeeks.org/devops/docker-exec-usr-bin-sh-exec-format-error/
54. Docker Buildx build: which node is used for which platform, accessed November 16, 2025, https://forums.docker.com/t/docker-buildx-build-which-node-is-used-for-which-platform/135525
55. Whati is the purpose of docker build output? - Stack Overflow, accessed November 16, 2025, https://stackoverflow.com/questions/59824331/whati-is-the-purpose-of-docker-build-output

