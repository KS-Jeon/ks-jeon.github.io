---
title: "AWS-Azure 간 Site-to-Site VPN 연결 - AWS편"
date: 2025-11-14
categories: [Cloud, AWS]
tags: [aws, azure, vpn, bgp, site-to-site-vpn]
description: "AWS와 Azure간 Site-to-Site VPN 연결 가이드 - AWS Side"
toc: true
---

## Introduction

현대의 IT 인프라 환경에서는 단일 클라우드에 모든 워크로드를 위치시키기보다, **AWS와 Azure를 병행 활용하는 멀티클라우드 아키텍처**가 보편화되고 있다. 이러한 구조에서는 두 클라우드 간 안정적이고 보안이 보장된 네트워크 연결이 필수적이며, 특히 서비스 간 프라이빗 통신을 가능하게 하는 **Site-to-Site VPN 구성**이 중요한 역할을 수행한다.

본 문서는 **AWS VPC와 Azure VNet 간 Site-to-Site VPN 연결 테스트**를 수행하고, 해당 구성에 대한 설정 값과 검증 절차를 정리한 자료이다. 테스트는 IKEv2 기반 IPsec 터널을 구성하고, **BGP(APIPA)를 활용한 동적 라우팅 확인**을 주요 목표로 한다. 이를 위해 AWS의 **가상 프라이빗 게이트웨이** 와 Azure의 **VPN Gateway** 를 사용하여 두 터널을 생성하고, Failover 시나리오를 고려한 Active-Standby 형태의 구조를 검증하였다.

**참고 자료**
- <https://docs.aws.amazon.com/ko_kr/vpn/latest/s2svpn/VPC_VPN.html>
- <https://learn.microsoft.com/ko-kr/azure/vpn-gateway/vpn-gateway-howto-aws-bgp>

---

## AWS Site-to-Site VPN 구성 가이드

AWS 환경에서 온프레미스, 데이터센터, 또는 Azure와 같은 외부 클라우드 환경과 안전하게 통신하기 위해서는 안정적이고 보안이 강화된 네트워크 연결 방식이 필요하다.

그중에서도 **AWS Site-to-Site VPN**은 IPsec 기반 암호화 터널을 사용하여 프라이빗 네트워크 간 통신을 가능하게 하는 대표적인 하이브리드 네트워크 구성 방식이다.

본 문서는 AWS에서 Site-to-Site VPN을 구성하는 전체 절차를 단계적으로 정리한 가이드이다.

AWS VPC 환경을 기반으로 **VPC 생성**, **서브넷 구성**, **라우팅 테이블 설정**, **가상 프라이빗 게이트웨이 생성**, **고객 게이트웨이 구성**, 그리고 **VPN 터널 생성 및 BGP 설정**에 이르기까지 실무에서 필요한 구성 요소를 체계적으로 설명한다.

특히 Azure VPN Gateway와의 연동을 염두에 두고 구성된 설정값을 기반으로, **BGP ASN 설정**, **APIPA 기반 BGP IP 사용 방식**, **터널 Active-Standby 구조** **등** 실제 멀티클라우드 환경에서 활용되는 구성 포인트를 함께 다룬다.

본 가이드는 AWS 환경에서 Site-to-Site VPN 연결을 처음 구성하는 사용자부터, 멀티클라우드 네트워크 구축을 준비하는 엔지니어까지 실무적으로 참고할 수 있는 문서로 활용될 수 있다. 이어지는 장에서는 각 컴포넌트의 역할과 구성 방법을 상세히 설명하며, 설정 값과 구성 흐름을 따라가면 AWS-Azure 간 VPN 연동 테스트 환경을 동일하게 재현할 수 있다.

### VPC 구성

Site-to-Site VPN 테스트를 위한 AWS 측 네트워크는 **기본적인 VPC + 서브넷 + 라우팅 테이블** 정도면 충분하다.

아래 예시 값 기준으로 구성한다.

#### 1-1. VPC 생성

| 항목 | 값(예시) |
| --- | --- |
| 이름 | test-vpc |
| IPv4 CIDR | 10.1.0.0/16 |

**설명**

- 멀티 서브넷 구성이 가능하도록 /16 대역으로 설정한다.
- 중요한 것은 **Azure VNet과 CIDR이 겹치지 않는 것**이다.

#### 1-2. 서브넷 생성

| 항목 | 값(예시) |
| --- | --- |
| 서브넷 이름 | test-subnet |
| 가용 영역 | ap-northeast-2a |
| IPv4 CIDR | 10.1.0.0/24 |

**설명**

- VPN 연결 후 테스트용 EC2 인스턴스를 배치할 서브넷이다.
- 단일 AZ, 단일 서브넷만 있어도 VPN 통신 테스트에는 충분하다.

#### 1-3. 라우팅 테이블 구성

