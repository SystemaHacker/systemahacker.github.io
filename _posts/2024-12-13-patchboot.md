---
layout: article
title: "PatchBoot: 보안 부팅 우회"
excerpt: "PatchBoot: 보안 부팅 우회"
date: 2024-12-13 17:04 +0900
author: SystemaHacker
categories: [Anti-Cheat Bypass]
tags: [Anti-Cheat Bypass, UEFI, Security]
permalink: /anti-cheat-bypass/patchboot
---

## 개요
해당 문서는 [https://github.com/SamuelTulach/PatchBoot](https://github.com/SamuelTulach/PatchBoot) 저장소의 내용을 참고하였으며 한국어로 번역합니다.   

해당 문서에서는 [보안 부팅](https://learn.microsoft.com/ko-kr/windows-hardware/design/device-experiences/oem-secure-boot)을 우회하고 서명되지 않은 실행파일을 로드할 수 있도록 [AMI Aptio V UEFI 펌웨어](https://www.ami.com/bios-uefi-utilities/)를 패치하는 가이드를 제공합니다.   

## 주의
메인보드의 펌웨어를 수정하면 손상될 위험이 있습니다, 일부 메인보드에서는 복구 불가할 수 있습니다.   

## 원본 펌워어 다운로드
메인보드는 [MSI B360M 박격포](https://kr.msi.com/Motherboard/B360M-MORTAR/support) 제품으로 진행합니다.   
메인보드 제조사의 사이트에서 최신 펌웨어를 다운로드합니다.   
![메인보드 제조사 사이트에서 최신 펌웨어 다운로드](/assets/anti-cheat-bypass/patchboot/b360m-mortar-driver-download.png)   

## EFI 바이너리 추출하기
다운로드한 펌웨어 파일을 [UEFITool](https://github.com/LongSoft/UEFITool)을 이용해 열어줍니다. (UEFITool의 NE 버전은 모듈 교체 기능을 지원하지 않습니다. 여기서는 이를 지원하는 1.28.0 버전을 사용하겠습니다.   
![UEFITool을 이용해 다운로드한 펌웨어를 열기](/assets/anti-cheat-bypass/patchboot/uefitool-open-driver.png)   

파일 로딩이 끝나면 File > Search... 를 눌러서 "image verification"을 텍스트로 검색합니다.   
![UEFITool 텍스트 검색](/assets/anti-cheat-bypass/patchboot/uefitool-search-string.png)   

하단 검색 결과 `Unicode text "image verification" found in PE32 image section at offset 7514h`를 클릭해서 `SecurityStubDxe` 모듈로 이동합니다.
이후 "PE32 image section"을 우클릭 후 "Extract body"를 눌러 이미지를 추출합니다.   
![UEFITool 텍스트 검색 결과](/assets/anti-cheat-bypass/patchboot/uefitool-extract-body.png)   

## 이미지 검증 핸들러 찾기
이제 가장 어려운 부분입니다, 이미지 검증 역할을 하는 함수를 찾기 위해 여기서는 [Ghidra](https://github.com/NationalSecurityAgency/ghidra)를 사용하겠습니다.   
Ghidra에서 이미지를 열고 Search > Memory를 클릭 후 Search Value에 0x800000000000001를 입력하고 Hex로 검색합니다.   
![Ghidra 함수 찾기](/assets/anti-cheat-bypass/patchboot/ghidra-search-value.png)   

여러 개의 결과 중 짧은 함수를 제외한 긴 함수 하나가 이미지 검증 역할을 담당합니다.   
직접 하나하나 확인해서 찾아야 합니다. 필자는 미리 찾은 함수 이름을 "ImageVerificationHandler"로 변경해 놓았습니다.   
이렇게 찾은 함수의 오프셋을 기억해 놓습니다. 필자의 경우 `15DC`에 위치해있습니다.   
![Ghidra 함수 찾기](/assets/anti-cheat-bypass/patchboot/ghidra-find-function.png)   


## 이미지 검증 함수 패치하기
패치 과정은 쉽습니다. 검증 함수가 EFI_SUCCESS(0)를 반환하게 패치하기위해 필요한 어셈블리는 다음 사진과 같습니다.   
이를 확인하기 위해 온라인 어셈블러 사이트를 이용하였습니다.   
[https://defuse.ca/online-x86-assembler.htm#disassembly](https://defuse.ca/online-x86-assembler.htm#disassembly)   
![패치에 필요한 어셈블리](/assets/anti-cheat-bypass/patchboot/assembly.png)   

준비된 어셈블리로 패치하기위해 헥스 에디터를 이용해 이미지를 열어줍니다. 여기에서는 [HxD](https://mh-nexus.de/en/hxd/)를 이용합니다.   
HxD에서 이미지를 열고 Search > Go to...를 클릭하여 패치할 함수 오프셋인 `15DC`를 입력하고 이동합니다.   
![HxD 함수로 이동](/assets/anti-cheat-bypass/patchboot/hxd-goto-function.png)   

이렇게 찾은 함수의 첫 4바이트를 준비된 값으로 수정합니다.   
![HxD 함수 패치](/assets/anti-cheat-bypass/patchboot/hxd-patch-function.png)   
패치를 완료했으면 파일을 저장하고 닫아도 좋습니다.   


## 펌웨어 리빌딩
다시 UEFITool을 열고 펌웨어 파일을 불러옵니다, 아까와 같이 `SecurityStubDxe` 모듈을 찾고 "PE32 image section"을 우클릭 후 "Replace body..."를 누른 후 패치된 모듈로 수정합니다.   
수정이 완료되면 저장하고 UEFITool을 닫아도 좋습니다.   
![UEFITool 모듈 교체](/assets/anti-cheat-bypass/patchboot/uefitool-replace-body.png)   


## 펌웨어 플래싱
MSI B360M 박격포 모델은 [USB BIOS FlashBack](https://www.asus.com/us/support/faq/1038568/) 기능을 지원하지 않으므로 BIOS 설정에서의 펌웨어 업그레이드 기능을 이용하여 수정한 펌웨어를 플래싱합니다. (수정된 펌웨어의 파일 이름은 원본과 동일해야 합니다.)   
저와 같은 경우가 아니라면 USB BIOS FlashBack 기능을 이용하여 플래싱하는 것을 추천드립니다.   


## 탐지 가능성
제가 가장 좋아하는 부분입니다. OS가 로드된 후 DXE 드라이버는 이미 메모리에서 사라진 상태이므로 안티치트가 EFI 어플리케이션을 이용하여 패치나 서명되지 않은 드라이버를 확인하지 않는 한 탐지로부터 안전합니다.   
