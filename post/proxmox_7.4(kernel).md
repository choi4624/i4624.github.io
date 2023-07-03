proxmox 7.4 설치 및 VM 올리는 분투기

1. 본 글은 인텔 13세대 (12세대 포함) 시스템을 바탕으로 진행한 proxmox 를 설치하는 방법을 기록한 것으로 추후 삽질을 줄이고 체크해야 할 포인트들을 기록해두는 용도로 만든 글입니다.
2. 현 시점에서 proxmox 8 이 등장했으나, 6.2.x kernel로 설치한 7.4의 경우 쉽게 업그레이드 가능하니 8을 설치할 때에도 사용할 수 있습니다.
3. 뭔지 잘 모르는 상세한 내용은 proxmox wiki 등을 같이 보면 좋습니다. 한글로 적어서 읽기 조금 편하다 + 영알못 (읽기 싫은사람 포함) 을 위한 축약본에 가깝습니다.

* 하드웨어 소개

intel i5 13500

intel B660

ddr5 64GB

M.2 NVMe disk 1

M.2 based ASM1166 chipset SATA Extension card (알리에서만 파는 물건입니다. 해당 칩셋은 오리코에서 만드는 카드에서도 들어갑니다. )

GT 1030

intel X550-T2 (pci-e 2.0 x8)

HDD 5개

SATA SSD 2개 > 1개는 host (proxmox), 1개는 vm 을 위해 할당

host SSD에서 iso 등을 넣어서 vm 설치를 하는 저장공간으로도 쓸 수 있기 때문에 256GB 정도의 용량이 있으면 좋습니다. 아예 host에 1T를 할당하고 모든 VM의 OS 공간으로 사용하셔도 무방합니다.

단, proxmox 내에 존재하는 LVM-Thin 을 이용할 수 없는 디스크 이므로 주의해야 합니다. 오직 directory 로만 가능합니다.

---

wifi -> intel AX211

wifi 칩셋이 있는 경우 proxmox 페이지에서 제공하는 부팅 파일을 통한 설치가 되지 않고 debian 을 통해 간접적으로 설치하는 것만 가능합니다.

온보드 wifi 끄는 기능이 없는 상황이 대다수에, 카드를 물리적으로 제거하는 것도 부담스러우므로 반드시 debian 12를 설치해주셔야 합니다.

---

본 페이지는 위의 wifi 칩셋이 있다는 가정 하에 작성됩니다.

* requirements

본 글은 윈도우 설치 정도는 그냥 해내는 분들을 기준으로 작성되었으며 만약 장치에 직접적으로 os를 설치하는 baremetal 환경에서의 usb 설치를 하질 않으셨다면 ubuntu / debian  부팅 usb 만들기 등의 과정을 익히는 것을 추천합니다.

또한 ssh, 리눅스 파일시스템, vi 혹은 nano / gedit 등을 사용하여 시스템 파일 수정하는 법을 아는 사람에게 유용합니다. 어느정도의 기본 linux 지식과 영어 게시글에 대한 부담이 적으신 분들을 위한 글(즉, 나) 입니다.

주요 과정

1. Debian 설치
2. Debian을 통해 proxmox kernel 로 변경
3. proxmox에서 Linux Bridge (vm switch) 설정하기
4. pci express 장치 할당
5. VM 설치
6. xpenology 설치
7. VM 내에서 windows 11 제한사항
8. LVM-thin
9. 총 결과
10. Debian 설치

1. Debian 설치 
Debian 설치는 12 bookwarm 기준으로 진행하며 아래 웹 사이트를 통해 설정하여 진행할 수 있다.

[Install Proxmox VE on Debian 12 Bookworm - Proxmox VE](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm)

NIC 가 여러개인 경우, 주 네트워크(가능하면 인터넷 네트워크)에만 설정해두고, 보조 네트워크 혹은 단독 네트워크도 등록하는 것은 지양해주는 편이 좋습니다. 추후 linux bridge 설정할 때에 보조 네트워크를 쉽게 물려야 하는데, 보조 네트워크에 장치가 등록되어 있으면 끄고 네트워크 서비스 다시 설정하고 터미널 다시 설정하는 등의 삽질이 들어갑니다.

update 등을 진행하기 위해 인터넷 연결을 필요로 합니다.

2. Debian을 통해 proxmox kernel 로 변경

![](assets/20230703_160844_2023-07-03_160833.png)

recommended는 os 선택지를 grub 내에서 남겨놓는걸 없애버리는 걸 의미하므로, 그냥 해주셔야 합니다.

proxmox를 host로 설정할 정도면 리눅스 멀티부팅을 할 이유는 없을테니

참고: wifi 드라이버가 proxmox 패키지에 설치되어 있지 않으므로, wired nic(랜카드)를 사용해서 연결하거나 별도의 작업을 진행하여 wifi 드라이버를 통해 proxmox를 실행할 수 있도록 해야 합니다.

3. proxmox에서 Linux Bridge (vm switch) 설정하기

