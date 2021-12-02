## 목차 <!-- omit in toc -->

- [1. 분석환경 구축](#1-분석환경-구축)
  - [1.1. 플래시 메모리 덤프](#11-플래시-메모리-덤프)
  - [1.2. QEMU 가상환경 구축](#12-qemu-가상환경-구축)
  - [1.3. 디버그 인터페이스](#13-디버그-인터페이스)
  - [1.4. 쉘 접속](#14-쉘-접속)
  - [1.5. 디버깅 환경 구축](#15-디버깅-환경-구축)

---

# 1. 분석환경 구축

PE204 제품의 경우 펌웨어가 공개되어 있지 않아서 먼저 분석환경을 구축한 뒤 취약점 탐색을 진행했다.

## 1.1. 플래시 메모리 덤프



## 1.2. QEMU 가상환경 구축

```bash
#!/bin/bash

if [ ! -f debian_wheezy_mipsel_standard.qcow2 ]; then
	wget https://people.debian.org/~aurel32/qemu/mipsel/debian_wheezy_mipsel_standard.qcow2
fi

if [ ! -f vmlinux-3.2.0-4-4kc-malta ]; then
	wget https://people.debian.org/~aurel32/qemu/mipsel/vmlinux-3.2.0-4-4kc-malta
fi

DISK='./debian_wheezy_mipsel_standard.qcow2.1'
KERNEL='./vmlinux-3.2.0-4-4kc-malta.1'

SSH_PORT=2223
TELNET_PORT=2323
HTTP_PORT=8081
HTTP_PORT_2=41876

qemu-system-mipsel -M malta -kernel $KERNEL -hda $DISK -append "root=/dev/sda1" -nographic -redir tcp:$SSH_PORT::22 -redir tcp:$TELNET_PORT::23 -redir tcp:$HTTP_PORT::80 -redir tcp:$HTTP_PORT_2::41876
```


## 1.3. 디버그 인터페이스


## 1.4. 쉘 접속


## 1.5. 디버깅 환경 구축