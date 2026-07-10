# ===================================================================
# Isaac Sim 6.0.1 (Ubuntu 24.04/Noble) 베이스 + sudo 유저 + Claude Code
#   ROS 2 Jazzy 는 컨테이너 내에서 직접 설치 예정
#   [변경] Claude Code 설치를 런타임 HOME(/isaac-sim)으로 통일 →
#          /opt/claude 방식 및 /usr/local/bin 심링크 제거, HOME 불일치 해소
# ===================================================================
FROM nvcr.io/nvidia/isaac-sim:6.0.1

USER root
ARG DEBIAN_FRONTEND=noninteractive
# 이미지 내장 비root 유저 (uid 1234, HOME=/isaac-sim)
ARG USERNAME=isaac-sim

# 1) sudo, 그래픽 유틸, locale
RUN apt-get update && apt-get install -y --no-install-recommends \
      sudo curl gnupg2 lsb-release locales \
      mesa-vulkan-drivers vulkan-tools libgl1-mesa-dri mesa-utils libxkbcommon-x11-0 git \
 && locale-gen en_US en_US.UTF-8 \
 && update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8 \
 && rm -rf /var/lib/apt/lists/*
ENV LANG=en_US.UTF-8

# 2) sudo NOPASSWD + 하드웨어/렌더 그룹
RUN echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
 && chmod 0440 /etc/sudoers.d/$USERNAME \
 && usermod -aG dialout,plugdev,video $USERNAME \
 && (getent group render || groupadd render) \
 && usermod -aG render $USERNAME

# [변경] 3) /opt/claude 설치 블록 + /usr/local/bin/claude 심링크 → 전부 삭제.
#          (빌드 HOME=/opt vs 런타임 HOME=/isaac-sim 불일치가 자기점검 경고의 근원)

# ---- 비root 유저로 전환 ----
USER $USERNAME
WORKDIR /isaac-sim

# [변경] 4) 베이스 이미지가 root 소유로 만들어둔 ~/.local 을 유저 소유로 교정.
#          (없으면 생성) 이 선점이 없으면 claude 설치가 EACCES 로 실패함.
RUN sudo mkdir -p /isaac-sim/.local/share \
 && sudo chown -R "$USERNAME:$USERNAME" /isaac-sim/.local

# [변경] 5) Claude Code -> 런타임 HOME(/isaac-sim/.local)에 설치. sudo 절대 금지.
#          유저 소유 = 바이너리 경로·설정 경로가 런타임과 일치 → doctor 통과.
#          간헐 SYN 드롭 대비 재시도 + 실패 시 빌드 중단(set -e).
#          버전 고정이 필요하면: bash /tmp/cc-install.sh <version>  (형식은 --help 로 확인)
RUN set -eux; \
    curl -4 --retry 10 --retry-all-errors --retry-delay 3 --connect-timeout 5 \
         -fsSL https://claude.ai/install.sh -o /tmp/cc-install.sh; \
    bash /tmp/cc-install.sh; \
    test -x "$HOME/.local/bin/claude"; \
    rm -f /tmp/cc-install.sh

# [변경] 6) 실제 설치 경로를 PATH에 노출 (doctor PATH 경고 해소, 심링크 불필요).
ENV PATH="/isaac-sim/.local/bin:${PATH}"

# [변경] 7) 컨테이너는 이미지 재빌드로 버전 관리 → 백그라운드 자동 업데이트 비활성화.
#          (임시 레이어에 write → 인스턴스별 드리프트 방지, 재현성 확보)
ENV DISABLE_AUTOUPDATER=1

# ROS 2 자동 소싱 (나중에 /opt/ros/jazzy 설치 시 유효, 없으면 조용히 무시)
RUN echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc

# (환경 특화) 사내망 SSL 우회 — 보안 완화라 꼭 필요할 때만 유지
#   주의: 아래는 이 유저의 '모든' HTTPS 검증을 끔(claude→api.anthropic.com 포함).
#         가능하면 사내 미러 호스트로만 스코프하는 걸 권장(git insteadOf / per-repo).
# RUN echo "check_certificate = off" >> ~/.wgetrc \
#  && echo "insecure" >> ~/.curlrc \
#  && git config --global http.sslVerify false
RUN git config --global http.sslVerify false

CMD ["/bin/bash"]
