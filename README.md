수요조사 설문 예시: https://forms.gle/Y6iVQB3p6yadck3R9

비밀번호 조사 필요
disk 용량 (컨테이너 용량, data disk 용량)
마운트할 데이터 저장소의 File System 결정 필요 (ext4, xfs, …)
컨테이너의 disk 용량 제한 방법 (조건 있음)
 docker run | Docker Documentation
기본값으로 현재 overlay2를 사용하는 듯한데, 우리의 backing fs (docker image가 설치되는 곳, host의 /)가 ext4인데 overlay2에서는 xfs에서만 container의 용량을 제한할 수 있는 것으로 보임. graph engine을 다른 것으로 변경할 필요가 있음. 예컨대 bacchus에서는 btrfs를 사용하는 것으로 앎.

마운트할 데이터 저장소는 미리 종류, 용량 설정하여 포멧 후 mount 하면 됨.
디스크 마운트 시 docker run | Docker Documentation

docker 이미지 및 container는 host의 ext4로 되어있는 / 경로에 (1GPU 당 약 250GB)
추가적으로 host의 /raid에 있는 건 포멧 후 mount 해서 제공하는 것으로 (1GPU 당 약 1.75TB)
도합 1GPU 당 약 2TB

file system 지원 목록 Docker storage drivers | Docker Documentation

DGX-A100
OS image: ssd 2개 -> raid 1 -> 1.8TB (안정성), ext4 -> 건들면 X
/raid mount: ssd 4개 -> raid 0 -> 14TB (data disk), ext4 -> btrfs, zfs, xfs with ftype=1

docker container size를 제한하려면
docker storage driver: devicemapper, btrfs, windowsfilter, zfs, overlay2(xfs) (default)
backing (host) fs: direct-lvm, btrfs, …, zfs, xfs with ftype=1 and ext4

docker container
container storage, data storage mounting

A100 메뉴얼 DGX-A100-User-Guide.book (nvidia.com)
SSD 4개 현재 묶여있는거 풀고, (btrfs로 format하고, 다시 raid0으로 묶고), docker 설치 경로 수정하고 (/var/lib/docker -> /raid/docker (실제 저장)), container 배포할 때 disk size 나눠주고
docker 저장소 위치 How do I change the default docker container location? - Stack Overflow

performance 비교 Filesystem performance of ext4, xfs, btrfs + raid 0 - Chia Plotting - Chia Forum

btrfs raid 구성 Using Btrfs with Multiple Devices - btrfs Wiki (kernel.org)

TODO: GPU, CPU, 메모리 affinity docker에서 설정 조사, htop 등 리소스 모니터링 시 내 할당 리소스만 보이게 하는 법
Docker Related
1. Install Docker (Skip if Docker is already installed)
curl https://get.docker.com | sh \
    && sudo systemctl start docker \
    && sudo systemctl enable docker 

2. Install NVIDIA Container Toolkit
sudo apt-get update
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
    && distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
    && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list \
    && sudo apt-get update
sudo apt-get install -y nvidia-docker2 \
    && sudo systemctl restart docker

NVIDIA Container Toolkit의 설치 방식이 최근에 업데이트 되었습니다.
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-docker2
sudo systemctl restart docker

3. (Optional) Check for a working setup
sudo docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi


MIG Related (run as sudo -s)
1. Enable MIG
For all GPUs
nvidia-smi -mig 1

For GPUID=0
nvidia-smi -i 0 -mig 1

Check for creatable GPU Instance profile for GPUID=0
nvidia-smi mig -i 0 -lgip

2. Create GPU Instance for GPUID=0, ProfileID=9,14,19
nvidia-smi mig -i 0 -cgi 9,14,19

Check Instance IDs for pre-created GPU Instances for GPUID=0
nvidia-smi mig -i 0 -lgi

3. Create Compute Instance

Check for creatable Compute Instance profile for GPUID=0, GPUInstanceID=1
nvidia-smi mig -i 0 -gi 1 -lcip

Create Compute Instance profile for GPUID=0, GPUInstanceID=1, ComputeProfileID=1
nvidia-smi mig -i 0 -gi 1 -cci 1

Check for pre-created Compute Instances for GPUID=0, GPUInstanceID=1
nvidia-smi mig -i 0 -gi 1 -lci

Check for MIG Device using:
nvidia-smi

4. Destroy Compute Instance
For specific GPUID=0, GPUInstanceID=1, ComputeInstanceID=1
nvidia-smi mig -i 0 -gi 1 -ci 1 -dci

For specific GPUID=0, GPUInstanceID=1
nvidia-smi mig -i 0 -gi 1 -dci

