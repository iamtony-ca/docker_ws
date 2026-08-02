# 1) 새 공유 폴더 생성
mkdir -p ~/blog_ws/volume

# 2) host(tony) ↔ container(1234) 양방향 편집용 ACL
MYUID=$(id -u)
sudo setfacl -R    -m u:$MYUID:rwx -m u:1234:rwx -m m:rwx ~/blog_ws/volume
sudo setfacl -R -d -m u:$MYUID:rwx -m u:1234:rwx -m m:rwx ~/blog_ws/volume

# 3) 컨테이너 생성 + 진입
docker run --name isaacism-blog --entrypoint bash -it --gpus all -e "ACCEPT_EULA=Y" --network=host \
    -e "PRIVACY_CONSENT=Y" \
    -v $HOME/.Xauthority:/isaac-sim/.Xauthority \
    -e DISPLAY \
    -v ~/blog_ws/volume:/isaac-sim/volume:rw \
    -u 1234:1234 \
    isaac-sim-claude:6.0.1 
