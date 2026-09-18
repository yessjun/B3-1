# 리소스 정리 체크리스트

실습에 만든 리소스를 모두 삭제했습니다. 헬스체크 방식을 확인하느라 같은 구성을 한 번 더 만들었기 때문에 정리도 두 번 했고, 아래에 두 번 모두 기록했습니다. 항목별 확인 결과는 마지막 절에 있습니다.

| 항목 | 상태 | 확인 방법 |
|---|---|---|
| EC2 인스턴스 | 종료 완료 | `describe-instances`에서 `terminated` |
| EBS 볼륨 | 삭제 완료 | 인스턴스 종료 시 함께 삭제, `describe-volumes` 빈 목록 |
| Elastic IP | 해당 없음 | 할당하지 않음, `describe-addresses` 빈 목록 |
| Internet Gateway | 분리 후 삭제 완료 | `describe-internet-gateways` 빈 목록 |
| VPC | 삭제 완료 | `describe-vpcs` 빈 목록 |
| Subnet | 삭제 완료 | `describe-subnets` 빈 목록 |
| Route Table | 삭제 완료 | `describe-route-tables` 빈 목록 |
| Security Group | 삭제 완료 | `describe-security-groups` 빈 목록 |
| 키페어 | 삭제 완료 | `describe-key-pairs`에서 존재하지 않음 |
| NAT Gateway | 해당 없음 | 생성하지 않음 |
| ELB/ALB | 해당 없음 | 생성하지 않음 |
| RDS | 해당 없음 | 생성하지 않음 |

## 삭제 순서

인스턴스를 먼저 종료해야 그 인스턴스가 사용하던 네트워크 인터페이스가 풀리고, 그래야 보안 그룹과 서브넷을 지울 수 있습니다. 인터넷 게이트웨이는 VPC에서 분리한 뒤에 삭제되고, VPC는 안에 남은 리소스가 하나도 없어야 삭제됩니다. 그래서 인스턴스, 보안 그룹, 서브넷, 라우트 테이블, 인터넷 게이트웨이, VPC 순으로 지웠습니다.

## 인스턴스 종료

```bash
$ aws ec2 terminate-instances --instance-ids i-03b18cea700e89319 \
    --query 'TerminatingInstances[0].{Id:InstanceId,Previous:PreviousState.Name,Current:CurrentState.Name}'
{
    "Id": "i-03b18cea700e89319",
    "Previous": "running",
    "Current": "shutting-down"
}
$ aws ec2 wait instance-terminated --instance-ids i-03b18cea700e89319
$ aws ec2 describe-instances --instance-ids i-03b18cea700e89319 --query 'Reservations[0].Instances[0].State.Name' --output text
terminated
```

루트 볼륨은 인스턴스 생성 시 삭제 옵션이 켜져 있어 종료와 함께 사라졌습니다. 볼륨 ID로 조회하면 존재하지 않습니다.

```bash
$ aws ec2 describe-volumes --volume-ids vol-0a0024f0cf452a8bd --query 'Volumes[].VolumeId'

aws: [ERROR]: An error occurred (InvalidVolume.NotFound) when calling the DescribeVolumes operation: The volume 'vol-0a0024f0cf452a8bd' does not exist.
```

## 네트워크 리소스 삭제

```bash
$ aws ec2 delete-security-group --group-id sg-0d48678d6a69187e0
{
    "Return": true,
    "GroupId": "sg-0d48678d6a69187e0"
}
$ aws ec2 delete-subnet --subnet-id subnet-07dfd6eac691dacaa
$ aws ec2 delete-route-table --route-table-id rtb-0f3a3f86b4e749e9d
$ aws ec2 detach-internet-gateway --internet-gateway-id igw-0229ed6acb6aa410a --vpc-id vpc-03eae37b07d72dc9f
$ aws ec2 delete-internet-gateway --internet-gateway-id igw-0229ed6acb6aa410a
$ aws ec2 delete-vpc --vpc-id vpc-03eae37b07d72dc9f
$ aws ec2 delete-key-pair --key-name b3-1-key
{
    "Return": true,
    "KeyPairId": "key-0c33ede1c7f42815a"
}
```

## 헬스체크 확인용 리소스 정리

같은 순서로 두 번째 구성도 지웠습니다.

```bash
$ aws ec2 terminate-instances --instance-ids i-068670d0e0e50962b --query 'TerminatingInstances[0].CurrentState.Name' --output text
shutting-down
$ aws ec2 wait instance-terminated --instance-ids i-068670d0e0e50962b
$ aws ec2 delete-security-group --group-id sg-06449f52cc86ba02c
{
    "Return": true,
    "GroupId": "sg-06449f52cc86ba02c"
}
$ aws ec2 delete-subnet --subnet-id subnet-0ae0d7a1f68da0660
$ aws ec2 delete-route-table --route-table-id rtb-0e910eacbfdbafc52
$ aws ec2 detach-internet-gateway --internet-gateway-id igw-011fa5287192e33e4 --vpc-id vpc-095376937e4df04a6
$ aws ec2 delete-internet-gateway --internet-gateway-id igw-011fa5287192e33e4
$ aws ec2 delete-vpc --vpc-id vpc-095376937e4df04a6
$ aws ec2 delete-key-pair --key-name b3-1-key --query Return --output text
True
```

## 남은 리소스 확인

이름 태그 `b3-1-` 로 조회한 결과와 계정 전체의 볼륨, Elastic IP 조회 결과입니다.

```bash
$ aws ec2 describe-instances --filters "Name=tag:Name,Values=b3-1-web" --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name}'
[
    {
        "Id": "i-03b18cea700e89319",
        "State": "terminated"
    },
    {
        "Id": "i-068670d0e0e50962b",
        "State": "terminated"
    }
]
$ aws ec2 describe-volumes --query 'Volumes[].VolumeId'
[]
$ aws ec2 describe-addresses --query 'Addresses[].PublicIp'
[]
$ aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=b3-1-*" --query 'InternetGateways[].InternetGatewayId'
[]
$ aws ec2 describe-vpcs --filters "Name=tag:Name,Values=b3-1-*" --query 'Vpcs[].VpcId'
[]
$ aws ec2 describe-subnets --filters "Name=tag:Name,Values=b3-1-*" --query 'Subnets[].SubnetId'
[]
$ aws ec2 describe-route-tables --filters "Name=tag:Name,Values=b3-1-*" --query 'RouteTables[].RouteTableId'
[]
$ aws ec2 describe-security-groups --filters "Name=tag:Name,Values=b3-1-*" --query 'SecurityGroups[].GroupId'
[]
$ aws ec2 describe-key-pairs --key-names b3-1-key
aws: [ERROR]: An error occurred (InvalidKeyPair.NotFound) when calling the DescribeKeyPairs operation: The key pair 'b3-1-key' does not exist
```

종료된 인스턴스는 조회 목록에 한동안 `terminated` 상태로 남았다가 사라집니다. 이 상태에서는 요금이 발생하지 않습니다.
