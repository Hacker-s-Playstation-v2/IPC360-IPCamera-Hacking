# IPC360-IPCamera-Hacking

본 문서에서는 IPC360 기반의 IP 카메라에 대한 연구를 진행하면서 알게 된 분석 환경 구축, 취약점 탐색, 취약점 공격 방법에 대해 소개한다.

![image](https://user-images.githubusercontent.com/39231485/144426439-68fecb3e-1711-4f7f-bf85-cd6836ca1ebb.png)

타겟으로 할 IP 카메라를 구경하던 중, 위 이미지에 적힌 문구를 보고 **해킹 욕구**를 참을 수 없었다.

이후 해당 기기를 구매하여 펌웨어 덤프, `UART`와 `SSH` 실행 등 분석에 필요한 환경 구성을 완료하였고, 3달여간 분석을 진행한 결과 운이 좋게도 기기 `LAN-side RCE`를 성공시킬 수 있었다.

현재 그 이외 다른 취약점들을 찾기 위해 분석 및 퍼징 중이며 찾게 된다면 쭉쭉 업데이트 할 예정이다.

# Contents

1. [분석환경 구축](/ANALYSIS.md)
   1. [플래시 메모리 덤프](/ANALYSIS.md/)
   2. [QEMU 가상환경 구축](/ANALYSIS.md/)
   3. [디버그 인터페이스](/ANALYSIS.md/)
   4. [쉘 접속](/ANALYSIS.md/)
   5. [디버깅 환경 구축](/ANALYSIS.md/)
2. [연구](/RESEARCH.md)
   1. [To Be Continued]()
