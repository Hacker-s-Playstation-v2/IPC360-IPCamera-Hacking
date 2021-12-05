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

먼저 취약점 탐색을 진행하기 위해 기기 내의 펌웨어를 추출해야 한다.
기기의 펌웨어는 대부분 Non-volatile memory인 Flash에 저장되기 때문에 Flash 메모리를 읽게 된다면 펌웨어를 추출하여 분석을 진행할 수 있을 것이다.

![image](https://user-images.githubusercontent.com/39231485/144432489-78a1c7c4-f80f-47d4-8faa-04d4c879cd12.png)

어떤 식으로 펌웨어를 추출할까 하다가 보드 내에 UART 홀이 있어 연결을 시도해봤더니 `115200` baudrate로 기기와 연결할 수 있었다.
부트 쉘 내부 환경변수 중 `bootdelay`가 1초로 설정되어 있어서 별다른 fault injection 없이 delay 중에 아무키나 입력하면 수월하게 부트 쉘에 접근할 수 있었다.

flash를 기기와 연결하는 `sf probe` 명령어 이후 flash 데이터를 읽는 `sf read [memory addr] [flash offset] [length]` 통해 flash 내부 데이터를 모두 램으로 읽어들인 후 `md` 명령어를 통해 램에 있는 모든 데이터들을 읽었다.
아래는 md를 통해 읽은 데이터를 파일에 저장하는 python 코드이다.
```
from serial import Serial
import binascii

def macro(text) :
	a = bytearray.fromhex(text)
	a.reverse()
	s = ''.join(format(x, '02x') for x in a)
	return binascii.unhexlify(s)

port = "COM3"
baud = 115200

ser = Serial(port, baud, timeout=1)

text = "md 0x80600000 0xa0000\r\n"

ser.write(text.encode())

pp = ""

out = ""

f = open("dump.dat", "wb+")
#f = open("dump.dat", "ab") // APPEND

while True :
	try:
		out = ser.readline().decode("utf-8")
	except:
		pass
	if ":" in out:
		res = out.split(" ")
		save = ""
		for i in range(1,5):
			save += macro(res[i])
		f.write(save)
		if "000:" in res[0]:
			print(res[0])
```
![image](https://user-images.githubusercontent.com/39231485/144435207-050426a0-a1f4-4fd4-8acf-49803aa11b6e.png)

읽은 펌웨어를 binwalk에 돌려보니 파일시스템이 잘 뽑히는 것을 확인할 수 있었다.

<img src="https://user-images.githubusercontent.com/39231485/144436560-cb7071d4-f649-4fcc-a8da-1c510859eca6.jpg" width="40%">

혹시 모르니 위에 보이는 flash에 ic 후크 클립으로 라즈베리파이와 연결하여 spi 인터페이스로 flash 데이터를 직접적으로 읽어줬다.

![image](https://user-images.githubusercontent.com/39231485/144437448-b5c978da-7af5-46a5-90b3-75cba6eae04e.png)

Flashrom에서 GD25Q64 flash를 지원해줬기 때문에 아래 명령어를 사용하여 쉽게 읽을 수 있었다.

`flashrom -p linux_spi:dev=/dev/spidev0.0 -r ./data.bin`

## 1.2. QEMU 가상환경 구축

System mode emulation을 통해 ip 카메라의 전체 시스템을 구현해주기 위해 아래와 같이 qemu를 실행시켰다.

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

이 이후에 해당 qemu 안에 `squashfs-root` 파일시스템을 넣어주고 chroot . ./bin/sh로 해당 파일시스템의 쉘에 접속하였다.
ip 카메라의 서비스를 제공하는 바이너리인 `Alloca`를 실행시켜보니 아래와 같은 에러가 발생했다.

![image](https://user-images.githubusercontent.com/39231485/144738887-3b55c567-b0b9-40d3-a78f-3a78c3c95e0e.png)


이를 고쳐주기 위해 `Binary Patch`와 `LD_PRELOAD`를 사용해 main 함수를 덮어 프로그램을 컨트롤 하려 했지만 결국에 해내지 못했다.

## 1.3. 디버그 인터페이스


## 1.4. 쉘 접속


## 1.5. 디버깅 환경 구축
