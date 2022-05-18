# NVIDIA Docker Container 기반 DGX-A100 자원 관리 매뉴얼

* 매뉴얼 작성자: 컴퓨터공학부 이재욱 교수님 연구실 석사 과정 민선홍(sunhongmin@snu.ac.kr), 인턴 백우현(baneling100@snu.ac.kr)
* 마지막 수정일: 2022년 5월 18일

## 기본 가정
* 본 매뉴얼은 컴퓨터공학부의 DGX-A100 머신을 기준으로 작성되었습니다.
* 해당 머신은 x86_64 아키텍처를 사용하고 있으며 그 위에 Ubuntu 20.04.4 LTS 운영체제가 설치되어 있습니다.
* 해당 머신에는 GPU 사용을 위한 NVIDIA Cuda Toolkit이 설치되어 있습니다. 아래와 같이 `nvidia-smi` 명령어를 통해 NVIDIA Cuda Toolkit 설치 여부를 확인할 수 있습니다. 만약 설치가 되어있지 않으면 [NVIDIA Cuda Toolkit 11.6 Update 2 Downloads](https://developer.nvidia.com/cuda-11-6-2-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=20.04&target_type=runfile_local)를 통해 설치해주시기 바랍니다.
* CUDA는 크게 driver 단에서의 버전과 toolkit의 버전으로 나뉩니다. 전자는 `nvidia-smi`를 실행해서 확인할 수 있고, 후자는 `nvcc --version`으로 확인할 수 있습니다. Toolkit의 버전이 driver의 버전보다 낮거나 같을 때에는 문제가 없지만, 더 큰 경우에는 호환이 되지 않을 수 있습니다. 현재 CUDA toolkit이 11.7까지 나왔습니다만, driver 버전과의 호환을 위해 최신으로 업데이트하는 건 지양하길 바랍니다.

```console
nvadmin@dgx-a100:~$ nvidia-smi
Mon May 16 11:16:19 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 510.47.03    Driver Version: 510.47.03    CUDA Version: 11.6     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100-SXM...  On   | 00000000:07:00.0 Off |                    0 |
| N/A   28C    P0    54W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
|   1  NVIDIA A100-SXM...  On   | 00000000:0F:00.0 Off |                    0 |
| N/A   28C    P0    52W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
...
+-------------------------------+----------------------+----------------------+
|   7  NVIDIA A100-SXM...  On   | 00000000:BD:00.0 Off |                    0 |
| N/A   30C    P0    58W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## 사전 작업

### I. Docker Engine 설치

* 아래 설명을 따라하시거나 [Official Docker Engine Installation Manual](https://docs.docker.com/engine/install/ubuntu/)을 참고해주세요.

0. 아래 명령어를 통해 Docker(>= 19.03)가 이미 설치되어 있는 것을 확인하셨다면 **5. Docker Engine 설치 확인하기**로 넘어가주세요.

```console
nvadmin@dgx-a100:~$ docker version --format '{{.Server.Version}}'
20.10.15
nvadmin@dgx-a100:~$ docker version --format '{{.Client.Version}}'
20.10.15
```

1. Repository 설정하기

```console
$ sudo apt-get update
$ sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

2. Docker의 official GPG key 추가하기

```console
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

3. Stable version repository 설정하기

```console
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Docker Engine 설치하기

```console
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

5. Docker Engine 설치 확인하기

```console
nvadmin@dgx-a100:~$ sudo docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

### II. NVIDIA Container Toolkit 설치

* 아래 설명을 따라하시거나 [Official NVIDIA Container Toolkit Installation Manual](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)을 참고해주세요.

0. 아래 명령어를 통해 NVIDIA Docker(>= 2.8.0)가 이미 설치되어 있는 것을 확인하셨다면 **5. Base CUDA Container 실행을 통한 설치 확인하기**로 넘어가주세요.

```console
nvadmin@dgx-a100:~$ nvidia-docker version | grep "NVIDIA Docker"
NVIDIA Docker: 2.10.0
```

1. Docker 설정하기

```console
$ curl https://get.docker.com | sh && sudo systemctl --now enable docker
```

`Warning: the "docker" command appears to already exist on this system.`이 나오면 Ctrl+C를 눌러 해당 실행을 멈춥니다.

```console
nvadmin@dgx-a100:~$ curl https://get.docker.com | sh && sudo systemctl --now enable docker
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 19305  100 19305    0     0   204k      0 --:--:-- --:--:-- --:--:--  204k
# Executing docker install script, commit: 614d05e0e669a0577500d055677bb6f71e822356
Warning: the "docker" command appears to already exist on this system.

If you already have Docker installed, this script can cause trouble, which is
why we're displaying this warning and provide the opportunity to cancel the
installation.

If you installed the current Docker package using this script and are using it
again to update Docker, you can safely ignore this message.

You may press Ctrl+C now to abort this script.
+ sleep 20
^C
```

2. 패키지 repository와 GPG key 설정하기

```console
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

3. NVIDIA Docker 설치하기

```console
$ sudo apt-get update
$ sudo apt-get install -y nvidia-docker2
```

4. Docker daemon 재시작하기

```console
$ sudo systemctl restart docker
```

5. Base CUDA Container 실행을 통한 설치 확인하기

```console
nvadmin@dgx-a100:~$ sudo docker run --rm --gpus all nvidia/cuda:11.0.3-base-ubuntu20.04 nvidia-smi
Unable to find image 'nvidia/cuda:11.0.3-base-ubuntu20.04' locally
11.0.3-base-ubuntu20.04: Pulling from nvidia/cuda
d5fd17ec1767: Already exists
136ed510ad77: Pull complete
c7f8ead1f79b: Pull complete
be868dff62ae: Pull complete
61470f7d15cf: Pull complete
Digest: sha256:96422d2730adcb0cfcb260d2b1e96186ba5f05cb295b265631d98ea03d423a17
Status: Downloaded newer image for nvidia/cuda:11.0.3-base-ubuntu20.04
Mon May 16 03:18:49 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 510.47.03    Driver Version: 510.47.03    CUDA Version: 11.6     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100-SXM...  On   | 00000000:07:00.0 Off |                    0 |
| N/A   34C    P0    55W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
|   1  NVIDIA A100-SXM...  On   | 00000000:0F:00.0 Off |                    0 |
| N/A   34C    P0    53W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
...
+-------------------------------+----------------------+----------------------+
|   7  NVIDIA A100-SXM...  On   | 00000000:BD:00.0 Off |                    0 |
| N/A   36C    P0    61W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

### III. DGX-A100에 Docker Disk allocation이 지원 가능한 File system 구축하기

기본적으로 DGX-A100 머신의 디스크는 `ext4` file system으로 설정되어 있고, Docker는 `overlay2` storage driver를 사용하고 있습니다. 그러나 `overlay2` storage driver의 경우 backing file system이 `ext4`이 아닌 `xfs`(with `ftype=1`, `pquota` mount option)로 되어 있어야 disk allocation이 가능합니다. 따라서 사전 작업으로 DGX-A100 머신 내에 `raid0`으로 묶여있는 `/dev/md1` 디스크를 `xfs` file system으로 초기화합니다. 해당 작업을 진행하면 `/dev/md1` 디스크의 내용이 전부 초기화되므로 mount point를 확인하고 필요한 데이터가 있다면 반드시 백업해두시기 바랍니다.

0. 아래 명령어를 통해 DGX-A100 머신에 4개의 3.5TiB짜리 SSD가 `raid0` 옵션으로 묶여서 14TiB의 공간을 형성하고 있음을 알 수 있습니다.

```console
nvadmin@dgx-a100:~$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
...
nvme0n1     259:6    0  3.5T  0 disk
└─md1         9:1    0   14T  0 raid0 /raid
nvme4n1     259:7    0  3.5T  0 disk
└─md1         9:1    0   14T  0 raid0 /raid
nvme5n1     259:8    0  3.5T  0 disk
└─md1         9:1    0   14T  0 raid0 /raid
nvme3n1     259:9    0  3.5T  0 disk
└─md1         9:1    0   14T  0 raid0 /raid
```

또한, 다음 명령어를 통해 `/dev/md1` 디스크가 `ext4` file system으로 되어 있음을 알 수 있습니다. 이것을 `xfs` (with `ftype=1`) 로 변경하는 작업을 진행할 것입니다.

```console
nvadmin@dgx-a100:~$ sudo parted -l
...
Model: Linux Software RAID Array (md)
Disk /dev/md1: 15.4TB
Sector size (logical/physical): 512B/4096B
Partition Table: loop
Disk Flags:

Number  Start  End     Size    File system  Flags
 1      0.00B  15.4TB  15.4TB  ext4
```

1. 우선 `lsblk`로 확인한 `MOUNTPOINT`인 `/raid`를 unmount 시킵니다.

```console
nvadmin@dgx-a100:~$ sudo umount /raid
```

2. `/dev/md1` 디스크를 `xfs`로 포맷합니다. 이 때 `-n ftype=1` 옵션을 반드시 주도록 합니다. `-f` 옵션은 디스크 내에 내용물이 있어도 강제로 overwrite하는 옵션입니다.

```console
nvadmin@dgx-a100:~$ sudo mkfs.xfs -f -n ftype=1 /dev/md1
```

이 시점에 다시 `sudo parted -l`을 실행하면 다음과 같이 file system이 변경되었음을 확인할 수 있습니다.

```console
nvadmin@dgx-a100:~$ sudo parted -l
...
Model: Linux Software RAID Array (md)
Disk /dev/md1: 15.4TB
Sector size (logical/physical): 512B/4096B
Partition Table: loop
Disk Flags:

Number  Start  End     Size    File system  Flags
 1      0.00B  15.4TB  15.4TB  xfs
```

3. 다음 명령어로 `/dev/md1`을 `/raid` 디렉토리로 mount 시킵니다. `-o pquota` 옵션을 반드시 포함해주시기 바랍니다. `df` 명령어로 제대로 mount가 됐는지 확인할 수 있습니다.

```console
nvadmin@dgx-a100:~$ sudo mount -o pquota /dev/md1 /raid
nvadmin@dgx-a100:~$ df -h /raid/
Filesystem      Size  Used Avail Use% Mounted on
/dev/md1         14T  100G   14T   1% /raid
```

4. 모든 Docker container와 image를 삭제합니다.

```console
$ sudo docker rm -f $(docker ps -aq); docker rmi -f $(docker images -q)
```

5. Docker service를 중지합니다.

```console
$ sudo systemctl stop docker
```

6. Docker storage 디렉토리 내부를 비웁니다.

```console
$ sudo rm -rf /var/lib/docker
$ sudo mkdir /var/lib/docker
```

7. `/raid` 내에 새로운 디렉토리를 만들어서 bind mount 합니다.

```console
$ sudo mkdir /raid/docker
$ sudo mount --rbind /raid/docker /var/lib/docker
```

8. Docker service를 다시 시작합니다.

```console
$ sudo systemctl start docker
```


## 본 작업

### I. Google Form을 통한 수요 조사 진행

* 기본 인적 사항 외에 원하는 1) GPU 개수, 2) CPU 코어 수, 3) Memory 용량, 4) Disk 용량, 5) 운영체제, 6) CUDA 버전, 7) cuDNN 버전(선택 사항)을 조사할 필요가 있습니다.

