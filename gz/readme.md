j

## 결론부터

Isaac Sim 때 ACL이 필요했던 이유는 **컨테이너 UID가 1234로 고정**되어 있어서 호스트(1000)와 어긋났기 때문입니다. `osrf/ros:jazzy-desktop-full`은 직접 Dockerfile을 얹을 수 있으니 **컨테이너 유저 UID/GID를 호스트와 동일하게 맞추는 게 정답**입니다. 그러면 `chown`도 `setfacl`도 필요 없고, `volume` 폴더 하나만 bind mount 하면 양방향 편집이 그냥 됩니다.

한 가지 함정: **Ubuntu 24.04(noble) 베이스 이미지에는 이미 `ubuntu` 유저가 uid/gid 1000으로 존재**합니다. `osrf/ros:jazzy-*`가 noble 기반이라 그대로 상속하므로, 호스트 UID가 1000이면 `useradd -u 1000`이 충돌로 실패합니다. 아래 Dockerfile에서 이걸 처리합니다.

---

## 1. 호스트 디렉토리 구조

```bash
mkdir -p ~/ws/volume/nav2_ws/src
mkdir -p ~/ws/volume/manip_ws/src
mkdir -p ~/ws/volume/bt_ws/src
# 필요한 만큼
```

```
~/ws/
└── volume/                 <- 여기부터 컨테이너와 공유 (mount point 1개)
    ├── nav2_ws/src/
    ├── manip_ws/src/
    └── bt_ws/src/
```