| 항목 | 값(예시) |
| --- | --- |
| 라우팅 테이블 이름 | test-rt |
| 연결 VPC | test-vpc |
| 연결 서브넷 | test-subnet |
| 라우팅 전파 | 가상 프라이빗 게이트웨이(VGW) 생성 후 **전파 활성화** |

**설명**

- 처음에는 기본 `local` 라우트(10.1.0.0/16)만 존재한다.
- 이후 가상 프라이빗 게이트웨이를 생성하고 VPC에 연결한 뒤, 해당 라우팅 테이블에서 **라우팅 전파를 활성화하면 BGP로 학습한 Azure 측 CIDR 경로**가 자동으로 추가된다.

### 가상 프라이빗 게이트웨이 구성

AWS에서 Site-to-Site VPN을 구성하기 위해서는 **VPC 경계에 해당하는 가상 프라이빗 게이트웨이(Virtual Private Gateway)** 를 먼저 생성해야 한다. VGW는 AWS 측에서 VPN 터널을 종단하는 역할을 하며, 이후 고객 게이트웨이(Customer Gateway)와 함께 실제 VPN 연결을 구성하게 된다.

#### 2-1. 가상 프라이빗 게이트웨이 생성

| 항목 | 값(예시) |
| --- | --- |
| 이름 | test-vgw |
| ASN | Amazon 기본 ASN (64512) |
| 연결 VPC | test-vpc |

**설명**

- AWS VGW는 별도의 Public IP를 사용하지 않는다.
- VPN 터널은 VGW ↔ Azure VPN Gateway 간에 자동으로 형성된다.
- Amazon 기본 ASN(64512)을 그대로 사용해도 되고, 필요 시 커스텀 ASN을 부여할 수도 있다.

#### 2-2. VPC에 VGW 연결(Attach)

VGW는 생성만 해서는 동작하지 않으며, 반드시 **특정 VPC에 Attach** 해야 라우팅이 가능해진다.

연결 절차는 아래와 같다.

1. AWS 콘솔에서 **VPC 서비스 → 가상 프라이빗 게이트웨이** 메뉴 이동
2. 방금 생성한 **test-vgw** 선택
3. 상단 메뉴에서 **"작업" → "VPC 연결"** 클릭
4. 팝업 창에서 아래와 같이 선택 후 Attach

| 항목 | 선택 값 |
| --- | --- |
| VPC | test-vpc |

VGW를 test-vpc에 연결하면 해당 VPC는 외부 네트워크와 통신할 수 있는 경계 지점을 갖추게 된다.

이 지점을 통해 AWS와 Azure 사이의 VPN 터널이 형성되며, 이후 BGP 기반 경로 학습과 VPN 라우팅이 모두 이 VGW를 중심으로 동작하게 된다.

또한 VGW가 VPC에 연결된 시점부터, **해당 VPC 라우팅 테이블에서는 라우팅 전파 기능을 활성화할 수 있게 되어 Azure에서 전달하는 네트워크 경로를 자동으로 수신하고 적용**할 수 있다.

#### 2-3. 라우팅 테이블에서 라우팅 전파 활성화

VGW를 VPC에 Attach한 후, 해당 VPC에 연결된 **라우팅 테이블로 이동하여 라우팅 전파를 활성화**한다.

라우팅 전파(Route Propagation)는 Azure 측에서 BGP를 통해 광고하는 VNet CIDR 정보(예: 10.2.0.0/16)를 AWS가 자동으로 학습하기 위해 반드시 활성화해야 하는 기능이다.

이 설정이 비활성화된 상태에서는 **Azure에서 전달되는 경로 정보가 AWS 라우팅 테이블에 반영되지 않기 때문에**, AWS의 EC2 인스턴스에서 Azure로 향하는 트래픽이 적절한 목적지 경로를 찾지 못해 통신이 실패하게 된다.

### 고객 게이트웨이 구성

고객 게이트웨이(Customer Gateway)는 **AWS가 외부 네트워크(Azure)의 라우터 정보를 인식하기 위해 필요한 객체**이다.

Azure VPN Gateway의 Public IP와 Azure 측 BGP ASN 정보를 AWS에 알려주는 역할을 하며, 이후 VPN 터널과 BGP 세션이 이 정보를 기반으로 생성된다.

#### 3-1. 고객 게이트웨이 생성

CGW 생성 시 필요한 핵심 정보는 다음 두 가지이다.

- **Azure VPN Gateway Public IP**
- **Azure BGP ASN(기본값: 65515)**

생성 화면에 입력하는 값은 아래 표와 주의 사항을 참고한다.

| 항목 | 값(예시) |
| --- | --- |
| 이름 | test-cgw |
| 라우팅 옵션 | 동적(BGP) |
| BGP ASN | 65515 (Azure 기본 ASN) |
| IP 주소 | Azure VPN Gateway Public IP |
| 인증서 ARN | (사용 안 함) |

**주의사항**

