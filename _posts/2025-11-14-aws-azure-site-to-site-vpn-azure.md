---
title: "AWS-Azure 간 Site-to-Site VPN 연결 - Azure편"
date: 2025-11-14 23:26:00 +0900
categories: [Cloud, Azure]
tags: [azure, aws, vpn, bgp, site-to-site-vpn]
description: "AWS와 Azure간 Site-to-Site VPN 연결 가이드 - Azure Side"
toc: true
---

## Introduction
현대의 IT 인프라 환경에서는 특정 클라우드 플랫폼에 워크로드를 집중시키기보다, **Azure와 AWS를 함께 활용하는 멀티클라우드 아키텍처**가 점점 더 일반화되고 있습니다. 이러한 환경에서는 서로 다른 클라우드 간 안정적이고 보안이 보장된 네트워크 연결이 필수적이며, 특히 서비스 간 프라이빗 통신을 가능하게 하는 **Site-to-Site VPN 구성**은 매우 중요한 역할을 수행합니다.

본 문서는 **Azure VNet과 AWS VPC 간 Site-to-Site VPN 연결 테스트**를 수행하고, Azure 환경에서 필요한 설정 값과 검증 절차를 정리한 자료입니다. 테스트는 IKEv2 기반 IPsec 터널을 구성하고, **BGP(APIPA)를 활용한 동적 라우팅 동작**을 확인하는 것을 주요 목표로 합니다. 이를 위해 Azure의 **VPN Gateway** 와 AWS의 **가상 프라이빗 게이트웨이(VGW)** 를 구성하여 두 개의 터널을 생성하고, 장애 상황에서도 안정적으로 트래픽이 전환될 수 있는 Active-Standby 구조를 검증하였습니다.

**참고 자료**
- <https://docs.aws.amazon.com/ko_kr/vpn/latest/s2svpn/VPC_VPN.html>
- <https://learn.microsoft.com/ko-kr/azure/vpn-gateway/vpn-gateway-howto-aws-bgp>

---

## Azure Site-to-Site VPN 구성 가이드

Azure 환경에서 온프레미스, 데이터센터, 또는 AWS와 같은 외부 클라우드 환경과 안정적으로 통신하기 위해서는 **보안이 강화된 전용 네트워크 연결 방식**이 필수적입니다.

그중에서도 **Azure Site-to-Site VPN**은 IPsec 기반 암호화 터널을 통해 Azure Virtual Network(VNet)과 외부 네트워크 간에 프라이빗 통신을 제공하는 대표적인 하이브리드 네트워크 구성 기술입니다.

본 문서는 Azure 환경에서 Site-to-Site VPN을 구성하는 전체 절차를 단계적으로 정리한 실무 가이드입니다. Azure VNet 환경을 기반으로 **VNet 생성**, **서브넷 구성**, **VPN Gateway 생성**, **Local Network Gateway 등록**, 그리고 **VPN 연결 생성 및 BGP 설정**까지 실제 운영 환경에서 요구되는 핵심 구성 요소들을 체계적으로 다룹니다.

특히 AWS Virtual Private Gateway(VGW)와의 연동을 고려하여 구성된 값들을 기반으로, **Azure BGP ASN 설정**, **Azure APIPA 기반 BGP IP 사용 방식**, **터널 단위의 독립된 연결 관리**, **Active-Standby 구조** 등 멀티클라우드 환경에서 반드시 알아야 하는 설정 포인트를 함께 정리하였습니다.

본 가이드는 Azure 환경에서 Site-to-Site VPN을 처음 구성하는 사용자뿐 아니라,

AWS–Azure 간 멀티클라우드 네트워크를 설계하는 엔지니어에게도 실무적인 참고 자료로 활용될 수 있습니다.

이어지는 장에서는 각 구성 요소의 역할과 설정 방법을 상세히 설명하며, 제공된 구성 값 표와 절차를 그대로 따라가면 Azure–AWS 간 VPN 연동 테스트 환경을 동일하게 재현할 수 있습니다.

### VNet 구성

Azure 측 네트워크는 AWS와의 Site-to-Site VPN 테스트를 위해 **기본적인 VNet + Subnet + GatewaySubnet** 정도면 충분합니다.

아래는 예시 값 기준으로 구성한 내용입니다.

#### 1-1. VNet 생성