bind mount는 절대경로만 받습니다. 호스트 경로가 없으면 Docker가 **root 소유로** 만들어버리니 `docker run` 전에 반드시 먼저 `mkdir` 하세요. ([Docker bind mounts 공식 문서](https://docs.docker.com/engine/storage/bind-mounts/))

---

## 2. Dockerfile

```dockerfile
# ===================================================================
# ROS 2 Jazzy dev container (osrf/ros:jazzy-desktop-full 기반)
#   - 호스트 UID/GID와 동일한 유저 생성 -> volume 양방향 편집 시 ACL 불필요
#   - Ubuntu 24.04 기본 'ubuntu'(1000:1000) 유저 충돌 처리 포함
# ===================================================================
FROM osrf/ros:jazzy-desktop-full

ARG DEBIAN_FRONTEND=noninteractive
ARG USERNAME=tony
ARG UID=1000
ARG GID=1000

USER root

# 1) 개발 도구 (colcon/rosdep/vcstool은 ros-base 단계에 이미 포함)
RUN apt-get update && apt-get install -y --no-install-recommends \
      sudo git vim nano less tree curl gnupg2 lsb-release locales \
      build-essential cmake gdb clangd \
      python3-pip python3-argcomplete \
      ros-jazzy-rmw-cyclonedds-cpp \
      mesa-utils libxkbcommon-x11-0 \
 && locale-gen en_US en_US.UTF-8 \
 && update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8 \
 && rm -rf /var/lib/apt/lists/*
ENV LANG=en_US.UTF-8

# 2) UID/GID 충돌 정리 후 유저 생성 (핵심)
#    noble 베이스는 'ubuntu' 유저를 1000:1000으로 미리 갖고 있음
RUN set -eux; \
    if getent passwd ${UID} >/dev/null; then \
        old="$(getent passwd ${UID} | cut -d: -f1)"; \
        if [ "$old" != "${USERNAME}" ]; then userdel -r "$old" || true; fi; \
    fi; \
    if ! getent group ${GID} >/dev/null; then groupadd -g ${GID} ${USERNAME}; fi; \
    if ! getent passwd ${USERNAME} >/dev/null; then \
        useradd -m -u ${UID} -g ${GID} -s /bin/bash ${USERNAME}; \
    fi; \
    echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME}; \
    chmod 0440 /etc/sudoers.d/${USERNAME}; \
    usermod -aG dialout,plugdev,video ${USERNAME}; \
    (getent group render || groupadd render); \
    usermod -aG render ${USERNAME}

# 3) rosdep 초기화 (베이스에서 이미 init 되어 있으면 무시)
RUN rosdep init 2>/dev/null || true

USER ${USERNAME}
WORKDIR /home/${USERNAME}

RUN rosdep update || true

# 4) 셸 환경: ROS 소싱 + volume 아래 워크스페이스 자동 소싱 헬퍼
RUN echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc \
 && echo 'source /usr/share/colcon_argcomplete/hook/colcon-argcomplete.bash' >> ~/.bashrc \
 && echo 'export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp' >> ~/.bashrc \
 && echo 'sws() { source ~/volume/$1/install/setup.bash; }' >> ~/.bashrc

# (환경 특화) 사내망 SSL 우회 — 정말 필요할 때만
RUN git config --global http.sslVerify false

# 마운트 포인트를 미리 유저 소유로 만들어 둠
RUN mkdir -p /home/${USERNAME}/volume

ENTRYPOINT ["/ros_entrypoint.sh"]
CMD ["bash"]
```

빌드:

```bash
cd ~/docker/ros2-jazzy-build
docker build --network=host \
  --build-arg USERNAME=$(id -un) \
  --build-arg UID=$(id -u) \
  --build-arg GID=$(id -g) \
  -t ros2-jazzy-dev:latest .
```

---

## 3. 실행

```bash
xhost +local:

docker run -it --name ros2-jazzy \
  --gpus all \
  --network=host \
  --ipc=host \
  --privileged \
  -e DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -e NVIDIA_DRIVER_CAPABILITIES=graphics,utility,compute \
  -e ROS_DOMAIN_ID=0 \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/.Xauthority:/home/$(id -un)/.Xauthority:rw \
  -v $HOME/jazzy_ws/volume:/home/$(id -un)/volume:rw \
  ros2-jazzy-dev:latest
```

`-u`를 따로 줄 필요 없습니다 (Dockerfile의 `USER`가 이미 적용됨). Isaac Sim 이미지처럼 UID가 고정된 경우에만 `-u`가 의미 있었던 겁니다.


##
##
##
##

컨테이너 안에서:

```bash
cd ~/volume/nav2_ws
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=RelWithDebInfo
sws nav2_ws
```

호스트에서 VS Code로 `~/ws/volume/nav2_ws/src`를 열어 편집하면 컨테이너에 즉시 반영되고, 컨테이너가 만든 `build/ install/ log/`도 호스트에서 본인 소유로 보입니다.

---

## 4. UID를 못 맞추는 경우에만 ACL

호스트 UID != 컨테이너 UID인 상황(예: 베이스 이미지의 `ubuntu` 1000을 그대로 쓰는데 호스트가 1001)이라면 Isaac Sim 때와 같은 방식이 필요합니다:

```bash
CUID=1000   # 컨테이너 유저 UID
sudo setfacl -R    -m u:$(id -u):rwx -m u:${CUID}:rwx -m m:rwx ~/ws/volume
sudo setfacl -R -d -m u:$(id -u):rwx -m u:${CUID}:rwx -m m:rwx ~/ws/volume
```

`-d`(default ACL)가 핵심입니다. 이게 있어야 컨테이너가 새로 만드는 `build/`, `install/` 하위 파일에도 규칙이 상속됩니다. 다만 ACL은 파일시스템이 지원해야 하고(ext4는 기본 지원, 일부 마운트 옵션/NFS는 주의), git이 퍼미션 변화를 감지하는 등 잔가지가 생기니 **가능하면 UID 매칭 쪽을 쓰는 걸 권합니다.**

---

## 주의할 점 몇 가지

**빌드 산출물 혼용 금지** — 같은 `nav2_ws`를 호스트에서도 `colcon build` 하면 `build/install/`이 서로 다른 ABI/경로로 덮어써져서 깨집니다. 빌드는 컨테이너 안에서만 하거나, 호스트 빌드가 필요하면 `--build-base build_host --install-base install_host`로 분리하세요.

**`.gitignore`** — `volume/*/` 각 워크스페이스에 `build/ install/ log/`를 넣어두세요. 컨테이너가 만든 산출물이 호스트 git에 그대로 보입니다.

**`--privileged`** — USB 장치(RealSense, ZED, UR16e 이더넷은 무관) 접근 때문에 넣었는데, 보안상 줄이려면 `--device=/dev/bus/usb` + `-v /dev:/dev` 정도로 좁히는 게 낫습니다.

**컨테이너 재접속** — `docker start -i ros2-jazzy` 또는 `docker exec -it ros2-jazzy bash`. 새 셸마다 `xhost +local:`이 풀렸으면 다시 실행해야 GUI가 뜹니다.

**DDS 멀티 컨테이너** — `--network=host --ipc=host` 조합이면 호스트/다른 컨테이너의 ROS 2 노드와 그냥 통신됩니다. 섞이는 게 싫으면 `ROS_DOMAIN_ID`를 프로젝트별로 다르게 주세요.

참고: [osrf/docker_images 저장소](https://github.com/osrf/docker_images) (해당 이미지의 실제 Dockerfile 확인용), [Docker bind mounts](https://docs.docker.com/engine/storage/bind-mounts/).
