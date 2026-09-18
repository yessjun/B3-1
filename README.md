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
