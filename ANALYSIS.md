## 목차 <!-- omit in toc -->

- [1. 분석환경 구축](#1-분석환경-구축)
	- [1.1. 플래시 메모리 덤프](#11-플래시-메모리-덤프)
	- [1.2. 디버그 인터페이스](#12-디버그-인터페이스)
	- [1.3. QEMU 가상환경 구축](#13-qemu-가상환경-구축)
	- [1.4. 디버깅 환경 구축](#14-디버깅-환경-구축)

---

# 1. 분석환경 구축

PE204 제품의 경우 펌웨어가 공개되어 있지 않아서 먼저 분석 환경을 구축한 뒤 취약점 탐색을 진행했다.

## 1.1. 플래시 메모리 덤프

IoT 해킹을 위한 첫 단계는 제품을 분해해서 PCB 기판을 살펴보는 것이다. 타겟 장비는 총 2개의 PCB로 구성되어 있었고, 그 중 핵심적인 기능을 담당하는 PCB 기판에서 플래시 메모리와 디버그 인터페이스를 모두 식별할 수 있었다.

앞면 | 뒷면
:-:|:-:
![image](https://user-images.githubusercontent.com/45416961/145000182-ab83b808-e534-4c85-9a34-962c2f63b725.png) |  ![image](https://user-images.githubusercontent.com/45416961/145000015-8b452901-13b4-4575-a350-a3504e2d6cef.png))

본격적으로 취약점 탐색을 진행하기에 앞서 기기 내의 펌웨어를 추출해야 했다. 기기의 펌웨어는 대부분 Non-volatile memory인 Flash에 저장되기 때문에 플래시 메모리를 읽게 된다면 펌웨어를 추출하여 분석을 진행할 수 있을 것이다.

![image](https://user-images.githubusercontent.com/39231485/144432489-78a1c7c4-f80f-47d4-8faa-04d4c879cd12.png)

어떤 식으로 펌웨어를 추출할까 하다가 보드 내에 `UART` 홀이 있어 연결을 시도해봤더니 `115200` baudrate로 기기와 연결할 수 있었다.
부트 쉘 내부 환경변수 중 `bootdelay`가 1초로 설정되어 있어서 별다른 Fault injection 없이 `delay` 중에 아무키나 입력하면 수월하게 부트 쉘에 접근할 수 있었다.

플래시 메모리를 기기와 연결하는 `sf probe` 명령어 이후, 데이터를 읽는 `sf read [memory addr] [flash offset] [length]` 통해 내부 데이터를 모두 램으로 불러왔다. 그 다음 `md` 명령어를 통해 램에 있는 모든 데이터들을 읽었다.
아래는 `md`를 통해 읽은 데이터를 파일에 저장하는 `python` 코드이다.

```python
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

읽은 펌웨어를 `binwalk`로 분석해보니 파일시스템이 잘 뽑히는 것을 확인할 수 있었다.

<img src="https://user-images.githubusercontent.com/39231485/144436560-cb7071d4-f649-4fcc-a8da-1c510859eca6.jpg" width="40%">

혹시 모르니 위에 보이는 ic 후크 클립으로 플래시 메모리를 라즈베리파이와 연결하여 `spi` 인터페이스로 데이터를 읽어보았다.

![image](https://user-images.githubusercontent.com/39231485/144437448-b5c978da-7af5-46a5-90b3-75cba6eae04e.png)

`Flashrom`에서 `GD25Q64` 칩을 지원해줬기 때문에 아래 명령어를 사용하여 쉽게 읽을 수 있었다.

```bash
flashrom -p linux_spi:dev=/dev/spidev0.0 -r ./data.bin
```


## 1.2. 디버그 인터페이스

이번 섹션에서는 디버그 인터페이스를 통해 쉘에 접속하는 방법을 기술한다. **1.1 플래시 메모리 덤프** 섹션에서도 언급했듯이, PCB에 프린팅 된 식별자 덕분에 멀티미터기 없이 `UART` 인터페이스를 쉽게 발견할 수 있었다. 인터페이스의 정확한 위치와 연결 방법은 아래 이미지를 참조하면 된다.

디버그 인터페이스 위치 | UART 연결 방법
:-:|:-:
![image](https://user-images.githubusercontent.com/45416961/144432852-cc0a1b13-7785-4b5b-ab34-46ffa130f079.png) |  ![image](https://user-images.githubusercontent.com/45416961/144437762-e38c7178-44e6-4302-8d47-ff89a296697d.png)


**[UART 연결 과정 정리]**

1. PCB에서 디버그 인터페이스를 식별한다.
2. 각 핀이 `VCC`, `GND`, `TX`, `RX` 중 어떤 것인지 구분한다.
3. USB-to-TTL Serial 케이블을 준비하고 `GND`, `TX`, `RX` 핀을 연결한다.
4. PC에서 `PuTTY`, `XShell`(Windows), `Screen`(Mac), `Minicom`(Linux) 등을 이용하여 시리얼 콘솔에 접속한다. (`Baudrate`는 일반적으로 115200이 많이 사용된다)

| ![image](https://user-images.githubusercontent.com/45416961/144437904-505d6f7b-c73f-4933-992d-d7ce48ae4fcf.png) |
| :-: |
| 부트로더 쉘 로그 |


논문이나 학회 발표 등 각종 분야에서 디버그 인터페이스에 대한 연구가 활발히 이루어지고 있고, 제조사에서도 하드웨어 보안에 대한 중요성을 높이 평가하고 있는 실정이다. 그래서 비교적 최근에 나온 IoT 제품들은 하드웨어/소프트웨어 측면의 방법을 통해 디버그 인터페이스를 비활성화 하거나, 인증 메커니즘 및 매직 키를 적용하여 접근성이 떨어지도록 전처리 하기도 한다.

본 문서에서 다루는 IP 카메라 또한 전처리가 되었는지 부트로더 쉘에는 바로 진입할 수 있었지만 리눅스 쉘에는 진입할 수 없었다. 분석 및 공격을 수행하기 위해서는 쉘 접속이 필수불가결한 요소였으므로, 어떻게든 문제를 해결해야 했다.

먼저, 부트로더 쉘에서 `env print` 명령어를 통해 환경 변수를 살펴보았다.


```
isvp_t21# env print
HWID=0000000000000000000000000000000000000000
ID=0000000000000000000000000000000000
IP=192.202.5.91
MAC=40:6A:8E:7E:CD:5B
SENSOR=F23
SSID_NAME=SC2-44
SSID_VALUE=abc.12456
TYPE=T21N
WIFI=8188FTV
baudrate=115200
bootargs=console=ttyS1,115200n8 mem=39M@0x0 rmem=25M@0x2700000 init=/linuxrc rootfstype=squashfs root=/dev/mtdblock2 rw mtdparts=jz_sfc:512K(boot),1600k(kernel),2816k(root),1536k(user),832k(web),896k(mtd)
bootcmd=sf probe;sf read 0x80600000 0x80000 0x280000; bootm 0x80600000
bootdelay=1
ethact=Jz4775-9161
ethaddr=40:6A:8E:7E:CD:5B
gatewayip=193.169.4.1
ipaddr=193.169.4.81
ipncauto=1
ipncuart=1
loads_echo=1
netmask=255.255.255.0
serverip=193.169.4.2
stderr=serial
stdin=serial
stdout=serial

Environment size: 755/65532 bytes
```

그 중 `bootargs`의 `console`과 `init`이 눈에 띄었다.

```
bootargs=console=ttyS1,115200n8 mem=39M@0x0 rmem=25M@0x2700000 init=/linuxrc 
```

- `console` : 시리얼 콘솔 세션
- `init` : 부팅 후 실행될 초기 명령


 `console` 값이 `ttyS1`로 설정되어 있는데, 부팅 후 `linuxrc`를 통해 초기화 되는 과정에서 콘솔 세션이 `/dev/ttyS1` 경로에 제대로 생성되지 않으면 통신이 되지 않을 수 있다. 즉, 부트로더에서 정의한 콘솔과 리눅스 커널에서 정의한 콘솔의 불일치로 인해 쉘 접속이 불가능할 수 있는 가능성을 염두에 두고 분석을 이어나갔다.

*1.1 플래시 메모리 덤프* 에서 추출한 펌웨어를 확인해본 결과 초기화 스크립트인 `/etc/inittab`에서 ttyS1로 콘솔 세션을 설정하는 부분이 존재하지 않았다.

```
# Put a getty on the serial port
console::respawn:/sbin/getty -L console 115200 vt100 # GENERIC_SERIAL
```

그래서 `ttyS1` 세션을 열어주도록 아래와 같이 코드를 수정했고, 추가적으로 `rcS` 스크립트에 `telnetd`를 구동하는 구문을 삽입하여 부팅과 동시에 원격 접속이 가능하도록 했다.

```
➜  squashfs-root cat etc/inittab | grep getty
# Put a getty on the serial port
ttyS1::respawn:/sbin/getty -L ttyS1 115200 vt100 # GENERIC_SERIAL

➜  squashfs-root cat etc/init.d/rcS | grep telnet
# Start telnet daemon
telnetd &
```

이렇게 핵심 파일들을 수정한 뒤, squashfs를 다시 빌드하였고, 기존 이미지와 크기가 동일한지 확인해보았다.
```
-rwxrwxrwx 1 lee lee 2699264 Aug 24  2021 modified.img
-rwxrwxrwx 1 lee lee 2883584 Aug 24 22:25 origin.img
```

![image](https://user-images.githubusercontent.com/45416961/144741194-9a723733-5e56-4375-aab1-28c71cd50859.png)
![image](https://user-images.githubusercontent.com/45416961/144741203-64626ee2-08cf-4987-856e-980455baed3a.png)

`ls` 명령어로 확인했을 때는 두 `img` 파일의 크기가 달라보였지만, 헥스 에디터(`HxD`)로 열어서 확인해보니 바이트가 끝나는 지점의 오프셋이 모두 `0x292FFF`로 동일했다. 어차피 부트로더에서 메모리 영역을 덮어쓸 때 오프셋 값을 지정해 주기 때문에 파일을 카빙 하지는 않았다.

```
Boot partition (524,288 bytes)
Kernel partition (1,638,400 bytes)
Root partition (2,883,584 bytes)
User partition (1,572,864 bytes)
Web partition (851,968 bytes)
MTD partition (917,504 bytes)
```

이렇게 수정한 이미지는 Root partition 영역에 넣어 주면 된다.

| ![image](https://user-images.githubusercontent.com/45416961/144741320-f81f53f0-17f3-4ff4-8859-9051b54afb60.png) |
| :-: |
| 부트로더 쉘에서 loady 실행 |

| ![image](https://user-images.githubusercontent.com/45416961/144741281-277cc7b2-cf20-42a2-bd38-8e9f22a741b1.png) |
| :-: |
| ExtraPuTTY로 파일 전송 |

ExtraPuTTY 프로그램을 설치한 뒤, `YMODEM` 프로토콜로 바어너리를 전송할 수 있다. 이 때, 부트로더에서 `loady` 라는 커맨드를 먼저 실행해 주어야 한다.

| ![image](https://user-images.githubusercontent.com/45416961/144741456-cf4f9366-18b9-476c-acb1-4ae2156bb906.png) |
| :-: |
| 메모리 오프셋 |


`ExtraPuTTY`로 업로드 한 파일은 메모리 영역 어딘가에 저장되는데, 그 위치는 `loady` 명령어의 실행 로그로 출력된다. 우리의 경우 `0x82000000` 주소에 저장되었다. 따라서 해당 위치에 저장되어 있는 데이터를 플래시 메모리의 `Root Partition`에 옮겨주면 모든 과정이 완료된다. `Root Partition`의 오프셋은 `Boot partition (524,288 bytes) + Kernel partition (1,638,400 bytes)` = `2162688` = `0x2100000` 이다.


```
# Appendex : sf 커맨드 사용법
# Reference : https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842223/U-boot

uboot> sf
Usage:
sf probe [[bus:]cs] [hz] [mode] - init flash device on given SPI bus and chip select
sf read addr offset len         - read 'len' bytes starting at 'offset' to memory at 'addr'
sf write addr offset len        - write 'len' bytes from memory at 'addr' to flash at 'offset'
sf erase offset [+]len          - erase 'len' bytes from 'offset'; '+len' round up 'len' to block size
sf update addr offset len       - erase and write 'len'bytes from memory at 'addr' to flash at 'offset
```

![image](https://user-images.githubusercontent.com/45416961/144741384-30a7a2cc-f8ca-48c6-8b0e-c334aa2f0b2f.png)


위 명령어를 통해 플래시 메모리를 덮어쓴 뒤 `bootcmd`를 통해 부팅을 시도했고, 그 결과 다음과 같이 리눅스 쉘에 성공적으로 접속할 수 있었다.

| ![image](https://user-images.githubusercontent.com/45416961/144741610-d337f520-bc18-4a85-a6b7-a10170694eec.png) |
| :-: |
| UART 접속 |

| ![image](https://user-images.githubusercontent.com/45416961/144741708-1da96c4c-7172-4745-bee2-44d8fdc5cb0d.png) |
| :-: |
| TELNET 접속 |


## 1.3. QEMU 가상환경 구축

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


## 1.4. 디버깅 환경 구축

타겟 장비의 아키텍쳐에 맞는 `gdb/gdbserver`를 static하게 빌드하고, 이를 통해 디버깅을 수행할 수 있다. 우리는 [여기](https://github.com/rapid7/embedded-tools/blob/master/binaries/gdbserver/gdbserver.mipsle)에서 `MIPS little-endian` 아키텍쳐로 static 컴파일 된 `gdbserver`를 다운로드 받았다. 이후 `wget/nc` 커맨드로 바이너리를 옮기고 디버깅을 수행하였다.

`gdbserver`보다 `gdb`를 사용했을 때 디버깅 속도가 훨씬 빠른데 아쉽게도 타겟 장비의 플래시 메모리 용량이 매우 작아서 `gdb`를 옮길 수 없었다. 조금 느린 속도만 감수한다면 `gdbserver` 만으로도 큰 문제 없이 디버깅을 할 수 있다.

여담으로 만약 아키텍쳐에 맞게 static 컴파일 된 바이너리가 없다면 직접 소스코드를 받아서 크로스 컴파일 해야 한다.

| ![image](https://user-images.githubusercontent.com/45416961/144741107-37146db2-7d9e-4a6c-8bf1-9d1923a167ef.png) |
| :-: |
| 디버깅 화면 |
