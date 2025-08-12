```
ResourceInitializationError: failed to invoke EFS utils commands to set up EFS volumes: stderr: Failed to resolve "fs-0a8c98d014f191149.efs.us-east-1.amazonaws.com" - check that your file system ID is correct, and ensure that the VPC has an EFS mount target for this file system ID. See https://docs.aws.amazon.com/console/efs/mount-dns-name for more detail. Attempting to lookup mount target ip address using botocore. Failed to import necessary dependency botocore, please install botocore first. Traceback (most recent call last): File "/usr/sbin/supervisor_mount_efs", line 52, in <module> return_code = subprocess.check_call(["mount", "-t", "efs", "-o", opts, args.fs_id_with_path, args.dir_in_container], shell=False) File "/usr/lib64/python3.9/subprocess.py", line 373, in check_call raise CalledProcessError(retcode, cmd) subprocess.CalledProcessError: Command '['mount', '-t', 'efs', '-o', 'noresvport,tls,accesspoint=fsap-037c3014f9c04a8c4', 'fs-0a8c98d014f191149:/', '/efs-vols/efsvol']' returned non-zero exit st
```

---

01. Create an ECS VPC (+Subnets) (DNSresolution, DNShostnames Enabled)
02. Create an EFS VPC (+Subnets) (DNSresolution, DNShostnames Enabled)
03. Create a VPC Peering Connection, Edit Route Tables (ECS VPC - EFS VPC)
04. Create a EFS SecurityGroup (Inbound 2049, ECS VPC)
05. Create a EFS Volume (FileSystemPolicy Optional)
06. Create Route53 Resolver > Inbound endpoints 
  - Endpoint Category : Default
  - VPC in the Region : EFS VPC
  - Security group for this endpoint : (Inbound UDP 53 ECS VPC, Inbound TCP 53 ECS VPC)
  - Endpoint Type : IPv4
  - IP addresses : per AZ
  - Keep all other settings as default
07. Create Route53 Resolver > Outbound endpoints 
  - VPC in the Region : ECS VPC
  - Security group for this endpoint : (Outbound UDP 53 EFS VPC, Inbound TCP 53 EFS VPC or ALL 0.0.0.0/0)
  - Endpoint Type : IPv4
  - IP addresses : per AZ
  - Keep all other settings as default
08. Create Route53 Resolver > Rules
  - Rule type : Forward
  - Domain name : EFS DNS name (fs-xxxx.efs.ap-northeast-2.amazonaws.com)
  - VPCs that use this rule : ECS VPC
  - Outbound endpoint : Step 07
  - Target IP addresses : 2+ IPs assigned to Inbound Endpint Step 06
  - Keep all other settings as default
09. Create an ECS Cluster (Infrastructure : Fargate)
10. Create an ECS TaskDefinition
  - Launch type : AWS Fargate
  - Free to configure OS, Architecture, Task size, Task roles, Container
  - Storage > Volumes
    - Configuration type : Configure at task definition creation
    - Volume type : EFS
    - File system ID : Step 05
    - Root directory : /
    - Access pint ID : None
  - Storage > Container mountpoints
    - Container : Main Container
    - Source volume : Created EFS Volume
    - Free to configure Container path
11. Create an ECS Service
  - Task definition family : Created TaskDefinition
  - Capacity provider : FARGATE or FARGATE_SPOT
  - Platform version : LATEST or 1.4.0
  - Free to configure Desired tasks, Deployment options
  - Networking > VPC / Subnets : ECS VPC and its Subnets
  - Networking > SecurityGroup : Free to configure

---