* **II. Dockerfile을 통한 image 빌드하기**에서 사용하게 될 Dockerfile의 명령어가 현재로서는 Ubuntu 운영체제를 가정하고 작성되었으므로 운영체제로는 Ubuntu 계열만을 보기로 넣을 것을 추천드립니다.

* 수요 조사 예시: https://forms.gle/Y6iVQB3p6yadck3R9

### II. Dockerfile을 통한 image 빌드하기

* 예시: 사용자가 Ubuntu 18.04, CUDA 11.4.0, cuDNN 8 Developer을 선택한 경우

1. 빈 디렉토리(`cd ~` 등)에서 다음과 같은 내용의 `Dockerfile` 파일을 생성합니다. (빈 디렉토리가 아니라면 `.dockerignore` 파일을 생성해서 다른 폴더 및 파일을 무시하도록 합니다.) `Dockerfile`의 내용은 `nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04`라는 base image에 포트를 통한 SSH 접속이 가능케한 새로운 image를 빌드하기 위해 작성되었습니다. Base image 이름 및 태그를 검색하려면 [이 곳](https://hub.docker.com/r/nvidia/cuda/tags)을 이용해주세요.

```console
nvadmin@dgx-a100:~$ cat Dockerfile
# Instruction for Dockerfile to create a new image on top of the base image (nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04)
FROM nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04
ARG root_password
RUN apt-get update || echo "OK" && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo "root:${root_password}" | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

`.dockerignore` 파일 예시

```console
nvadmin@dgx-a100:~$ cat .dockerignore
folder_to_ignore
file_to_ignore
ignore_me*
```

2. `Dockerfile`을 이용해 새로운 image를 빌드합니다. 해당 image를 이용한 container의 비밀번호는 `password`로 초기화되어 있으며 이후에 각 사용자가 수정할 수 있습니다.

```console
$ docker image build --build-arg root_password=password --tag nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04-ssh .
```

### III. 각 사용자에게 나눠줄 container 만들기

* 예시: 사용자가 GPU 2개, CPU 코어 32개, Memory 128GB, Disk 512GB를 선택한 경우

1. 사용자가 선택한 요구사항에 맞게 container를 만듭니다. (사용자명을 `user1`이라고 가정)

우선 GPU Numa affinity를 확인하여 affinity rule에 위배되지 않는 GPU, CPU 조합이 available한지 확인합니다.

```console
nvadmin@dgx-a100:~$ lscpu
...
NUMA node0 CPU(s):               0-15,128-143
NUMA node1 CPU(s):               16-31,144-159
NUMA node2 CPU(s):               32-47,160-175
NUMA node3 CPU(s):               48-63,176-191
NUMA node4 CPU(s):               64-79,192-207
NUMA node5 CPU(s):               80-95,208-223
NUMA node6 CPU(s):               96-111,224-239
NUMA node7 CPU(s):               112-127,240-255
...
```

예를 들어, 위와 같은 경우에 GPU Node 1번을 할당한다면 CPU는 16-31번 혹은 144-159번 내에서 할당해줍니다. 아래와 같은 명령어로 할당을 완료합니다.

```console
$ docker container run --gpus '"device=0,1"' --cpus="32" -m 128G --storage-opt size=512G -d -P --name user1 nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04-ssh
```

2. Container 포트가 몇 번인지 확인하고 기록해둡니다.

```console
nvadmin@dgx-a100:~$ docker ps
CONTAINER ID   IMAGE                                             COMMAND                  CREATED          STATUS          PORTS                                     NAMES
24ad5af18c98   nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04-ssh   "/usr/sbin/sshd -D"      12 minutes ago   Up 12 minutes   0.0.0.0:49160->22/tcp, :::49160->22/tcp   user1
```

이제 `ssh -p 49160 root@147.46.92.213`와 같이 SSH를 통해 해당 container에 접속할 수 있습니다.

### IV. 실제 배포 및 각 사용자에게 전달할 자원 이용 설명서

OOO님, 신청하신 DGX-A100 자원이 할당되었습니다. 신청하신 자원은 NVIDIA Docker container로 만들어졌으며 아래와 같은 방법으로 접속하실 수 있습니다. 초기 비밀번호는 `password`입니다.

```console
ssh -p 49160 root@147.46.92.213
```

* Container에 접속하신 후 꼭 `passwd` 명령어를 통해 비밀번호를 변경해주시기 바랍니다. 비밀번호를 변경하지 않아서 생기는 보안, 데이터 손상 등의 문제는 저희가 책임지지 않습니다.

* Container로 데이터를 보내고 받으실 때에는 다음과 같은 명령어를 사용하시면 됩니다.

Container로 데이터를 보낼 경우

```console
user@desktop:~$ scp -P 49156 -r folder_to_send root@147.46.92.213:/root
```

Container에서 데이터를 보낼 경우

```console
root@194b0519ada6:~# scp -r folder_to_send user@<IP_ADDRESS>:/home/user
```

* 할당된 자원을 확인하실 때 `lscpu`, `free` 등의 명령어를 이용하시면 container가 아닌 host 머신 전체의 자원이 보이게 되므로 다른 방법을 사용하셔야 합니다. 아래와 같이 안내드립니다.

1. GPU

```console
root@24ad5af18c98:~# nvidia-smi
Tue May 17 10:00:30 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 510.47.03    Driver Version: 510.47.03    CUDA Version: 11.6     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100-SXM...  On   | 00000000:07:00.0 Off |                    0 |
| N/A   31C    P0    54W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
|   1  NVIDIA A100-SXM...  On   | 00000000:0F:00.0 Off |                    0 |
| N/A   30C    P0    52W / 400W |      0MiB / 40960MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

2. CPU: 아래 명령어를 실행해서 나온 결과를 `10000`로 나눔.

```console
root@24ad5af18c98:~# cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
320000
```

3. Memory: 아래 명령어를 실행해서 나온 결과를 `1024 * 1024 * 1024`로 나눔 (GiB 단위).

```console
root@24ad5af18c98:~# apt-get update && apt-get install -y cgroup-tools
root@24ad5af18c98:~# cgget -n --values-only --variable memory.limit_in_bytes /
137438953472
```

4. Disk
```console
root@24ad5af18c98:~# df -h /
Filesystem      Size  Used Avail Use% Mounted on
overlay         512G  2.7M  512G   1% /
```
* 기타 궁금하신 사항이 있으면 XXX로 연락주시기 바랍니다.

## 마무리 작업

### I. Container 회수 및 삭제

사용자가 container를 반납하면 다음과 같은 명령어로 강제 삭제할 수 있습니다.

```console
nvadmin@dgx-a100:~$ docker ps
CONTAINER ID   IMAGE                                             COMMAND                  CREATED          STATUS          PORTS                                     NAMES
24ad5af18c98   nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04-ssh   "/usr/sbin/sshd -D"      45 minutes ago   Up 45 minutes   0.0.0.0:49160->22/tcp, :::49160->22/tcp   user1
nvadmin@dgx-a100:~$ docker rm --force user1
user1
nvadmin@dgx-a100:~$ docker ps
CONTAINER ID   IMAGE                                             COMMAND                  CREATED          STATUS          PORTS                                     NAMES
```

만들어뒀던 image는 재사용이 가능하므로 특별한 경우가 아니면 image는 바로 삭제하지는 않는 것을 추천드립니다. 수요 조사 결과 사용자들의 요구사항이 비슷하다면 같은 image로 container만 다르게 만들어서 배포하는 것이 가능하기 때문입니다. Image가 너무 많이 쌓여서 용량을 많이 차지하고 있다면 (`docker images` 명령어로 확인 가능) 다음과 같은 명령어로 image를 강제 삭제하시기 바랍니다. 단, image가 삭제 가능하려면 해당 image로 만든 container가 없어야 합니다. 또한, 위에서처럼 base image에 포트를 통한 SSH 접속이 가능케한 새로운 image를 만든 경우, 그 새로운 image를 먼저 삭제하고, 그 다음에 base image를 삭제해야 합니다.

```console
nvadmin@dgx-a100:~$ docker images
REPOSITORY               TAG                                   IMAGE ID       CREATED          SIZE
nvidia/cuda              11.4.0-cudnn8-devel-ubuntu18.04-ssh   29e620c892e6   50 minutes ago   9.13GB
nvidia/cuda              11.4.0-cudnn8-devel-ubuntu18.04       d94cc7c49eef   2 weeks ago      8.98GB
nvadmin@dgx-a100:~$ docker rmi 29e620c892e6
Untagged: nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04-ssh
Deleted: sha256:29e620c892e6b7b3cd5f14772aece043ef10487ce3e9303f6d3408bedf2dce07
...
nvadmin@dgx-a100:~$ docker rmi d94cc7c49eef
Untagged: nvidia/cuda:11.4.0-cudnn8-devel-ubuntu18.04
Untagged: nvidia/cuda@sha256:867af080c04292bbab9b5971432c9f7955978f3c9e61cd613e86d95ea7926577
Deleted: sha256:d94cc7c49eef65a0c5eca81d6a88b067687baf1b4138ceebe22e5eb27a8007f3
...
nvadmin@dgx-a100:~$ docker images
REPOSITORY               TAG                         IMAGE ID       CREATED         SIZE
```

### II. 개인정보 파기

**I. Container 회수 및 삭제** 작업이 끝나면 해당 사용자의 개인정보를 파기해주시기 바랍니다.
