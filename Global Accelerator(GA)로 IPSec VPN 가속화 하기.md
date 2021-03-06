
# Global Accelerator(GA)로 IPSec VPN 가속화 하기

## 1. 배경  설명 
한국과 중국간 인터넷 통신은 매우 불안정합니다. 아래의 베이징(BJ)-서울(KR), 상하이(SH)-서울(KR) 간 ping test 결과를 살펴보면 평균 응답시간이 불규칙하고 ping loss 도 많이 발생하는 것을 알 수 있습니다. 

![](https://github.com/rnlduaeo/alibaba/blob/master/pingtime.png?raw=true)

이는 중국에서 비즈니스를 하는 대다수 고객들에게 많은 불편함을 야기합니다. 가령, 한국 본사와 중국 지사간 IPSec VPN으로 Site-to-Site VPN 통신을 맺고 시스템간 동기화 등의 연동을 맺고 있는 경우, 이 ping loss와 높은 지연시간은 시스템의 장애를 초래할 수 있는 매우 중대한 사안입니다. 
이번 포스팅에서는 알리바바 클라우드의 Global Accelerator(GA)를 통해 한국/중국 간 IPSec VPN 통신에 대한 네트워크의 안정성을 높이고 속도도 가속화하는 방법에 대해 다루어 보겠습니다. 

## 2. Solution Overview
### 2.1 Overview
이번 가이드에서는 중국 Alibaba Cloud의 VPN Gateway와 한국 AWS의 Virtual Private Gateway를 GA와 연동합니다. 이는 테스트를 위한 차선책으로 굳이 알리바바 클라우드의 VPN Gateway를 사용하지 않더라도 'NAT-T(Nat Traversal)'기능을 enable할 수 있는 고객사 VPN 장비라면 GA와 연동하여 가속화할 수 있습니다.
> Note: GA가 양단의 VPN 입장에서는 일종의 NAT 역할을 하는 장비가 되기 때문에, VPN장비 사이에 NAT 장비가 존재하게 되는 셈이고 이로 인해 Pair 메세지의 무결성이 침해되어 IKE Phase 1,2 협상 과정이 실패하게 됩니다. NAT-T 기능을 제공하는 VPN 장비로 이 문제를 해결할 수 있습니다. 

### 2.2 가속화 원리 
![](https://github.com/rnlduaeo/alibaba/blob/master/VPNoverGA.png?raw=true)

알리바바 클라우드의 [Global Accelerator(GA)](https://www.alibabacloud.com/help/doc-detail/153189.htm?spm=a2c63.l28256.b99.5.82586796Hc8DP7)
는 사용자 시스템의 IP/Domain만 등록하여 네트워크 통신을 가속화하는 솔루션입니다. 이번 가이드에서는 중국 상해에 고객사의 중국지사 VPN장비가 있고(Alibaba Cloud VPN Gateway로 대체) 한국에 본사 VPN 장비가 있다고(AWS Virtual Private Gateway로 대체) 가정하고 테스트를 수행합니다. 

## 3. 사전  조건 
1. [Alibaba Cloud VPN Gateway](https://www.alibabacloud.com/help/doc-detail/64960.htm?spm=a2c63.l28256.b99.5.5d6ae889hLNiHt) - 고객사 VPN 장비(NAT-T enabled)로 대체 가능
2. [AWS Virtual Private Gateway](https://docs.aws.amazon.com/ko_kr/vpn/latest/s2svpn/how_it_works.html) - 고객사 VPN 장비(NAT-T enabled)로 대체 가능
3. [Alibaba Cloud Global Accelerator](https://www.alibabacloud.com/help/doc-detail/153189.htm?spm=a2c63.l28256.b99.5.82586796Hc8DP7) 
	 
	* GA 인스턴스를 만든 후 티켓을 통해 GA Instance ID로 'Source-Consistent' 기능 enable할 것을 요청합니다. 	
	* Listener 설정이 끝난 후 티켓을 통해 GA OFF IP를 획득합니다.(이 EIP는 한국 VPN 장비의 Peer IP로 사용됨) - GA 내부적으로 로드발란서의 가중치(weight)를 0으로 수정 (1 개의 ECS 만 포워딩 용으로 예약)하고 나머지 ECS의 EIP를 획득하는 과정입니다. 

## 4. Main steps
각 단계의 큰 흐름은 아래와 같습니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Picture1.png?raw=true)
### 4.1 GA 인스턴스 생성과 Source Consistent 요청(Ticket)
#### 4.1.1 GA 인스턴스 및 밴드위스 생성
GA는 가속화 요건에 따라 다양한 조합의 구매가 가능합니다. 이번 시나리오는 중국과 한국간 네트워크를 가속화 하고, 양 종단이 Alibaba Cloud 외 리소스인 조건이므로 아래의 조합으로 구매해 주시면 됩니다. 
> Note: [GA 인스턴스 및 밴드위스 구매 조건](https://www.alibabacloud.com/help/doc-detail/153194.htm?spm=a2c63.p38356.b99.10.571c2e24WeD1J9)에 대한 자세한 사항은 클릭하여 확인해 주시기 바랍니다. 

|GA Instance Type|Basic Bandwidth Type|Cross Border Acceleration |
|---|---|---|
|Small I(필요한 최대 밴드위스에 따라 구매)|Enhanced Bandwidth|구매|

[GA 인스턴스를 구매](https://www.alibabacloud.com/help/doc-detail/153200.htm?spm=a2c63.p38356.b99.22.3e8d3ec5YMYcrz) 하고 [Basic Bandwidth를 구매](https://www.alibabacloud.com/help/doc-detail/153205.htm?spm=a2c63.p38356.b99.27.111077493Kolwl)하고 [Cross Border Acceleration을 구매](https://www.alibabacloud.com/help/doc-detail/155107.htm?spm=a2c63.p38356.b99.35.4f37763eng34lg)합니다. 
그리고 구매한 Basic Bandwidth와 Cross Bandwidth를 GA 인스턴스에 bind합니다. 바인딩에 대한 자세한 사항은 [bind basic bandwidth 문서](https://www.alibabacloud.com/help/doc-detail/153206.htm?spm=a2c63.p38356.b99.28.34528816XU1IGd)와 [bind cross border acceleration 문서](https://www.alibabacloud.com/help/doc-detail/155108.htm?spm=a2c63.p38356.b99.36.4095289crKAgox)를 참조해주시기 바랍니다.


#### 4.1.2 Source Consistent 요청
Ticket을 통해 GA Instance ID를 전달하며 'Source Consistent' 작업을 요청합니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%2011.20.04%20AM.png?raw=true)

### 4.2 GA 가속화 리전 생성
가속화 리전은 상하이로 선택하여 생성합니다. [자세한 가이드는 클릭](https://www.alibabacloud.com/help/doc-detail/153212.htm?spm=a2c63.l28256.b99.43.418e6796yJw0kV)하여 확인해 주시기 바랍니다. 생성한 후 획득한 Accelerated IP Address를 복사(상하이 VPN Gateway에서 peer IP로 사용 예정)해 둡니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%203.39.11%20PM.png?raw=true)

### [Optional] 4.3 종단 간 VPN Gateway 생성 (기존 장비가 있다면 생략 가능)
#### 4.3.1 [Alibaba Cloud VPN Gateway를 생성](https://www.alibabacloud.com/help/doc-detail/65290.htm?spm=a2c63.l28256.b99.23.d9a5e889eDXe0e)합니다.
#### 4.3.2 [AWS Virtual Private Gateway를 생성](https://docs.aws.amazon.com/ko_kr/vpn/latest/s2svpn/SetUpVPNConnections.html#vpn-create-target-gateway) 하고 VPC에 attach 합니다.
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%203.50.49%20PM.png?raw=true)

### 4.4 GA Listener 등록 및 GA OFF IP 신청
#### 4.4.1 GA Listener 등록
GA에서 Listener를 추가합니다. 이 리스너에는 VPN장비의 NAT-T에서 사용하는 프로토콜과 포트인 UDP 4500, 500번 포트를 등록 합니다. 이번테스트에서 사용된 AWS Virtual Private Gateway는 4500번 포트를 사용하기에 4500번만 등록하였습니다.
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%203.58.51%20PM.png?raw=true)

Endpoint Group 등록 시 Backend Service에는 한국 VPN 장비의 Public IP(Serial IP)를 등록합니다. 
> Note: 이번 테스트에서 사용된 AWS VPN의 경우는 Customer Gateway를 등록하고 Site-to-Site VPN connection을 생성해야 tunnel IP(VPN Public IP)가 나오기에 이번 단계에서는 임시 public IP를 등록하고 후속 단계를 거친 뒤 listener를 수정하는 방법으로 진행하겠습니다. 

![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%203.59.12%20PM.png?raw=true)

[GA 리스너 등록에 대한 자세한 사항](https://www.alibabacloud.com/help/doc-detail/153217.htm?spm=a2c63.l28256.b99.48.68606796n12ytp)은 클릭해서 확인해 주세요.

#### 4.4.2 GA OFF IP 신청
리스너를 등록하고 난 뒤 티켓을 통해 GA instance-id을 전달하며 GA OFF IP를 신청합니다. GA OFF IP를 전달 받으면 메모해 두고 추 후 AWS Customer Gateway IP로 사용합니다. 
> Note: 이 단계는 반드시 리스너 등록 후 수행해야 유효한 IP를 전달 받을 수 있습니다. 또한 리스너를 삭제하고 재설정하는 경우 GA OFF IP는 변하기 때문에 재신청해야 합니다. 리스너 수정시에는 GA OFF IP가 유지되기에 재신청할 필요가 없습니다. 

### 4.5 양 종단의 Customer Gateway 등록
고객사 장비를 연결한다면 각 장비에서 Remote Peer IP를 아래와 동일한 IP로 등록해 주면 됩니다.

#### 4.5.1 Alibaba Cloud VPN Gateway에서 [Customer Gateway 등록](https://www.alibabacloud.com/help/doc-detail/65286.htm?spm=a2c63.l28256.b99.30.3067e889Usd0nF)
Customer Gateway를 생성하고 GA에서 획득한 상하이 Accelerated IP를 등록합니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%204.19.50%20PM.png?raw=true)

#### 4.5.2 AWS VPN Gateway에서 [Customer Gateway 등록](https://docs.aws.amazon.com/ko_kr/vpn/latest/s2svpn/SetUpVPNConnections.html#vpn-create-cgw)
AWS에서 Customer Gateway를 생성하고 IP Address에는 GA OFF IP를 등록합니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%204.20.40%20PM.png?raw=true)


### [Optional] 4.6 AWS Site-to-Site VPN Connections 생성
> Note: 고객사 장비로 설정한다면 4.6 ~ 4.8 단계는 각 고객사 장비에 맞게 설정해 주시면 됩니다. 

#### 4.6.1 Site-to-Site VPN Connection을 생성합니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%204.55.21%20PM.png?raw=true)
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%204.55.33%20PM.png?raw=true)

#### 4.6.2 Route Tables에서 vgw를 propagate 합니다.
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%205.21.07%20PM.png?raw=true)


### [Optional] 4.7 Download Configuration from AWS VPN Connections
고객사 장비에 맞게 configuration file을 다운로드 합니다. 이번 테스트에서는 Cisco system 장비로 다운로드 받아서 설정 값만 참조하겠습니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%205.05.46%20PM.png?raw=true)

### [Optional] 4.8 Alibaba IPSec Connection 생성
#### 4.8.1 IPSec Connection 생성
AWS에서 다운로드 받은 configuration file의 정보를 사용하여 IPSec Connection을 생성합니다. 
> Note: AWS와 Alibaba가 정의하는 Local Network와 Remote Network의 정의가 달라 헷갈릴 수 있습니다. 공식 문서에 나와있는 설명을 토대로 작성하시기 바랍니다. 

![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%205.08.58%20PM.png?raw=true)

local ID와 remote ID를 무시할 수 있는 ikev2로 설정합니다. IKE configuration과 IPSec configuration 정보는 AWS configuration file 내용과 동일한 값으로 설정해야 합니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%205.09.07%20PM.png?raw=true)

VPN Gateway는 기본적으로 NAT Traversal이 enable되어 있습니다. 이 기능은 GA와의 연동에 가장 중요한 기능입니다. 그대로 설정합니다. 이번 테스트에는 BGP가 아닌 Static으로 라우팅을 설정했기 때문에 BGP 설정은 건너뛰겠습니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%205.09.22%20PM.png?raw=true)

