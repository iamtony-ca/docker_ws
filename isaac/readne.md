
# 공식 docs 원본 디렉토리

mkdir -p ~/docker/isaac-sim-601/cache/main
mkdir -p ~/docker/isaac-sim-601/cache/computecache
mkdir -p ~/docker/isaac-sim-601/config
mkdir -p ~/docker/isaac-sim-601/data
mkdir -p ~/docker/isaac-sim-601/logs
mkdir -p ~/docker/isaac-sim-601/pkg
mkdir -p ~/.cache/ov/hub

# 공유 폴더 추가

mkdir -p ~/docker/isaac-sim-601/volume

sudo chown -R 1234:1234 ~/docker/isaac-sim-601 ~/.cache/ov/hub

# 공유 폴더: host(tony)와 container(1234) 양방향 편집 가능하게 ACL

MYUID=$(id -u)   # host user = 1000
sudo setfacl -R    -m u:$MYUID:rwx -m u🔢rwx -m m:rwx ~/docker/isaac-sim-601/volume
sudo setfacl -R -d -m u:$MYUID:rwx -m u🔢rwx -m m:rwx ~/docker/isaac-sim-601/volume



cd ~/docker/isaac-sim-build
docker build -t isaac-sim-cc:6.0.1 .
docker build --network=host --no-cache -t isaac-sim-cc:6.0.1 .





xhost +local:
docker run --name isaac-sim-601 --entrypoint bash -it --gpus all -e "ACCEPT_EULA=Y" --network=host \
    -e "PRIVACY_CONSENT=Y" \
    -v $HOME/.Xauthority:/isaac-sim/.Xauthority \
    -e DISPLAY \
    -v ~/docker/isaac-sim-601/cache/main:/isaac-sim/.cache:rw \
    -v ~/docker/isaac-sim-601/cache/computecache:/isaac-sim/.nv/ComputeCache:rw \
    -v ~/docker/isaac-sim-601/logs:/isaac-sim/.nvidia-omniverse/logs:rw \
    -v ~/docker/isaac-sim-601/config:/isaac-sim/.nvidia-omniverse/config:rw \
    -v ~/docker/isaac-sim-601/data:/isaac-sim/.local/share/ov/data:rw \
    -v ~/docker/isaac-sim-601/pkg:/isaac-sim/.local/share/ov/pkg:rw \
    -v ~/docker/isaac-sim-601/volume:/isaac-sim/volume:rw \
    -v ~/.cache/ov/hub:/var/cache/hub:rw \
    -u 1234:1234 \
    isaac-sim-cc:6.0.1 