| 항목 | 값(예시) |
| --- | --- |
| VNet 이름 | **test-vnet** |
| IPv4 주소 공간 | **10.2.0.0/16** |
| 리전 | Korea Central |

**설명**

- /16 대역을 사용하여 여러 개의 서브넷을 손쉽게 구성할 수 있습니다.
- 가장 중요한 점은 **AWS VPC와 CIDR이 절대 겹치지 않아야 한다는 점**입니다.
- Azure는 VNet 생성 시 **GatewaySubnet이 자동 생성되지 않기 때문에, 다음 단계에서 별도로 구성**해야 합니다.

| 항목 | 값(예시) |
| --- | --- |
| 구분 | VNet 기본 서브넷 |
| 서브넷 이름 | **default** |
| IPv4 CIDR | **10.2.0.0/24** |

**설명**

- VNet 생성 시 함께 구성되는 기본 Subnet입니다.
- 테스트 VM을 배치할 용도로 사용하며, 단일 서브넷만으로도 VPN 통신 검증에는 충분합니다.

#### 1-2. Gateway Subnet 생성

| 항목 | 값(예시) |
| --- | --- |
| 구분 | VPN Gateway 전용 서브넷 |
| 서브넷 이름 | **GatewaySubnet** *(예약된 이름)* |
| IPv4 CIDR | **10.2.1.0/24** |

**설명**

- Azure VPN Gateway는 반드시 **GatewaySubnet**이라는 예약된 이름의 전용 Subnet에 생성해야 합니다.
- 이 서브넷에는 VPN Gateway 외 다른 리소스는 배치할 수 없습니다.
- /24 규모면 테스트 환경에서 충분합니다.

### VPN Gateway 구성

Site-to-Site VPN 연결을 위해서는 Azure 측에 **VPN Gateway**를 생성해야 합니다.

이를 위해서는 앞서 생성한 **GatewaySubnet**이 필수이며, Gateway는 해당 서브넷에만 배치할 수 있습니다.

#### 2-1. VPN Gateway 생성

| 항목 | 값(예시) |
| --- | --- |
| 이름 | **test-vpngw** |
| 지역(Region) | **Korea Central** |
| Gateway 종류 | **VPN** |
| SKU | **VpnGw2AZ** |
| 세대 | **Generation 2** |
| 공인 IP 생성 여부 | **새 공용 IP 주소 생성(test-vpngw-ip)** |
| VNet | **test-vnet** |
| 서브넷 | **GatewaySubnet (10.2.1.0/24)** |
| Active-Active 모드 | 비활성화 |
| BGP 설정 | **사용함** |
| BGP ASN | **65515** |
| BGP APIPA 주소(Tunnel 1) | **169.254.21.2** |
| BGP APIPA 주소(Tunnel 2) | **169.254.22.2** |

**설명**

- VPNGW는 Azure–AWS Site-to-Site VPN 연결의 핵심 구성 요소로, 반드시 **GatewaySubnet**에 생성해야 합니다.
- 캡처 기준으로 지역은 **Korea Central**, SKU는 **VpnGw2AZ**, 세대는 **Generation 2**로 구성되어 있습니다.
- 공인 IP는 **VPN 게이트웨이를 생성하면서 함께 생성하며**, 이는 AWS 측에서 **고객 게이트웨이를 설정**할 때 사용됩니다.
- BGP 사용을 위해 **ASN은 65515로 설정하고, 터널별 Azure BGP APIPA 주소는 169.254.21.2 / 169.254.22.2로 구성됩니다.**

### 로컬 네트워크 게이트웨이 구성

Azure에서는 AWS 터널별로 **각각 하나의 LNG**를 생성해야 합니다.

따라서 **Tunnel 1용 LNG**, **Tunnel 2용 LNG** 총 2개가 필요합니다.

#### 3-1. Local Network Gateway 1 (Tunnel 1)

| 항목 | 값(예시) |
| --- | --- |
| 이름 | **test-lng1** |
| 지역(Region) | **Korea Central** |
| IP 주소 | **AWS Tunnel 1 Outside IP 입력** |
| 주소 공간 | **10.1.0.0/16** |
| BGP 사용 여부 | **사용함** |
| BGP ASN | **64512** *(AWS 기본 ASN)* |
| BGP 피어 IP | **169.254.21.1** |

**설명**