#### 4.8.2 Add route entry
IPSec Connection 설정 시 해당 연결을 라우팅에 publish할 지에 대해 팝업창이 뜨는데 'Ok'를 누르면 자동으로 라우팅 테이블에 적용됩니다. 만약 적용이 안되었다면 VPN gateway 를 클릭, Policy-based Routing으로 들어가서 방금 설정한 IPSec Connection을 라우팅에 추가해 줍니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%205.19.09%20PM.png?raw=true)



### 4.9 AWS tunnel IP를 GA Backend Service IP로 등록
AWS VPN connection에서 받은 configuration file을 보면 AWS에서 생성된 tunnel IP 2개를 확인할 수 있습니다. 물론, 콘솔에서도 확인할 수 있습니다. 
AWS에서는 기본적으로 2개의 tunnel IP를 생성합니다. 이번 테스트에는 이 중 하나의 tunnel IP로 하나의 IPSec connection만 구성하도록 하겠습니다. 운영환경에서는 2개의 connection을 구성하여 가용성을 높이는 아키텍쳐로 구성하는 것을 권장드립니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%206.44.09%20PM.png?raw=true)

GA 콘솔로 가서 [Edit Endpoint Group]을 클릭하여 Backend Service의 IP를 위에서 획득한 AWS Tunnel 1 IP로 변경합니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%203.59.12%20PM.png?raw=true)