Proxmox VE 를 installation file로 설치한 경우 기본 NIC를 자동으로 bridge로 설정하여 연결합니다. 다만 debian을 통한 설치는 자동으로 설정되지 않으므로, nic를 bridge를 설정해주셔야 합니다.

예시>

3nic (2개는 메인 망, 1개는 10G 망-pc 간 direct connection)

![](assets/20230703_162748_2023-07-03_162440.png)

주의사항: 설정에 실패하는 경우 웹 접근이 어려워지고 저 gui 설정을 이제 cli로 pc에서 설정해야 합니다.

당연히 물리적 접근이 안되면 매우 곤란한 상황에 처해질 수 있습니다.

gui가 싫거나
실패하거나
기타 여러 이유로 안되는 경우
아래 링크와 이미지를 참고해주세요

[Network Configuration - Proxmox VE](https://pve.proxmox.com/wiki/Network_Configuration)


![](assets/20230703_165529_2023-07-03_164859.png)

설명

nic 카드들 목록 

enp5s0 (RealTek 2.5G nic)

enpls0f0 (X550 10G -1 )

enpls0f1 (X550 10G -2 )

enp5s0 은 direct로 연결시킨 nic인데, 어떠한 이유로 스위치가 잘 작동하지 않거나

pci express로 장착된 **NIC를 교체했을 때 연결할 네트워크 접근 경로로서** 사용합니다.

dhcp로 설정해도 괜찮지만 dhcp를 잘 할당받지 못하거나, dhcp가 없는 단독망 환경 등에서 설정할 수 있기 때문에 ip 할당을 해놓는 것을 추천해 드립니다. 

vmbr0 으로 할당한 vm에 192.168.0.x대역으로 장치를 passthrough 하여 network에 할당한 상태입니다.

브릿지 설장할 네트워크 장치를 물려야 하며, 주의사항으로는 실제 nic는 활성화 되지 않은 상태로 두어야 합니다. 

vmbr1 역시 vm에 192.168.1.x 대역으로 할당하는 용도로 사용합니다. 

게이트웨이 설정이 되어있지 않은데 10G 라우터나 스위치는 없으므로 단독망이기 떄문에 그렇습니다. 

없어도 ip만 guest나 외부 장치에 잘 충돌없이 집어넣으면 작동합니다. 

* sudo systemctl restart network-service 등 설정 완료 후 network 관련 서비스 재시작을 해야 합니다.
* 당연히 저 파일은 sudo로 편집해야 합니다.


4. pci express 장치 passthrough

iommu는 필수입니다. 그리고 몇 가지 설정들을 해야 하므로 아래의 글을 참고해주세요. 

[https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/ (github.com)](https://gist.github.com/qubidt/64f617e959725e934992b080e677656f)

# Configuring Proxmox 

이 부분만 진행


shell에서 lspci로 장치 확인한 다음 관련 장치들 gui에서 할당하거나 할 수 있음.

당연한 이야기지만 이미 할당된 장치에는 사용하면 안돼며 그래픽카드의 경우 proxmox host에 있는 gpu를 치우고 vm에 할당해줘야 합니다.

![](assets/20230703_173021_2023-07-03_172139.png)



![](assets/20230703_173115_image.png)



iommu랑 드라이버 설정만 좀 건든 다음엔 gui로 별로 어렵지 않게 gpu passthrough 설정이나 pci passthrough 설정이 가능하며, nvme 장치를 direct로 할당하여 passthrough 해 가지고 바로 부팅시킬 수도 있습니다.

안쓰는 랜카드 중 하나를 할당해서 써도 됩니다. windows의 경우 wifi direct를 기반으로 한 TV 영상전송 혹은 다른 기능들을 쓸 수 있기 때문에 windows guest에 할당하면 유용합니다. 


즉, 기존 nvme 디스크를 쉽게 부팅하여 연결할 수 있습니다. 스냅샷 기능이 필요하지 않고, 추후 베어메탈로 이동할 nvme의 경우 위 방법으로 진행하도 되나 windows 10/11의 경우는 추천하지 않습니다. 기종 전환하면서 Nested Virtualization 과 관련한 문제가 있는 모양인지 부팅 실패가 무조건 발생합니다.

이는 윈도우가 현재 사실상 윈도우의 탈을 쓴 hyper-V 기반의 OS 인 점이 가장 큰 이유인듯 합니다. vmware  나 다른 앱플레이어를 실행하는게 어렵고 안드로이드 스튜디오 내의 가상 os 설치에도 지장이 있으므로 관련 기능 (WSL 포함)을 하고자 한다면 포기하는 편이 좋습니다.

아니면 이 문제가 인텔 13세대라는 비교적 최신 cpu라서 발생하는 문제일 수도 있습니다. 추후 패치나 업그레이드 혹은 다른 여러 문제가 해결되면 update 기록을 남길 예정입니다.


![](assets/20230703_173550_image.png)

참고: pci-express로 할당하셔야 합니다. 

6. VM 설치

7. VM 내에서 windows 11 제한사항

8. xpenology 설치


9.  LVM-thin


10. 총 결과