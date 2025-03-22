---
layout: article
title: "Proxmox Datacenter Manager 설치"
excerpt: "이 문서에서는 여러 노드 및 클러스터들을 관리하기위한 Proxmox Datacenter Manager를 설치하는 방법에 대해 설명합니다."
date: 2025-03-23 01:37 +0900
author: SystemaHacker
categories: [Proxmox]
tags: [Proxmox, Virtualization]
permalink: /proxmox/install-proxmox-datacenter-manager
---

## 개요
이 문서는 여러 노드 및 클러스터들을 관리하기 위해 `Proxmox Datacenter Manager`를 설치하는 방법에 대해 설명합니다.

---
## Proxmox Datacenter Manager 설치
`Proxmox Datacenter Manager`를 설치하기 위해 [Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/scripts?id=proxmox-datacenter-manager)에서 설치 스크립트를 복사 후 설치를 원하는 노드의 shell에 붙여 넣습니다.

``` bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/proxmox-datacenter-manager.sh)"
```

![Paste install command in PVE console](/assets/proxmox/install-proxmox-datacenter-manager/pve-console-paste-install-command.png)

   
설치를 진행하는 동안 몇 가지 설정들이 나타나는데 여기서는 기본 설정으로 선택하겠습니다.

![installation process](/assets/proxmox/install-proxmox-datacenter-manager/pve-console-install-process-1.png)

![installation process default settings](/assets/proxmox/install-proxmox-datacenter-manager/pve-console-install-process-2.png)

다음과 같은 화면이 나타나면 설치가 완료된 것입니다.

![install complete](/assets/proxmox/install-proxmox-datacenter-manager/pve-console-install-complete.png)

---
## 설정 
설치가 완료된 `Proxmox Datacenter Manager` lxc의 shell을 열고 `root` 비밀번호를 변경합니다.
```bash
passwd root
```   
![change root password](/assets/proxmox/install-proxmox-datacenter-manager/proxmox-datacenter-manager-change-root-password.png)   

기본 네트워크 설정인 `DHCP`가 아닌 `고정 IP 주소`를 사용하려면 `Proxmox Datacenter Manager` lxc의 네트워크를 설정합니다.

![configure lxc network](/assets/proxmox/install-proxmox-datacenter-manager/lxc-network-settings.png)   

---
## 노드 추가하기
`Proxmox Datacenter Manager`의 URL로 접속하면 다음과 같은 화면이 나타납니다. `root`로 로그인합니다.

![Login to Proxmox Datacenter Manager](/assets/proxmox/install-proxmox-datacenter-manager/proxmox-datacenter-manager-login.png)

관리할 노드를 추가하기 위해 `Remotes` 탭의 `Add Proxmox VE`를 눌러 다음과 같이 `서버 주소`와 `핑거프린트`를 입력하고 `Connect` 후 다음으로 넘어갑니다.   
`핑거프린트`는 추가할 `Proxmox VE`에서 노드를 선택한 후 `Certificates` 탭에서 `pve-ssl.pem`을 눌러서 확인할 수 있습니다.

![Add Proxmox VE into Proxmox Datacenter Manager](/assets/proxmox/install-proxmox-datacenter-manager/proxmox-datacenter-manager-add-remote-1.png)

![Check Proxmox VE fingerprint](/assets/proxmox/install-proxmox-datacenter-manager/pve-node-check-certificate.png)

다음 화면에서 `Remote ID`에 표시할 노드의 이름과 계정 정보를 입력 후 `Scan` 합니다.   
Proxmox VE 2FA는 지원하지 않음으로 해제하고 진행합니다.

![Add Proxmox VE into Proxmox Datacenter Manager](/assets/proxmox/install-proxmox-datacenter-manager/proxmox-datacenter-manager-add-remote-2.png)   

다음 사진과 같이 `엔드포인트`를 설정합니다.

![Add Proxmox VE into Proxmox Datacenter Manager 3](/assets/proxmox/install-proxmox-datacenter-manager/proxmox-datacenter-manager-add-remote-3.png)

이제 추가된 노드 및 클러스터를 `Proxmox Datacenter Manager`에서 관리할 수 있습니다.

![Add Proxmox VE into Proxmox Datacenter Manager complete](/assets/proxmox/install-proxmox-datacenter-manager/proxmox-datacenter-manager-add-complete.png)