### 4.10 IPSec Connection 상태 확인
설정이 제대로 되었다면 아래와 같이 Alibaba Cloud (상하이) 와 AWS(서울) 두개의 콘솔에서 모두 IPSec Connection이 정상적으로 연결됨이 확인 될 것입니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%207.06.50%20PM.png?raw=true)

![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-07-02%20at%206.44.09%20PM.png?raw=true)

>Note: 만약 Connection이 실패되었다면 [트러블슈팅 가이드](https://www.alibabacloud.com/help/doc-detail/65802.htm?spm=a2c63.l28256.b99.144.123ce889HXDiAH)를 참조하시기 바랍니다. 금번 테스트 도중 connection은 제대로 구성되었지만 ping이 도달하지 못하는 이슈가 있었는데 그 이유는 AWS VPC CIDR(172.32.0.0/16)을 알리바바 클라우드의 default system에서 Public IP 대역으로 인식했기 때문이었습니다. 이런 경우 Ticket을 통해 account uid, vpc-id를 전달하여 해당 IP대역을 Private으로 인식할 수 있도록 whitelist 해주어야 합니다. 

## 5. 결과 
### 5-1. 성능 개선 확인
아래는 48시간 동안 mtr report(1초에 1번씩 17280번의 ping 수행)를 걸어놓고 측정한 결과입니다. 인터넷을 통한 VPN은 ping loss가 50%에 육박하며 이는 총 패킷의 절반 가량이 loss되어 타켓 서버에 도달되지 못한 것을 의미합니다. 또한 지연시간 측면에서도 약 2배 가량 개선된 것을 확인할 수 있습니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/compasion2.png?raw=true)

<!--stackedit_data:
eyJoaXN0b3J5IjpbNzM2ODQ5NjUwLC0xNTI1MzA5MzczLC0xMD
czOTE1MzIsMzAxNDc3OTg5LDE3NDcwNTE4NzgsLTI1NjYzNDA2
Miw2NjUxNzI0NjUsMzYxMjUxMjA5LC0xODk2NjU4OTY4LDk1Mj
Y4ODgzNiw3MDIzMDk5MjQsMjA0MDU5NzEzOSwxOTM1MjAwNjc3
LC04MjM4OTAzNTksLTE3MzIwMzM4MSwxNzk5NTAzOTM1LC0xOD
Q4MzQwNTIzLC0xNzE0ODA2NTU1LC04NzQ3MDIwOTksLTE1MDU3
ODcwNjNdfQ==
-->