- **라우팅 옵션은 반드시 ‘동적(BGP)’을 선택해야 한다.** 정적 라우팅으로 생성하면 Azure와의 BGP 연동이 불가능하다.
- **BGP ASN은 Azure 게이트웨이가 사용하는 ASN과 동일해야 한다.** Azure 기본 ASN은 65515이므로 대부분 그대로 사용한다.
- **IP 주소는 Azure VPN Gateway의 Public IP를 그대로 입력한다.** 이는 VPN 터널의 상대 종단으로 사용된다.

#### 3-2. 고객 게이트웨이의 역할

CGW는 물리적인 장비가 아닌 AWS 내부에서 사용하는 **논리적 객체**이다.

그러나 실제로는 아래와 같은 중요한 역할을 수행한다.

| 역할 구분 | 설명 | 결과/영향 |
| --- | --- | --- |
| **Azure VPN Gateway의 정체성 정의** | AWS에 **Azure 게이트웨이가 어떤 Public IP와 어떤 BGP ASN을 사용하는지** 알려주는 단계 | 외부 게이트웨이를 식별 가능해지고, AWS는 해당 정보를 기준으로 터널 구성을 진행함 |
| **VPN 터널 외부 종단 정보 제공** | AWS가 IPsec 터널을 생성할 때 필요한 Azure 쪽 Public IP, 라우팅 방식(BGP) 정보를 제공 | AWS는 CGW 정보를 기반으로 **VPN 터널 설정(IPsec 파라미터, BGP Peer 구성)** 을 자동 생성함 |
| **BGP Neighbor 구성 정보 제공** | Azure BGP ASN을 입력하면, 해당 ASN이 AWS 터널의 BGP Peer 설정에 반영됨 | Azure–AWS 간 BGP 세션이 정상 수립되며, APIPA 주소 기반 BGP Neighbor가 구성됨 |

#### 3-3. 생성 후 상태

CGW는 생성만 하면 “available(사용 가능)” 상태가 되며, 아직 AWS와 Azure 사이에 VPN 터널은 생성되지 않는다.

VPN 터널은 다음 단계인 **Site-to-Site VPN 생성 단계**에서 VGW + CGW를 조합하여 구성한다.

### Site-to-Site VPN 연결 구성

Site-to-Site VPN 연결 단계에서는 **AWS VGW** 와 **Azure 고객 게이트웨이(CGW)** 를 실제 VPN 터널로 연결한다.

이 과정에서 **2개의 터널**이 자동으로 생성되며, 각 터널에는 **APIPA 기반 BGP IP** 와 **PSK(Pre-Shared Key)** 가 할당된다.

이 단계가 AWS–Azure간 Site-to-Site 통신의 핵심이며, BGP 세션 구성 및 경로 학습도 이 VPN 연결을 통해 이루어진다.

#### 4-1. VPN 연결 생성

| 항목 | 값(예시) |
| --- | --- |
| 이름 | test-connect |
| 가상 프라이빗 게이트웨이 | test-vgw |
| 고객 게이트웨이 | test-cgw |
| 라우팅 옵션 | 동적(BGP 필요) |
| PSK 스토리지 | 표준(Standard) |

#### 4-2. 터널 1 구성 정보

| 항목 | 값 |
| --- | --- |
| 내부 IPv4 CIDR (APIPA) | **169.254.21.0/30** |
| PSK | AWS에서 자동 생성 |
| 고급 옵션 | 기본 옵션 사용 |

터널 1의 CIDR /30 대역은 AWS가 자동으로 생성한 Azure–AWS 간 BGP 링크 주소임

- AWS BGP Peer: **169.254.21.1**
- Azure BGP Peer: **169.254.21.2**

#### 4-3. 터널 2 구성 정보

| 항목 | 값 |
| --- | --- |
| 내부 IPv4 CIDR (APIPA) | **169.254.22.0/30** |
| PSK | AWS에서 자동 생성 |
| 고급 옵션 | 기본 옵션 사용 |

터널 2 역시 동일 구조이며 다음과 같은 BGP Peer가 사용된다.

- AWS BGP Peer: **169.254.22.1**
- Azure BGP Peer: **169.254.22.2**

#### 4-4. 구성 설명

- **라우팅 옵션은 동적(BGP)** 을 반드시 선택해야 Azure–AWS 간 BGP 피어링이 활성화된다.
- 두 개의 터널이 생성되지만, AWS VPN의 기본 작동 방식은 **터널 1 Active / 터널 2 Standby 구조**이다.
- PSK는 AWS에서 자동 생성되며, Azure Portal에서 동일 키를 터널 별로 입력해야 한다.
- 내부 IPv4 CIDR(APIPA)은 양측 터널의 BGP Peer IP가 포함된 169.254.x.x/30 주소 대역으로 자동 생성된다.

#### 4-5. 생성 후 확인 사항

- 터널 상태가 **UP(연결됨)** 으로 표시되는지 확인
- BGP 상태가 **Established** 로 올라오는지 확인
- Route Table에서 Azure CIDR이 **BGP Learned Route** 로 자동 등록되는지 확인

---
