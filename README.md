# docker_ws

###
$USERNAME 인 부분은 실제 username 으로 모두 변경해서 해야될수도 있음.

## build
```

docker build \
  --build-arg USERNAME=$USER \
  --build-arg USER_UID=$(id -u) \
  --build-arg USER_GID=$(id -g) \
  --no-cache \
  -t ros2_jazzy_dev:latest /home/$USERNAME/nav_ws/src/docker_ws/

```

## run
```
xhost +local:docker

docker run -it \
  --gpus all \
  --privileged \
  --net=host \
  --ipc=host \
  --ulimit rtprio=99 \
  --ulimit memlock=-1 \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=graphics,utility,compute,display \
  -e DISPLAY=$DISPLAY \
  -e XAUTHORITY=/home/$USERNAME/.Xauthority \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/.Xauthority:/home/$USERNAME/.Xauthority:ro \
  -v /etc/localtime:/etc/localtime:ro \
  --name jazzy_gz_sim \
  ros2_jazzy_dev:latest


docker run -it --rm \
  --gpus all \
  --privileged \
  --net=host \
  --ipc=host \
  --ulimit rtprio=99 \
  --ulimit memlock=-1 \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=graphics,utility,compute,display \
  -e DISPLAY=$DISPLAY \
  -e XAUTHORITY=/home/$USERNAME/.Xauthority \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/.Xauthority:/home/$USERNAME/.Xauthority:ro \
  -v /etc/localtime:/etc/localtime:ro \
  -v /home/$USERNAME/nav_ws:/home/$USERNAME/nav_ws \
  --name jazzy_gz_test1 \
  ros2_jazzy_dev:latest
```


## container 진입 후 cli
```
sudo apt update && sudo apt upgrade -y

sudo apt install -y mesa-vulkan-drivers vulkan-tools libgl1-mesa-dri mesa-utils libxkbcommon-x11-0

glxinfo | head
vulkaninfo | head

gz sim shapes.sdf

```

## exec
```
xhost +local:docker
docker exec -it -u $USERNAME jazzy_gz_sim /bin/bash
```