EC2와 달리 Fargate 컴퓨팅 환경에서 실행된 ECS 태스크는 VPC Peering 연결을 교차하는 DNS 질의(Query)를 현재까지 지원하지 않습니다. 
이 때문에 타 VPC의 EFS 볼륨에 대한 DNS 질의가 실패하면서 ECS 태스크가 중지되며, 다음 오류 메시지가 출력됩니다.
================== ECS Task Stopped Reason ==================
ResourceInitializationError: failed to invoke EFS utils commands to set up EFS volumes: stderr: Failed to resolve "fs-xxxxxxxxxxxx.efs.ap-northeast-2.amazonaws.com" - check that your file system ID is correct, and ensure that the VPC has an EFS mount target for this file system ID. See https://docs.aws.amazon.com/console/efs/mount-dns-name for more detail. Attempting to lookup mount target ip address using botocore. Failed to import necessary dependency botocore, please install botocore first. Traceback (most recent call last): File "/usr/sbin/supervisor_mount_efs", line 52, in <module> return_code = subprocess.check_call(["mount", "-t", "efs", "-o", opts, args.fs_id_with_path, args.dir_in_container], shell=False) File "/usr/lib64/python3.9/subprocess.py", line 373, in check_call raise CalledProcessError(retcode, cmd) subprocess.CalledProcessError: Command '['mount', '-t', 'efs', '-o', 'noresvport,tls,accesspoint=fsap-xxxxxxxxxxxx', 'fs-xxxxxxxxxxxx:/', '/xxxxxx/xxxxxx']' returned non-zero exit st ...
=====================================================

AWS 공식 블로그[1]에서는 이에 대한 워크어라운드를 설명하고 있습니다. 워크어라운드에서 설명된 것과 같이 Route53 Resolver의 엔드포인트/규칙을 추가하시면 Fargate 컴퓨팅 환경 기반의 ECS 태스크(ECS_VPC)에서도 VPC 피어링 연결을 통해 타 VPC(EFS_VPC)의 EFS 볼륨을 마운트할 수 있습니다.
이 경우, Fargate 태스크가 EFS DNS 이름에 대해 질의하는 단계는 다음과 같습니다. 태스크는 ECS_VPC의 Amazon DNS 서버[2]에 "fs-xxxxxxxxxxxx.efs.ap-northeast-2.amazonaws.com"를 최초로 질의하고, 이어서 이 질의는 Route53 아웃바운드 엔드포인트, 인바운드 엔드포인트, EFS_VPC의 Amazon DNS 서버 순서로 전달됩니다. EFS_VPC의 Amazon DNS 서버는 EFS의 DNS 이름 "fs-xxxxxxxxxxxx.efs.ap-northeast-2.amazonaws.com"에 해당하는 IP 주소를 조회해 응답하고, 이 응답이 다시 Route53 인바운드 엔드포인트, 아웃바운드 엔드포인트, ECS_VPC의 Amazon DNS 서버를 역으로 거쳐 Fargate 태스크에 전달됨으로써 정상적인 마운트 작업(NFS 통신)이 이루어질 수 있습니다.
Fargate Task(ECS_VPC) 
  <-> Amazon DNS(ECS_VPC) 
  <-> Route53 Outbound Endpoint(ECS_VPC) 
  <-> Route53 Inbound Endpoint(EFS_VPC) 
  <-> Amazon DNS(EFS_VPC)
한편, Route53 리소스 생성은 추가 요금[3]을 발생시킵니다. 추가 요금의 경우, 계정 및 결제 지원 사례(케이스)를 새롭게 생성하실 경우, 더 전문적인 상담을 받으실 수 있습니다.

이 워크어라운드의 구체적인 절차는 다음과 같습니다. VPC 생성 등 이미 진행을 완료한 단계는 넘어가셔도 무방합니다.

01. ECS_VPC 생성 (서브넷 포함) (DNSresolution, DNShostnames 모두 활성화)
02. EFS_VPC 생성 (서브넷 포함) (DNSresolution, DNShostnames 모두 활성화)
03. VPC 피어링 연결 생성/수락, 피어링 연결을 포함해 라우팅 테이블 수정[4] (ECS_VPC - EFS_VPC)
04. EFS 보안그룹 생성 (2049번 포트, ECS_VPC 소스의 인바운드 허용 추가)
05. EFS 볼륨 생성 (파일 시스템 정책은 Optional)
06. Route53 Resolver(확인자) > 인바운드 엔드포인트 생성 
  - Endpoint Category : Default
  - 리전 내 VPC : EFS_VPC
  - 이 엔드포인트에 대한 보안 그룹 : 기존 보안그룹 활용 또는 새로 생성 (2개의 인바운드 규칙 필요 | UDP 53번 포트, ECS_VPC 소스의 인바운드 허용 추가, TCP 53번 포트, ECS_VPC 소스의 인바운드 허용 추가)
  - 엔드포인트 유형 : IPv4
  - IP 주소 : 가용영역별로 서브넷 선택
  - 그 외 기본 설정값 유지