5. Destroy GPU Instance
For specific GPUID=0, GPUInstanceID=1
nvidia-smi mig -i 0 -gi 1 -dgi

For specific GPUID=0
nvidia-smi mig -i 0 -dgi

6. Disable MIG
For all GPUs:
nvidia-smi -mig 0

For GPUID=0
nvidia-smi -i 0 -mig 0

Check if destroyed using:
nvidia-smi

Mixing them up
1. Run nvidia-smi from within a CUDA container (change MIG UUID to proper one)
To find out UUID:
nvidia-smi -L

sudo docker run --runtime=nvidia \
    -e NVIDIA_VISIBLE_DEVICES=MIG-718fa054-583b-5710-840e-e29cb9e18842 \
    nvidia/cuda:11.0-base nvidia-smi

NVIDIA-Docker Related
1. Download nvidia/cuda image from Docker Hub
nvidia-docker pull nvidia/cuda:11.0-base

2. Run a container using given image (specify limits)
docker run --gpus '"device=0,1"' --cpus="32" -m 128g --name user1 -dit nvidia/cuda:11.0-base /bin/bash

관리를 용이하게 하기 위해 컨테이너 이름을 지정하는 건가요? (--name user1) 네 그렇습니다.

3. Execute existing container by its name
docker exec -it user1 /bin/bash

4. Users can check their CPU / Memory resources by..
CPUs:
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
(then divide it by 10,000)
Memory:
apt-get update
apt-get install -y cgroup-tools
cgget -n --values-only --variable memory.limit_in_bytes /
(then divide it by 1024*1024*1024 to see it in GiB)
htop: 남이 돌리는 job을 볼 수 있는 문제

5. Send file(s) from host to container (or from container to host) using container ID
docker cp foo.txt container_id:/foo.txt
docker cp container_id:/foo.txt foo.txt
docker cp src/. container_id:/target
docker cp container_id:/src/. target

TODO:
Disk capacity 제한


sudo 권한 필요한 설치 파일들 설치 요청 시 손쉽게 할 수 있는 방법 (Dockerfile 등 이용)
Default(Ubuntu 20.04)가 아닌 OS 설치

지원 가능한 이미지 목록
nvidia/cuda Tags | Docker Hub

지원되는 OS 배포판: ubuntu20.04, ubuntu18.04, ubi8, ubi7, centos8, centos7
원하는 cuda 버전 (11.6.2, 11.5, etc), 원하는 옵션 (base, runtime, devel) 등 지원하는 Tags에서 검색 (설문에 포함)
위의 명령어에서 nvidia/cuda:11.0-base를 해당 image로 교체하면 됨
nvidia가 아닌 다른 image를 사용해도 괜찮지만, 이 경우에는 cuda가 설치되어있지 않아 있을 수 있음.
–gpus flag가 없으면 아예 gpu가 포함되지 않음.
설치되어있는 cuda version (/usr/local/cuda)와 nvidia-smi에서 나오는 cuda version이 서로 다를 수 있음. 전자는 container의, 후자는 host의 cuda version이 출력됨.


How to create a container using Dockerfile to enable SSH via port number
https://stackoverflow.com/questions/72159795/how-to-access-to-a-docker-container-via-ssh-using-ip-address/72162580#72162580

1. Create a Dockerfile whose content is as following:
# Instruction for Dockerfile to create a new image on top of the base image (nvidia/cuda:11.0-base) FROM nvidia/cuda:11.0-base ARG root_password RUN apt-get update || echo "OK" && apt-get install -y openssh-server RUN mkdir /var/run/sshd RUN echo "root:${root_password}" | chpasswd RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config EXPOSE 22 CMD ["/usr/sbin/sshd", "-D"]

2. (Optional) Create appropriate .dockerignore file

3. Build image from Dockerfile
docker image build --build-arg root_password=password --tag nvidia/cuda:11.0-base-ssh .

root의 password를 설정하는 순간 그 image는 특정 사용자에 종속되게 됨. 따라서 ssh server를 설치하는 것까지만, image로 빌드하는 건 안하는 게 좀 더 바람직해보임.
사용자가 자신의 container에 접속한 뒤에 passwd로 비번을 바꾸면 괜찮긴 한 것 같아요. 빌드된 image가 있으면 관리에 좀 더 용이할 것 같긴해서요.
default passwd: 1234

4. Create container
docker container run --gpus '"device=0,1"' --cpus="32" -m 128g -d -P --name msh nvidia/cuda:11.0-base-ssh
Then, run `docker ps` to see the container port.

소문자로 해서 port를 수동으로 정해줄 수도 있음.

5. Access the container using ssh via port number, e.g.,
ssh -p 49157 root@147.46.92.213
