---
title: "Azure Linux VM에 Blobfuse 설정하기"
date: 2021-07-22 09:07:82
category: azure
draft: false
---

![](images/setup-blobfuse.svg)

Azure Linux VM에서 Azure Storage Account - Blob을 마운트하여 파일시스템의 일부처럼 사용할 수 있게 하는 blobfuse라는 툴이 있다.

아래에서는 blobfuse connection config, sh 파일 작성 및 권한 설정, fstab 등록을 통해 재시작시 자동 마운트, 사용자계정으로 blobfuse를 사용하게 하는 방법을 다룬다. 아래는 Ubunut 20.04를 기준으로 작성하였다. 

## blobfuse 설치

아래 문서 blobfuse 설치만 진행하고, mount 단계는 건너뛰자.

[https://docs.microsoft.com/ko-kr/azure/storage/blobs/storage-how-to-mount-container-linux](https://docs.microsoft.com/ko-kr/azure/storage/blobs/storage-how-to-mount-container-linux)

## blobfuse 폴더 생성, 파일 생성, 권한 부여

먼저 경로 생성 및 소유권을 부여한다. 여기서는 사용자계정을 localadm으로 하였다.

```bash
sudo mkdir /usr/share/blobfuse
sudo chown localadm:localadm /usr/share/blobfuse
cd /usr/share/blobfuse/
touch fuse_connection.cfg
touch fuse_mount.sh
chmod 755 fuse_mount.sh
```

## fuse_connection.cfg

파일에는 Azure Portal에서 생성한 Storage Account의 name, key, blob의 container 이름을 입력하여 저장한다.

```bash
vi fuse_connection.cfg
------------------------
fuse_connection.cfg
------------------------
accountName
accountKey
containerName
------------------------
```

## fuse_mount.sh

 blobfuse에서 사용할 temp 저장소 경로, config 파일 경로, mount 경로를 입력하고 저장한다.

```bash
vi fuse_mount.sh
------------------------
fuse_mount.sh
------------------------
#!/bin/bash

# Variables
username=localadm
blobfusetmp="/mnt/resource/blobfusetmp"
cfgfile="/usr/share/blobfuse/fuse_connection.cfg"
mountloc="/myblob"

# Create Folder
newdir () {
	if [ $# -eq 0 ]; then
		echo "Usage : newdir {directory}"
	elif [ ! -d $1 ]; then
        sudo mkdir $1 -p
        sudo chown ${username}:${username} $1
	fi
}

newdir ${blobfusetmp}
newdir ${mountloc}

# if parameter does not exist
if [ "$#" -gt 0 ]; then
	mountloc=$1
fi

# BlobFuse Mount
blobfuse ${mountloc} --tmp-path=${blobfusetmp} --config-file=${cfgfile} -o attr_timeout=240 -o entry_timeout=240 -o negative_timeout=120 --log-level=LOG_DEBUG --file-cache-timeout-in-seconds=120 -o allow_other
------------------------
```

## Run & Test

이제 fuse_mount.sh 를 실행하고, 위에서 설정한 마운트 경로에 들어가 blob에 저장한 파일이 보이는지 확인한다.

```bash
./fuse_mount.sh
cd /myblob
```

## fstab
리부팅 후에도 항상 마운트 상태가 되도록 fstab 맨 마지막 라인에 한 줄 추가하여 준다. 

각 항목 띄어쓰기는 space로 해도 되지만, 보기 편하게 하기 위해 tab으로 구분하여 준다.

```bash
/etc/fstab
------------------------
/etc/fstab
------------------------
/usr/share/blobfuse/fuse_mount.sh    /myblob      fuse    _netdev
------------------------
```

## Unmount
혹시나 마운트를 해제해야 하는 상황이 오면 아래 명령어를 통해 unmount할 수 있다.

```bash
fusermount -u /myblob
```

## 🚨 주의할 점
blob은 일반 파일시스템과 달라 파일 동시 쓰기가 불가하기 때문에, 여러 호스트에 마운트하여 한 파일에 로그를 쌓는다든지 하는 등의 행위를 하면 에러가 발생하므로, 여러 호스트에서 동시 마운트하여 사용시에는 컨테이너를 구분하거나 하위 폴더를 구분하여 저장하도록 하자.