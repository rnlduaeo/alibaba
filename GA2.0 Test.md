# GA2.0 Test
한/중 간 네트워크를 개선하기 위하여 GA(Global Accelerator) 도입을 고려할 수 있다. GA는 알리바바의 기 구축된 전용회선을 기반으로 사용자에게 필요한 만큼의 bandwidth를 나누어주는 서비스이다. 사용자가 가속화 지역, 즉 end-user가 위치한 지역과 서비스 지역, 오리진 서버가 위치한 지역을 지정하고 필요한 bandwidth를 할당해 주면 그 구간은 바로 가속화 되는 형식이다. 

대부분의 고객은 한국에 서버를 두고 중국 사용자들의 접속 속도를 개선하기 위하여 GA를 검토할 수 있다. 접속의 종류에 상관없이, 즉 사용자가 mobile app을 통해 접속하든, mobile web을 통해 접속하든, 일반 PC 브라우저를 통해 접속하든 도메인을 기반으로 통신이 이루어지고 중국에서 중국 리소스(IP, 서버 등 IT리소스를 의미)를 활용하여 접속이 이루어진다면 ICP는 반드시 필요하다. 

따라서 우리는 고객이 ICP 비안을 받은 도메인을 보유했느냐, 그렇지 않느냐에 따라 다른 접근을 가져가야 한다. 그 말인즉슨 고객이 ICP 비안 도메인을 가지고 있다면 GA에서 가속화 지역을 중국 내 어느 곳이든 설정할 수 있지만, 그렇지 않다면 가속화 지역을 홍콩으로 설정함으로써 일정 부분 가속화 효과를 노려볼 수 있다.

## For ICP-filing domain
1. GA 세팅
a. Enhanced + crossborder
b. Accelerated Area: Beijing
c. Endpoint Area: Korea
d. Backend service type: custom IP (테스트로 진행할 ICP domain이 없어서 IP로 진행했다)

2. 테스트 시나리오
a. Beijing client --> Korea AWS server
b. Method: TCP ping

3. 테스트 시간
a. 16:00-17:00, 2020.06.11, normally has huge congestion in cross-border public network

4. 테스트 결과: 27배 속도 개선
a. Before GA: 221.8ms in average, 0% packet loss
b. After GA: 8.2ms in average, 0% packet loss
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202020-06-12%20at%207.29.05%20PM.png?raw=true)

## For Non-ICP domain
1. GA 세팅
a. Premium
b. Accelerated Area: Hongkong
c. Endpoint Area: Korea
d. Backend service type: custom domain (Non-ICP domain)

6. 테스트 시나리오
a. Beijing client --> Korea AWS server
b. Method: TCP ping

7. 테스트 결과: 1.72배 속도 개선 
a. Before GA: 112.6ms in average, 34% packet loss
![](https://github.com/rnlduaeo/alibaba/blob/master/Before%20GA.png?raw=true)
b. After GA: 65.4ms in average, 0% packet loss
![](https://github.com/rnlduaeo/alibaba/blob/master/GA%20application.png?raw=true)






<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4OTM2MjQ3NTEsMTQzOTE0NjE0MiwyOD
IyOTIzMTIsLTIwNzQ4NjUzMzQsLTE4NTQ0NTM1NzVdfQ==
-->