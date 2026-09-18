# B3-1 내가 만든 웹사이트를 인터넷에 올려 누구나 쓰게 하기

AWS 서울 리전에 VPC로 격리된 네트워크를 구성하고, 퍼블릭 서브넷에 배치한 EC2 인스턴스에 웹 서버를 올려 외부에서 접속 가능한 상태로 만듭니다. 보안 그룹 인바운드는 필요한 포트만 열고, 실습에는 EC2/VPC 구성에 필요한 범위로 권한을 제한한 IAM 사용자를 사용합니다.

외부 접속 검증은 (A) 브라우저에서 `http://<퍼블릭IP>`로 접속하는 방식을 선택했습니다.

## 실행 환경

로컬에서 AWS CLI로 리소스를 생성하고, SSH로 인스턴스에 접속해 웹 서버를 설치했습니다.

```bash
$ sw_vers
ProductName:		macOS
ProductVersion:		15.7.7
BuildVersion:		24G720
$ aws --version
aws-cli/2.36.47 Python/3.14.7 Darwin/24.6.0 source/arm64
$ ssh -V
OpenSSH_9.9p2, LibreSSL 3.3.6
```

모든 리소스는 서울 리전(ap-northeast-2)에 만들었습니다.

## IAM 사용자와 권한

실습 전용 IAM 사용자 `b3-1-practice`를 만들고, EC2와 VPC 구성에 필요한 액션만 허용한 정책을 직접 연결했습니다. 루트 계정은 이 사용자를 만들 때만 사용했고, 이후 모든 명령은 이 사용자의 액세스 키를 등록한 프로필로 실행했습니다.

```bash
$ export AWS_PROFILE=codyssey-b3-1
$ aws sts get-caller-identity
{
    "UserId": "AIDAYGB5FKKCKVE3I6QMJ",
    "Account": "************",
    "Arn": "arn:aws:iam::************:user/b3-1-practice"
}
```

연결한 정책은 다음과 같습니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Ec2ReadOnly",
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Sid": "Ec2NetworkAndInstanceManage",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:ModifySubnetAttribute",
        "ec2:CreateInternetGateway",
        "ec2:DeleteInternetGateway",
        "ec2:AttachInternetGateway",
        "ec2:DetachInternetGateway",
        "ec2:CreateRouteTable",
        "ec2:DeleteRouteTable",
        "ec2:CreateRoute",
        "ec2:DeleteRoute",
        "ec2:AssociateRouteTable",
        "ec2:DisassociateRouteTable",
        "ec2:CreateSecurityGroup",
        "ec2:DeleteSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:CreateKeyPair",
        "ec2:DeleteKeyPair",
        "ec2:ImportKeyPair",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-northeast-2"
        }
      }
    }
  ]
}
```

조회용 `ec2:Describe*` 외의 액션은 `aws:RequestedRegion` 조건으로 서울 리전에서만 허용됩니다. EC2의 생성 계열 액션은 대부분 리소스 ARN 단위 제한을 지원하지 않기 때문에, 서비스와 액션 목록에 리전 조건을 더하는 방식으로 범위를 좁혔습니다. S3, RDS 같은 실습과 무관한 서비스와 IAM 조작 권한은 넣지 않았고 AdministratorAccess도 연결하지 않았습니다.

실제로 이 사용자로는 자신에게 연결된 정책조차 조회할 수 없습니다.

```bash
$ aws iam list-attached-user-policies --user-name b3-1-practice