07. Route53 Resolver(확인자) > 아웃바운드 엔드포인트 생성
  - 리전 내 VPC : ECS VPC
  - 이 엔드포인트에 대한 보안 그룹 : 기존 보안그룹 활용 또는 새로 생성 (UDP 53번 포트, EFS_VPC 대상의 아웃바운드 허용 추가, TCP 53번 포트, EFS_VPC 대상의 아웃바운드 허용 추가) 또는 (모든 포트, 0.0.0.0/0 대상의 아웃바운드 허용 추가)
  - 엔드포인트 유형 : IPv4
  - IP 주소 : 가용영역별로 서브넷 선택
  - 그 외 기본 설정값 유지
08. Route53 Resolver(확인자) > 규칙 생성 
  - 규칙 유형 : 전달(Forward)
  - 도메인 이름 : EFS DNS 이름 (fs-XXXXXX.efs.ap-northeast-2.amazonaws.com)
  - 이 규칙을 사용하는 VPC : ECS_VPC
  - 아웃바운드 엔드포인트 : 07단계에서 생성한 아웃바운드 엔드포인트
  - 대상 IP 주소 : 06단계에서 생성한 인바운드 엔드포인트의 IP(s)
  - 그 외 기본 설정값 유지
09. ECS 클러스터 생성 (인프라 : AWS Fargate)
10. ECS 태스크 정의 생성
  - 시작 유형 : AWS Fargate
  - OS, 아키텍쳐, 태스크 크기, 태스크 역할, 컨테이너 자유롭게 설정
  - 스토리지 > 볼륨 추가
    - 구성 유형 : 작업 정의 생성 시 구성
    - 볼륨 유형 : EFS
    - 파일 시스템 ID : 05단계에서 생성한 EFS 선택
    - 루트 디렉토리 : /
    - 액세스 포인트 ID : None
  - 스토리지 > 컨테이너 탑재 지점
    - 컨테이너 : 마운트를 진행할 메인 컨테이너
    - 소스 볼륨 : 위에서 추가한 EFS Volume 선택
    - 컨테이너 경로 : 자유롭게 설정
11. ECS 서비스 생성
  - 태스크 정의 패밀리 : 10단계에서 생성한 TaskDefinition 선택
  - 용량 공급자 : FARGATE 또는 FARGATE_SPOT
  - 플랫폼 버전 : LATEST 또는 1.4.0
  - 원하는 태스크 수, 배포 옵션 자유롭게 설정
  - 네트워킹 > VPC / 서브넷 : ECS_VPC와 소속 서브넷 선택
  - 네트워킹 > 보안그룹 자유롭게 설정

-----------------------------------------
References
- [1] https://repost.aws/ko/knowledge-center/fargate-unable-to-mount-efs
- [2] https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/AmazonDNS-concepts.html
- [3] https://aws.amazon.com/ko/route53/pricing/
- [4] https://docs.aws.amazon.com/ko_kr/vpc/latest/peering/vpc-peering-routing.html
-----------------------------------------

AWS 공식 블로그[1]에서 제안한 워크어라운드의 경우, Route53 Resolver 엔드포인트를 기반으로 DNS 이름 해석(Resolution)을 수행하기 때문에 해당 엔드포인트에 대한 추가 비용이 부과되며, 그 요금은 0.125 USD/시간/ENI입니다. 예를 들어, A존/C존으로 다중 가용영역 ECS/EFS를 구성하고 ECS에서 EFS에 연결하기 위해 Route53 인바운드/아웃바운드 엔드포인트를 각각 생성할 경우, 엔드포인트 2개(인바운드/아웃바운드) x 가용영역 2개(A존/C존), 총 4개의 ENI가 필요합니다. 이 경우 월(30일)마다 추가되는 요금은 0.125 * 24 * 4 * 30 = 360 USD에 달합니다. 이러한 추가 비용을 줄이기 위해 Route53 Resolver 엔드포인트가 아닌 다른 워크어라운드가 있는지 조사해 보았으며, 테스트를 통해 보다 비용 효율적인 새로운 워크어라운드를 찾을 수 있었습니다.

