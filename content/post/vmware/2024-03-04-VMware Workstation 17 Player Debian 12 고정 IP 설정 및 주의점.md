---
title: VMware Workstation 17 Player Debian 12 고정 IP 설정 및 주의점
description:
date: 2024-03-04
summary: VMware Workstation 17 Player Debian 12 고정 IP 설정 및 주의점
---

## 서론

VMware Workstation 17 Player 가상 머신 환경 중 Debian 12에서 고정 IP를 설정하는 방법과 주의점을 살펴보겠습니다.

## 환경

- Host PC
  - OS: Windows 11
- VMware Workstation 17 Player
  - OS: Debian 12

## 구축

Debian 12는 interfaces 또는 netplan을 통해 고정 IP를 설정할 수 있습니다. 예시로 사용된 네트워크 인터페이스는 `ens32`입니다. 또한 예시로 사용된 IP 대역은 `192.168.100.X`, 서브넷 마스크는 `255.255.255.0`, 게이트웨이는 `192.168.100.2`입니다. IP 대역, 서브넷 마스크, 게이트웨이는 `C:\ProgramData\VMware\vmnetnat.conf`에서 확인할 수 있습니다.

### interfaces

`/etc/network/interfaces`를 주석을 제외하고 보면 다음과 같습니다.

```txt
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens33
iface ens32 inet dhcp
```

`ens32`에 설정된 DHCP 대신 `192.168.100.10`으로 IP를 고정하겠습니다. `dns-nameservers` 설정은 선택입니다. 이 글에서는 생략하겠습니다.

```txt
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens32

auto ens32
iface ens32 inet static
    address 192.168.100.10  # 고정 IP 주소
    netmask 255.255.255.0  # 서브넷 마스크
    gateway 192.168.100.2    # 게이트웨이 IP 주소
```

설정을 마친 후, 파일을 저장하고 닫습니다. 그리고 네트워크 서비스를 다시 시작하여 변경 사항을 적용합니다.

```sh
sudo systemctl restart networking
```

> 자문자답 Q&A
>
> Q. ifconfig를 입력했을 때 lo를 제외한 다른 네트워크 인터페이스(ens'숫자')가 뜨지 않아요!  
> A. 오탈자에 주의하고 다시 한번 봐주세요.
>
> Q. ifconfig를 입력했을 때 다른 네트워크 인터페이스가 뜨지만, 인터넷이 안 돼요!  
> A. 게이트웨이 주소를 확인해 주세요.

### netplan

제 Debian 12의 경우 `/etc/netplan` 디렉터리애 설정 파일이 없어서 `01-netcfg.yaml` 이름으로 파일을 생성했습니다.

```yaml
netowrk:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.100.10/24
      gateway4: 192.168.100.2
  version: 2
```

파일 저장 후, 아래와 같은 명령어로 적용할 수 있습니다.

```sh
sudo netplan apply
```

> 자문자답 Q&A
>
> Q. `Permissions for 파일 are too open. Netplan configuration should NOT be accessible by others.` 다음과 같은 경고가 떠요!  
> A. `chmod`로 파일 권한을 600으로 변경해 주세요.
>
> Q. `gateway4 has been deprecated use default routes instead.` 다음과 같은 경고가 떠요!  
> A. gateway4 설정 방법이 변경되었습니다. gateway4를 아래와 같이 변경해 주세요.
>
> ```yaml
> routes:
>   - to: default
>     via: 192.168.100.2
> ```

### 주의점

interfaces, netplan으로 고정 IP를 설정하면 IP 충돌 위험이 있습니다. 예시로 PC1, PC2가 `192.168.100.10`로 등록하고, PC3이 `sudo arping -c 1 192.168.100.10` 명령으로 1번의 ARP 요청을 보내면 2개의 응답을 받는 것을 볼 수 있습니다.

### 해결방안

OS에서 설정하는 것이 아닌 DHCP에서 MAC 주소를 통해 설정하는 방법을 권장해 드립니다. `C:\ProgramData\VMware\vmnetdhcp.conf` 파일 하단에 다음과 같은 구성으로 고정 IP를 설정할 수 있습니다. Host PC에서 일관적으로 고정 IP를 관리하기 때문에 관리에 용이합니다.

```conf
# PC 1
host VMnet8 {
    hardware ethernet MAC1;
    fixed-address 192.168.100.10;
}

# PC 2
host VMnet8 {
    hardware ethernet MAC2;
    fixed-address 192.168.100.11;
}
```

파일 저장 후, 관리자 권한으로 실행된 명령 프롬프트를 통해 `vmnetdhcp`를 재시작하여 적용할 수 있습니다.

```cmd
net stop vmnetdhcp
net start vmnetdhcp
```