aws: [ERROR]: An error occurred (AccessDenied) when calling the ListAttachedUserPolicies operation: User: arn:aws:iam::************:user/b3-1-practice is not authorized to perform: iam:ListAttachedUserPolicies on resource: user b3-1-practice because no identity-based policy allows the iam:ListAttachedUserPolicies action.
```

## 네트워크 구성

VPC, 퍼블릭 서브넷, 인터넷 게이트웨이, 라우트 테이블을 차례로 만들었습니다. 모든 리소스에는 `b3-1-` 접두사를 붙인 Name 태그를 달아 나중에 목록에서 골라낼 수 있게 했습니다.

```bash
$ aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=b3-1-vpc}]' \
    --query 'Vpc.{VpcId:VpcId,CidrBlock:CidrBlock,State:State}'
{
    "VpcId": "vpc-03eae37b07d72dc9f",
    "CidrBlock": "10.0.0.0/16",
    "State": "pending"
}
```

서브넷은 VPC 대역 10.0.0.0/16 안에서 10.0.1.0/24를 잘라 가용 영역 ap-northeast-2a에 만들었습니다.

```bash
$ aws ec2 create-subnet --vpc-id vpc-03eae37b07d72dc9f --cidr-block 10.0.1.0/24 \
    --availability-zone ap-northeast-2a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=b3-1-public-subnet}]' \
    --query 'Subnet.{SubnetId:SubnetId,CidrBlock:CidrBlock,AvailabilityZone:AvailabilityZone,MapPublicIpOnLaunch:MapPublicIpOnLaunch}'
{
    "SubnetId": "subnet-07dfd6eac691dacaa",
    "CidrBlock": "10.0.1.0/24",
    "AvailabilityZone": "ap-northeast-2a",
    "MapPublicIpOnLaunch": false
}
```

`MapPublicIpOnLaunch`가 false이면 이 서브넷에서 만든 인스턴스에 퍼블릭 IP가 붙지 않습니다. 서브넷 속성을 켜서 이후 생성하는 인스턴스가 자동으로 퍼블릭 IP를 받도록 했습니다.

```bash
$ aws ec2 modify-subnet-attribute --subnet-id subnet-07dfd6eac691dacaa --map-public-ip-on-launch
$ aws ec2 describe-subnets --subnet-ids subnet-07dfd6eac691dacaa --query 'Subnets[0].MapPublicIpOnLaunch'
true
```

인터넷 게이트웨이를 만들어 VPC에 연결했습니다. 연결 명령은 성공하면 출력이 없습니다.

```bash
$ aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=b3-1-igw}]' \
    --query 'InternetGateway.InternetGatewayId' --output text
igw-0229ed6acb6aa410a
$ aws ec2 attach-internet-gateway --internet-gateway-id igw-0229ed6acb6aa410a --vpc-id vpc-03eae37b07d72dc9f
```

라우트 테이블을 만들고 기본 경로를 인터넷 게이트웨이로 향하게 한 뒤 서브넷에 연결했습니다.

```bash
$ aws ec2 create-route-table --vpc-id vpc-03eae37b07d72dc9f \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=b3-1-public-rtb}]' \
    --query 'RouteTable.RouteTableId' --output text
rtb-0f3a3f86b4e749e9d
$ aws ec2 create-route --route-table-id rtb-0f3a3f86b4e749e9d --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0229ed6acb6aa410a
{
    "Return": true
}
$ aws ec2 associate-route-table --route-table-id rtb-0f3a3f86b4e749e9d --subnet-id subnet-07dfd6eac691dacaa --query 'AssociationId' --output text
rtbassoc-0f2fb76bb65cb0765
```

구성한 라우트 테이블의 경로와 연결 상태입니다.

```bash
$ aws ec2 describe-route-tables --route-table-ids rtb-0f3a3f86b4e749e9d \
    --query 'RouteTables[0].{Routes:Routes[].{Destination:DestinationCidrBlock,Target:join(``,[GatewayId]),State:State},Subnet:Associations[0].SubnetId}'
{
    "Routes": [
        {
            "Destination": "10.0.0.0/16",
            "Target": "local",
            "State": "active"
        },
        {
            "Destination": "0.0.0.0/0",
            "Target": "igw-0229ed6acb6aa410a",
            "State": "active"
        }
    ],
    "Subnet": "subnet-07dfd6eac691dacaa"
}
```

local 경로는 VPC를 만들 때 자동으로 생기며 VPC 내부 통신을 처리합니다. 여기에 더한 0.0.0.0/0 경로가 인터넷 게이트웨이를 향하기 때문에, 이 서브넷에 있는 인스턴스는 VPC 대역에 속하지 않는 주소로 나가는 트래픽을 게이트웨이로 보냅니다. 이 경로가 없으면 인스턴스에 퍼블릭 IP가 있어도 외부와 통신하지 못합니다.