새로운 워크어라운드의 경우, ECS Fargate 태스크는 프라이빗 호스팅 영역에 추가된 A 레코드를 통해 EFS DNS 이름("fs-xxxx.efs.ap-northeast-2.amazonaws.com")에 대한 해석을 진행하고, EFS ENI의 IPv4 주소를 응답받습니다.
Route53 호스팅 영역과 레코드, 쿼리 등에 대해서만 추가 요금[2]을 부과하면 되기 때문에 Resolver 엔드포인트보다 비교적 저렴하게 DNS 이름 해석을 수행할 수 있습니다. 다만, 다중 가용영역 ECS/EFS를 구성하실 경우 호스팅 영역 레코드에 가용영역별로 EFS ENI 하나씩, 다수의 ENI IP를 동일 레코드 이름("fs-xxxx.efs.ap-northeast-2.amazonaws.com")에 등록해야 하며 이 때문에 부득이 교차 가용영역(Cross AZ) 트래픽이 발생합니다. 교차 가용영역 트래픽에 대해서는 별도의 추가 요금(복제 및 데이터 전송)이 부과됩니다.

Route53 프라이빗 호스팅 영역 기반의 워크어라운드 절차는 다음과 같습니다. VPC 생성 등 이미 진행을 완료한 단계는 넘어가셔도 무방합니다.

01. ECS_VPC 생성 (서브넷 포함) (DNSresolution, DNShostnames 모두 활성화)
02. EFS_VPC 생성 (서브넷 포함) (DNSresolution, DNShostnames 모두 활성화)
03. VPC 피어링 연결 생성/수락, 피어링 연결을 포함해 라우팅 테이블 수정[4] (ECS_VPC - EFS_VPC)
04. EFS 보안그룹 생성 (2049번 포트, ECS_VPC 소스의 인바운드 허용 추가)
05. EFS 볼륨 생성 (파일 시스템 정책은 Optional)
06. Route53 호스팅 영역 생성 
  - 도메인 이름 : efs.ap-northeast-2.amazonaws.com
  - 유형 : 프라이빗 호스팅 영역
07. Route53 호스팅 영역 레코드 생성 (fs-XXXXXX.efs.ap-northeast-2.amazonaws.com)
  - 레코드 이름 : EFS ID (fs-XXXXXX)
  - 레코드 유형 : A
  - 값 : EFS ENI ID(s) - EFS 파일 시스템 콘솔 > "네트워크" 탭 > IPv4 주소에서 조회 
    ===== 예시 =====
    10.85.170.OOO
    10.85.170.XXX
    ==============
  - 그 외 기본 설정값 유지
08. ECS 클러스터 생성 (인프라 : AWS Fargate)
09. ECS 태스크 정의 생성
  - 시작 유형 : AWS Fargate
  - OS, 아키텍쳐, 태스크 크기, 태스크 역할, 컨테이너 자유롭게 설정
  - 스토리지 > 볼륨 추가
    - 구성 유형 : 작업 정의 생성 시 구성
    - 볼륨 유형 : EFS
    - 파일 시스템 ID : 05단계에서 생성한 EFS ID 선택
    - 루트 디렉토리 : /
    - 액세스 포인트 ID : None
  - 스토리지 > 컨테이너 탑재 지점
    - 컨테이너 : 마운트를 진행할 메인 컨테이너
    - 소스 볼륨 : 위에서 추가한 EFS Volume 선택
    - 컨테이너 경로 : 자유롭게 설정
10. ECS 서비스 생성
  - 태스크 정의 패밀리 : 09단계에서 생성한 태스크 정의 선택
  - 용량 공급자 : FARGATE 또는 FARGATE_SPOT
  - 플랫폼 버전 : LATEST 또는 1.4.0
  - 원하는 태스크 수, 배포 옵션 자유롭게 설정
  - 네트워킹 > VPC / 서브넷 : ECS_VPC와 소속 서브넷 선택
  - 네트워킹 > 보안그룹 자유롭게 설정

---------------------------------------------
References
- [1] https://repost.aws/ko/knowledge-center/fargate-unable-to-mount-efs
- [2] https://aws.amazon.com/ko/route53/pricing/
- [3] https://aws.amazon.com/ko/efs/pricing/
- [4] https://docs.aws.amazon.com/ko_kr/vpc/latest/peering/vpc-peering-routing.html
---------------------------------------------