- LNG는 Azure 관점에서 “반대편 VPN 엔드포인트(AWS)”를 정의하는 리소스입니다.
- IP 주소는 AWS Tunnel 1의 **Outside IP** 값이며, **AWS Site-to-Site VPN 콘솔에서 ‘제너릭(Generic)’ 구성 파일을 다운로드하여 확인할 수 있습니다.**
- 주소 공간은 AWS VPC CIDR **10.1.0.0/16**을 입력합니다.
- BGP 설정을 사용하는 경우 AWS Tunnel 1의 BGP Peer IP인 **169.254.21.1**을 입력합니다. 이 값 또한 **AWS Generic 구성 파일 내에 명시되어 있습니다.**

#### 3-2. Local Network Gateway 2 (Tunnel 2)

| 항목 | 값(예시) |
| --- | --- |
| 이름 | **test-lng2** |
| 지역(Region) | **Korea Central** |
| IP 주소 | **AWS Tunnel 2 Outside IP 입력** |
| 주소 공간 | **10.1.0.0/16** |
| BGP 사용 여부 | **사용함** |
| BGP ASN | **64512** |
| BGP 피어 IP | **169.254.22.1** |

**설명**

- 두 번째 터널(Tunnel 2)을 Azure에서 정의하기 위한 LNG 리소스입니다.
- IP 주소는 AWS Tunnel 2의 **Outside IP** 이며, 마찬가지로 AWS VPN 페이지에서 **Generic 구성 파일을 다운로드하여 확인할 수 있습니다.**
- 주소 공간은 Tunnel 1과 동일하게 AWS VPC CIDR **10.1.0.0/16**입니다.
- BGP 사용 시 AWS Tunnel 2의 BGP Peer IP인 **169.254.22.1**을 입력하며, 이 값도 **Generic 구성 파일 내부에서 함께 제공됩니다.**

### VPN 연결 구성

Azure에서는 **AWS Tunnel 1용 Connection**, **AWS Tunnel 2용 Connection** 총 2개의 Site-to-Site(IPsec) 연결을 생성해야 합니다.

각 Connection은 앞서 생성한 **VPN Gateway(test-vpngw)** 와 **각각의 Local Network Gateway(test-lng1, test-lng2)** 를 연결하는 구조입니다.

#### 4-1. VPN Connection 1 (Tunnel 1)

| 항목 | 값(예시) |
| --- | --- |
| 이름 | **test-conn1** |
| 연결 유형 | **Site-to-Site(IPsec)** |
| VPN Gateway | **test-vpngw** |
| Local Network Gateway | **test-lng1** |
| 인증 방법 | **공유 키(PSK)** |
| 공유 키(PSK) | **AWS Tunnel 1 PSK 입력** |
| BGP 사용 여부 | **사용함** |
| 사용자 지정 BGP 주소 | **169.254.21.2** |

**설명**

- Connection 1은 **AWS Tunnel 1**과 매핑되는 Site-to-Site VPN 연결입니다.
- 인증 방식은 **공유 키(PSK)** 이며, AWS Generic 구성 파일에서 제공되는 **Tunnel 1 Pre-Shared Key** 값을 입력합니다.
- 사용자 지정 BGP 주소 **169.254.21.2는 Azure 측 APIPA BGP 주소이며, AWS Tunnel 1의 BGP Peer IP(169.254.21.1)와 쌍을 이루어 BGP 라우팅을 수행**합니다.

#### 4-2. VPN Connection 2 (Tunnel 2)

| 항목 | 값(예시) |
| --- | --- |
| 이름 | **test-conn2** |
| 연결 유형 | **Site-to-Site(IPsec)** |
| VPN Gateway | **test-vpngw** |
| Local Network Gateway | **test-lng2** |
| 인증 방법 | **공유 키(PSK)** |
| 공유 키(PSK) | **AWS Tunnel 2 PSK 입력** |
| BGP 사용 여부 | **사용함** |
| 사용자 지정 BGP 주소 | **169.254.22.2** |

**설명**

- Connection 2는 AWS Tunnel 2와 매핑되는 Site-to-Site VPN 연결입니다.
- PSK는 AWS Generic 구성 파일에서 제공되는 **Tunnel 2 Pre-Shared Key** 값을 입력합니다.
- 사용자 지정 BGP 주소 **169.254.22.2는 Azure 측 APIPA BGP 주소이며, AWS Tunnel 2의 BGP Peer IP(169.254.22.1)와 짝을 이루어 BGP 라우팅을 수행**합니다.

---
