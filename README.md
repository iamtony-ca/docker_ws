# docker_ws

###
$USER 인 부분은 실제 username 으로 모두 변경해서 해야될수도 있음.(먼저 해보고 안되면,,)

## build
```

docker build \
  --build-arg USERNAME=$USER \
  --build-arg USER_UID=$(id -u) \
  --build-arg USER_GID=$(id -g) \
  --no-cache \
  -t ros2_jazzy_dev:latest /home/$USER/nav_ws/src/docker_ws/

```

## run for gpu
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
  -e XAUTHORITY=/home/$USER/.Xauthority \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/.Xauthority:/home/$USER/.Xauthority:ro \
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
  -e XAUTHORITY=/home/$USER/.Xauthority \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/.Xauthority:/home/$USER/.Xauthority:ro \
  -v /etc/localtime:/etc/localtime:ro \
  -v /home/$USER/nav_ws:/home/$USER/nav_ws \
  --name jazzy_gz_test1 \
  ros2_jazzy_dev:latest
```


## container 진입 후 cli for gpu
```
sudo apt update && sudo apt upgrade -y

sudo apt install -y mesa-vulkan-drivers vulkan-tools libgl1-mesa-dri mesa-utils libxkbcommon-x11-0

glxinfo | head
vulkaninfo | head

gz sim shapes.sdf

```

## docker run for cpu
```
xhost +local:docker

docker run -it --rm \
  --name jazzy_cpu_test1 \
  --privileged \
  --net=host \
  --ipc=host \
  --ulimit rtprio=99 \
  --ulimit memlock=-1 \
  -e DISPLAY=$DISPLAY \
  -e XAUTHORITY=/home/$USER/.Xauthority \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/.Xauthority:/home/$USER/.Xauthority:ro \
  -v /etc/localtime:/etc/localtime:ro \
  -v /home/$USER/nav_ws:/home/$USER/nav_ws \
  ros2_jazzy_dev_cpu:latest

```




## exec
```
xhost +local:docker
docker exec -it -u $USER jazzy_gz_sim /bin/bash
```
