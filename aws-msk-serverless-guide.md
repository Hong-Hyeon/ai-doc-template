# AWS MSK Serverless Implementation Guide

## 📋 목차

1. [개요](#개요)
2. [MSK Serverless 이해하기](#msk-serverless-이해하기)
3. [아키텍처 설계](#아키텍처-설계)
4. [클러스터 생성 및 설정](#클러스터-생성-및-설정)
5. [보안 및 인증](#보안-및-인증)
6. [Producer 구현](#producer-구현)
7. [Consumer 구현](#consumer-구현)
8. [토픽 관리](#토픽-관리)
9. [모니터링 및 로깅](#모니터링-및-로깅)
10. [비용 최적화](#비용-최적화)
11. [트러블슈팅](#트러블슈팅)
12. [베스트 프랙티스](#베스트-프랙티스)

---

## 개요

본 문서는 **AWS MSK Serverless**를 사용한 Kafka 구현 가이드입니다.
MSK Serverless는 서버 관리 없이 Apache Kafka를 사용할 수 있는 완전 관리형 서비스로,
용량 계획 없이 자동으로 확장/축소됩니다.

### MSK Serverless 선택 이유

| 특징 | 설명 |
|------|------|
| **자동 확장** | 트래픽에 따라 자동으로 용량 조정 |
| **서버 관리 불필요** | 브로커, Zookeeper 관리 불필요 |
| **비용 효율** | 사용한 만큼만 지불 (Pay-as-you-go) |
| **고가용성** | 멀티 AZ 배포로 자동 장애 복구 |
| **보안** | IAM 기반 인증 및 VPC 격리 |

### 기술 스택

| 구성 요소 | 기술 |
|----------|------|
| Kafka 클러스터 | AWS MSK Serverless |
| 클라이언트 라이브러리 | aiokafka (Python) |
| 인증 | IAM, SASL/SCRAM |
| 모니터링 | CloudWatch, MSK Metrics |
| 인프라 코드 | Terraform / AWS CDK |

---

## MSK Serverless 이해하기

### MSK Serverless vs MSK Provisioned

| 항목 | MSK Serverless | MSK Provisioned |
|------|----------------|-----------------|
| 용량 관리 | 자동 | 수동 |
| 최소 용량 | 없음 | 브로커 인스턴스 필요 |
| 확장성 | 자동 | 수동 조정 |
| 비용 모델 | 사용량 기반 | 인스턴스 기반 |
| 설정 제어 | 제한적 | 완전한 제어 |
| 사용 사례 | 가변 트래픽, 개발/테스트 | 고정 트래픽, 프로덕션 |

### MSK Serverless 제한사항

1. **토픽당 최대 파티션 수**: 1,000개
2. **클라이언트당 최대 연결 수**: 1,000개
3. **메시지 크기**: 최대 1MB
4. **리전**: 특정 리전에서만 사용 가능
5. **Kafka 버전**: AWS가 관리하는 특정 버전만 지원

### MSK Serverless 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Services                      │
│  (EC2, ECS, Lambda, EKS 등)                                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ IAM / SASL 인증
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              AWS MSK Serverless Cluster                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Broker 1   │  │   Broker 2   │  │   Broker 3   │     │
│  │   (AZ-1)     │  │   (AZ-2)     │  │   (AZ-3)     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌──────────────────────────────────────────────┐           │
│  │         Auto-scaling Engine                  │           │
│  │  (트래픽에 따라 자동 확장/축소)                  │           │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
                       │
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              CloudWatch Metrics & Logs                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 아키텍처 설계

### 네트워크 아키텍처

MSK Serverless는 VPC 내에서만 동작합니다. 다음 네트워크 구성이 필요합니다:

```
┌─────────────────────────────────────────────────────────────┐
│                         VPC                                  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Private Subnets                         │  │
│  │  ┌──────────────┐  ┌──────────────┐                │  │
│  │  │  App Server  │  │  App Server  │                │  │
│  │  │   (AZ-1)     │  │   (AZ-2)     │                │  │
│  │  └──────┬───────┘  └──────┬───────┘                │  │
│  └─────────┼─────────────────┼─────────────────────────┘  │
│            │                 │                              │
│            ▼                 ▼                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         MSK Serverless Cluster                       │  │
│  │  (VPC Endpoints를 통해 접근)                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         VPC Endpoints (com.amazonaws.region.kafka)  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 토픽 설계 전략

#### 네이밍 컨벤션

```
{domain}.{entity}.{event-type}

예시:
- order.order.created
- order.order.status-changed
- payment.payment.completed
- payment.payment.failed
- inventory.stock.updated
- user.user.registered
```

#### 파티션 전략

```python
# 파티션 수 결정 가이드
def calculate_partitions(expected_throughput: int, target_throughput_per_partition: int = 1000) -> int:
    """
    파티션 수 계산
    
    Args:
        expected_throughput: 예상 초당 메시지 수
        target_throughput_per_partition: 파티션당 목표 처리량
    
    Returns:
        권장 파티션 수
    """
    partitions = (expected_throughput // target_throughput_per_partition) + 1
    # MSK Serverless 제한: 최대 1,000개
    return min(partitions, 1000)

# 예시
# 초당 5,000 메시지 예상
partitions = calculate_partitions(5000, 1000)  # 6개 파티션
```

#### 레플리케이션 팩터

MSK Serverless는 자동으로 3개의 복제본을 유지합니다.

### Consumer Group 설계

```python
# Consumer Group 네이밍 컨벤션
# {service-name}.{topic-pattern}

예시:
- order-service.order.order.created
- notification-service.order.order.*
- analytics-service.*.*.*
```

---

## 클러스터 생성 및 설정

### 1. AWS CLI를 사용한 클러스터 생성

#### 1.1 클러스터 생성

```bash
# 클러스터 생성
aws kafka create-cluster \
  --cluster-name "my-msk-serverless-cluster" \
  --serverless \
  --region ap-northeast-2

# 클러스터 ARN 저장
CLUSTER_ARN=$(aws kafka list-clusters \
  --cluster-name-filter "my-msk-serverless-cluster" \
  --query 'ClusterInfoList[0].ClusterArn' \
  --output text \
  --region ap-northeast-2)

echo "Cluster ARN: $CLUSTER_ARN"
```

#### 1.2 VPC 및 보안 그룹 설정

```bash
# VPC ID 확인
VPC_ID="vpc-xxxxxxxxx"

# 서브넷 ID 확인 (최소 2개 AZ 필요)
SUBNET_1="subnet-xxxxxxxxx"  # AZ-1
SUBNET_2="subnet-yyyyyyyyy"  # AZ-2
SUBNET_3="subnet-zzzzzzzzz"  # AZ-3

# 보안 그룹 생성 (MSK용)
SG_ID=$(aws ec2 create-security-group \
  --group-name msk-serverless-sg \
  --description "Security group for MSK Serverless" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# 인바운드 규칙 추가 (같은 VPC 내에서만 접근)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 9098 \
  --source-group $SG_ID
```

#### 1.3 클러스터 구성 업데이트

```bash
# 클러스터에 VPC 및 보안 그룹 연결
aws kafka update-cluster-configuration \
  --cluster-arn $CLUSTER_ARN \
  --current-version "1" \
  --serverless-configuration \
    ClientAuthentication="SASL_IAM" \
    VpcConfig="SubnetIds=$SUBNET_1,$SUBNET_2,$SUBNET_3,SecurityGroupIds=$SG_ID"
```

### 2. Terraform을 사용한 인프라 코드

#### 2.1 기본 리소스

```hcl
# terraform/msk-serverless/main.tf

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC 데이터 소스
data "aws_vpc" "main" {
  id = var.vpc_id
}

# 서브넷 데이터 소스
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [var.vpc_id]
  }
  
  tags = {
    Type = "private"
  }
}

# 보안 그룹
resource "aws_security_group" "msk" {
  name        = "${var.cluster_name}-sg"
  description = "Security group for MSK Serverless"
  vpc_id      = var.vpc_id

  ingress {
    description = "Kafka from VPC"
    from_port   = 9098
    to_port     = 9098
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.cluster_name}-sg"
  }
}

# MSK Serverless 클러스터
resource "aws_msk_cluster" "serverless" {
  cluster_name           = var.cluster_name
  kafka_version          = "3.5.1"  # MSK Serverless 지원 버전
  number_of_broker_nodes = 0  # Serverless는 브로커 노드 수가 0

  serverless {
    vpc_config {
      subnet_ids         = data.aws_subnets.private.ids
      security_group_ids = [aws_security_group.msk.id]
    }
  }

  tags = {
    Name        = var.cluster_name
    Environment = var.environment
  }
}

# 클러스터 ARN 출력
output "cluster_arn" {
  value = aws_msk_cluster.serverless.arn
}

output "bootstrap_servers" {
  value = aws_msk_cluster.serverless.bootstrap_brokers_sasl_iam
}
```

#### 2.2 변수 정의

```hcl
# terraform/msk-serverless/variables.tf

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-2"
}

variable "cluster_name" {
  description = "MSK Serverless cluster name"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for MSK cluster"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}
```

### 3. AWS CDK를 사용한 인프라 코드

```python
# infrastructure/msk_stack.py

from aws_cdk import (
    Stack,
    aws_ec2 as ec2,
    aws_msk as msk,
    CfnOutput,
)
from constructs import Construct


class MSKServerlessStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # VPC 가져오기
        vpc = ec2.Vpc.from_lookup(
            self, "VPC",
            vpc_id=self.node.try_get_context("vpc_id")
        )

        # 보안 그룹
        security_group = ec2.SecurityGroup(
            self, "MSKSecurityGroup",
            vpc=vpc,
            description="Security group for MSK Serverless",
            allow_all_outbound=True
        )

        security_group.add_ingress_rule(
            peer=ec2.Peer.ipv4(vpc.vpc_cidr_block),
            connection=ec2.Port.tcp(9098),
            description="Kafka from VPC"
        )

        # MSK Serverless 클러스터
        cluster = msk.CfnCluster(
            self, "MSKServerlessCluster",
            cluster_name="my-msk-serverless-cluster",
            kafka_version="3.5.1",
            number_of_broker_nodes=0,  # Serverless는 0

            serverless={
                "vpcConfig": {
                    "subnetIds": [subnet.subnet_id for subnet in vpc.private_subnets],
                    "securityGroupIds": [security_group.security_group_id]
                }
            },

            client_authentication={
                "sasl": {
                    "iam": {
                        "enabled": True
                    }
                }
            }
        )

        # 출력
        CfnOutput(
            self, "ClusterArn",
            value=cluster.ref,
            description="MSK Serverless Cluster ARN"
        )

        CfnOutput(
            self, "BootstrapServers",
            value=cluster.attr_bootstrap_broker_string_sasl_iam,
            description="MSK Bootstrap Servers"
        )
```

---

## 보안 및 인증

### 1. IAM 인증 설정

MSK Serverless는 IAM 기반 인증을 지원합니다.

#### 1.1 IAM 정책 생성

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:AlterCluster",
        "kafka-cluster:DescribeCluster"
      ],
      "Resource": "arn:aws:kafka:ap-northeast-2:123456789012:cluster/my-msk-serverless-cluster/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:*Topic*",
        "kafka-cluster:WriteData",
        "kafka-cluster:ReadData"
      ],
      "Resource": "arn:aws:kafka:ap-northeast-2:123456789012:topic/my-msk-serverless-cluster/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:AlterGroup",
        "kafka-cluster:DescribeGroup"
      ],
      "Resource": "arn:aws:kafka:ap-northeast-2:123456789012:group/my-msk-serverless-cluster/*"
    }
  ]
}
```

#### 1.2 IAM 역할 생성 (EC2/ECS/EKS용)

```bash
# IAM 역할 생성
aws iam create-role \
  --role-name msk-producer-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }]
  }'

# 정책 연결
aws iam put-role-policy \
  --role-name msk-producer-role \
  --policy-name MSKProducerPolicy \
  --policy-document file://msk-policy.json
```

### 2. Python 클라이언트에서 IAM 인증 사용

#### 2.1 boto3를 사용한 인증

```python
# src/infrastructure/messaging/msk_auth.py

import boto3
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
from aiokafka.helpers import create_ssl_context
import ssl

class MSKIAMAuth:
    """MSK IAM 인증 헬퍼"""
    
    def __init__(self, region: str = "ap-northeast-2"):
        self.region = region
        self.session = boto3.Session()
    
    def get_sasl_mechanism(self) -> str:
        """SASL 메커니즘 반환"""
        return "AWS_MSK_IAM"
    
    def get_sasl_plain_username(self) -> str:
        """IAM 사용자명 (빈 문자열)"""
        return ""
    
    def get_sasl_plain_password(self) -> str:
        """IAM 인증 토큰 생성"""
        # boto3를 사용하여 MSK 인증 토큰 생성
        # 실제로는 aiokafka가 내부적으로 처리
        return ""
    
    def get_security_protocol(self) -> str:
        """보안 프로토콜"""
        return "SASL_SSL"
    
    def get_ssl_context(self) -> ssl.SSLContext:
        """SSL 컨텍스트 생성"""
        context = create_ssl_context()
        context.check_hostname = False
        return context
```

#### 2.2 환경 변수 설정

```bash
# .env 파일
KAFKA_BOOTSTRAP_SERVERS=b-1.my-msk-cluster.xxxxx.c2.kafka-serverless.ap-northeast-2.amazonaws.com:9098
KAFKA_REGION=ap-northeast-2
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=AWS_MSK_IAM
```

### 3. VPC 엔드포인트 설정 (선택사항)

프라이빗 서브넷에서 인터넷 게이트웨이 없이 MSK에 접근하려면 VPC 엔드포인트가 필요합니다.

```hcl
# VPC 엔드포인트
resource "aws_vpc_endpoint" "kafka" {
  vpc_id              = var.vpc_id
  service_name         = "com.amazonaws.${var.aws_region}.kafka"
  vpc_endpoint_type    = "Interface"
  subnet_ids           = data.aws_subnets.private.ids
  security_group_ids   = [aws_security_group.vpc_endpoint.id]
  private_dns_enabled  = true

  tags = {
    Name = "msk-vpc-endpoint"
  }
}
```

---

## Producer 구현

### 1. 기본 Producer 구현

```python
# src/infrastructure/messaging/producer.py

import json
import asyncio
from typing import Optional, Dict, Any
from datetime import datetime
import uuid

from aiokafka import AIOKafkaProducer
import boto3
from botocore.auth import SigV4Auth
from botocore.awsrequest import AWSRequest

from src.core.config import settings
from src.core.logging import logger
from src.domain.events.base import DomainEvent


class MSKProducer:
    """MSK Serverless Producer with IAM authentication"""
    
    def __init__(
        self,
        bootstrap_servers: str,
        region: str = "ap-northeast-2",
    ):
        self.bootstrap_servers = bootstrap_servers
        self.region = region
        self.producer: Optional[AIOKafkaProducer] = None
        self._session = boto3.Session(region_name=region)
    
    async def start(self) -> None:
        """Producer 시작"""
        logger.info(
            "Starting MSK Producer...",
            bootstrap_servers=self.bootstrap_servers,
            region=self.region
        )
        
        # IAM 인증을 위한 SASL 설정
        self.producer = AIOKafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            security_protocol="SASL_SSL",
            sasl_mechanism="AWS_MSK_IAM",
            sasl_plain_username="",  # IAM 인증에서는 빈 문자열
            sasl_plain_password="",  # aiokafka가 내부적으로 처리
            ssl_context=self._create_ssl_context(),
            # Producer 설정
            acks="all",  # 모든 복제본에 쓰기 완료 후 응답
            enable_idempotence=True,  # 멱등성 활성화
            max_batch_size=16384,  # 16KB
            linger_ms=10,  # 배치 대기 시간
            compression_type="gzip",  # 압축
            retries=3,  # 재시도 횟수
            request_timeout_ms=30000,  # 요청 타임아웃
            client_id=f"producer-{uuid.uuid4().hex[:8]}",
        )
        
        await self.producer.start()
        logger.info("MSK Producer started")
    
    async def stop(self) -> None:
        """Producer 종료"""
        if self.producer:
            logger.info("Stopping MSK Producer...")
            await self.producer.stop()
            self.producer = None
            logger.info("MSK Producer stopped")
    
    def _create_ssl_context(self):
        """SSL 컨텍스트 생성"""
        import ssl
        from aiokafka.helpers import create_ssl_context
        
        context = create_ssl_context()
        context.check_hostname = False
        return context
    
    async def publish(
        self,
        topic: str,
        key: str,
        value: Dict[str, Any],
        headers: Optional[Dict[str, str]] = None,
    ) -> None:
        """
        메시지 발행
        
        Args:
            topic: 토픽 이름
            key: 파티션 키
            value: 메시지 값
            headers: 메시지 헤더
        """
        if not self.producer:
            raise RuntimeError("Producer not started. Call start() first.")
        
        try:
            # 메시지 헤더 추가
            kafka_headers = []
            if headers:
                for k, v in headers.items():
                    kafka_headers.append((k, v.encode("utf-8")))
            
            # 메시지 발행
            future = await self.producer.send(
                topic=topic,
                key=key.encode("utf-8") if key else None,
                value=json.dumps(value).encode("utf-8"),
                headers=kafka_headers,
            )
            
            # 응답 대기
            record_metadata = await future
            
            logger.info(
                "Message published",
                topic=topic,
                partition=record_metadata.partition,
                offset=record_metadata.offset,
                key=key,
            )
            
        except Exception as e:
            logger.error(
                "Failed to publish message",
                topic=topic,
                key=key,
                error=str(e),
                exc_info=True
            )
            raise
    
    async def publish_event(self, event: DomainEvent) -> None:
        """
        도메인 이벤트 발행
        
        Args:
            event: 도메인 이벤트
        """
        await self.publish(
            topic=event.topic,
            key=event.partition_key,
            value=event.to_dict(),
            headers={
                "event_type": event.event_type,
                "event_id": str(event.event_id),
                "timestamp": datetime.utcnow().isoformat(),
            }
        )
```

### 2. Producer 팩토리 및 싱글톤

```python
# src/infrastructure/messaging/producer_factory.py

from typing import Optional
from src.infrastructure.messaging.producer import MSKProducer
from src.core.config import settings
from src.core.logging import logger

_producer_instance: Optional[MSKProducer] = None


async def get_producer() -> MSKProducer:
    """Producer 싱글톤 인스턴스 반환"""
    global _producer_instance
    
    if _producer_instance is None:
        _producer_instance = MSKProducer(
            bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
            region=settings.KAFKA_REGION,
        )
        await _producer_instance.start()
    
    return _producer_instance


async def close_producer() -> None:
    """Producer 종료"""
    global _producer_instance
    
    if _producer_instance:
        await _producer_instance.stop()
        _producer_instance = None
        logger.info("Producer closed")
```

### 3. 이벤트 발행 예시

```python
# src/domain/events/order_created.py

from dataclasses import dataclass
from datetime import datetime
from uuid import UUID, uuid4
from typing import Dict, Any

from src.domain.events.base import DomainEvent


@dataclass
class OrderCreatedEvent(DomainEvent):
    """주문 생성 이벤트"""
    
    order_id: str
    user_id: str
    total_amount: int
    items: list[Dict[str, Any]]
    
    @property
    def topic(self) -> str:
        return "order.order.created"
    
    @property
    def event_type(self) -> str:
        return "order.created"
    
    @property
    def partition_key(self) -> str:
        return self.order_id
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "event_id": str(self.event_id),
            "event_type": self.event_type,
            "timestamp": self.timestamp.isoformat(),
            "order_id": self.order_id,
            "user_id": self.user_id,
            "total_amount": self.total_amount,
            "items": self.items,
        }


# 사용 예시
async def create_order(order_data: Dict[str, Any]) -> None:
    # 주문 생성 로직...
    
    # 이벤트 발행
    event = OrderCreatedEvent(
        event_id=uuid4(),
        timestamp=datetime.utcnow(),
        order_id=order_data["order_id"],
        user_id=order_data["user_id"],
        total_amount=order_data["total_amount"],
        items=order_data["items"],
    )
    
    producer = await get_producer()
    await producer.publish_event(event)
```

### 4. 배치 발행 (성능 최적화)

```python
# src/infrastructure/messaging/batch_producer.py

from typing import List, Tuple
from src.infrastructure.messaging.producer import MSKProducer


class BatchProducer:
    """배치 메시지 발행"""
    
    def __init__(self, producer: MSKProducer, batch_size: int = 100):
        self.producer = producer
        self.batch_size = batch_size
        self.buffer: List[Tuple[str, str, dict]] = []
    
    async def add(self, topic: str, key: str, value: dict) -> None:
        """메시지 버퍼에 추가"""
        self.buffer.append((topic, key, value))
        
        if len(self.buffer) >= self.batch_size:
            await self.flush()
    
    async def flush(self) -> None:
        """버퍼의 모든 메시지 발행"""
        if not self.buffer:
            return
        
        # 병렬 발행
        tasks = [
            self.producer.publish(topic, key, value)
            for topic, key, value in self.buffer
        ]
        
        await asyncio.gather(*tasks, return_exceptions=True)
        
        logger.info(f"Flushed {len(self.buffer)} messages")
        self.buffer.clear()
```

---

## Consumer 구현

### 1. 기본 Consumer 구현

```python
# src/infrastructure/messaging/consumer.py

import asyncio
import json
from typing import List, Optional, Callable, Awaitable, Dict, Any
from uuid import uuid4

from aiokafka import AIOKafkaConsumer
from aiokafka.errors import KafkaError

from src.core.config import settings
from src.core.logging import logger


EventHandler = Callable[[Dict[str, Any]], Awaitable[None]]


class MSKConsumer:
    """MSK Serverless Consumer with IAM authentication"""
    
    def __init__(
        self,
        topics: List[str],
        group_id: str,
        bootstrap_servers: str,
        region: str = "ap-northeast-2",
    ):
        self.topics = topics
        self.group_id = group_id
        self.bootstrap_servers = bootstrap_servers
        self.region = region
        self.consumer: Optional[AIOKafkaConsumer] = None
        self.handlers: Dict[str, EventHandler] = {}
        self.running = False
    
    def register_handler(self, event_type: str, handler: EventHandler) -> None:
        """
        이벤트 핸들러 등록
        
        Args:
            event_type: 이벤트 타입 (예: "order.created")
            handler: 이벤트 처리 함수
        """
        self.handlers[event_type] = handler
        logger.info(f"Registered handler for event type: {event_type}")
    
    async def start(self) -> None:
        """Consumer 시작"""
        logger.info(
            "Starting MSK Consumer...",
            topics=self.topics,
            group_id=self.group_id,
            bootstrap_servers=self.bootstrap_servers
        )
        
        self.consumer = AIOKafkaConsumer(
            *self.topics,
            bootstrap_servers=self.bootstrap_servers,
            group_id=self.group_id,
            security_protocol="SASL_SSL",
            sasl_mechanism="AWS_MSK_IAM",
            sasl_plain_username="",
            sasl_plain_password="",
            ssl_context=self._create_ssl_context(),
            # Consumer 설정
            auto_offset_reset="earliest",  # 또는 "latest"
            enable_auto_commit=False,  # 수동 커밋
            max_poll_records=100,  # 한 번에 가져올 최대 레코드 수
            session_timeout_ms=30000,  # 세션 타임아웃
            heartbeat_interval_ms=3000,  # 하트비트 간격
            value_deserializer=lambda m: json.loads(m.decode("utf-8")),
            key_deserializer=lambda k: k.decode("utf-8") if k else None,
            client_id=f"consumer-{uuid4().hex[:8]}",
        )
        
        await self.consumer.start()
        self.running = True
        
        logger.info("MSK Consumer started")
        
        # 메시지 소비 루프 시작
        asyncio.create_task(self._consume_loop())
    
    async def stop(self) -> None:
        """Consumer 종료"""
        logger.info("Stopping MSK Consumer...")
        self.running = False
        
        if self.consumer:
            await self.consumer.stop()
            self.consumer = None
        
        logger.info("MSK Consumer stopped")
    
    def _create_ssl_context(self):
        """SSL 컨텍스트 생성"""
        import ssl
        from aiokafka.helpers import create_ssl_context
        
        context = create_ssl_context()
        context.check_hostname = False
        return context
    
    async def _consume_loop(self) -> None:
        """메시지 소비 루프"""
        try:
            while self.running:
                try:
                    # 메시지 가져오기 (타임아웃 설정)
                    msg_pack = await asyncio.wait_for(
                        self.consumer.getmany(timeout_ms=1000, max_records=100),
                        timeout=2.0
                    )
                    
                    for topic_partition, messages in msg_pack.items():
                        for message in messages:
                            await self._process_message(message)
                    
                    # 수동 커밋
                    await self.consumer.commit()
                    
                except asyncio.TimeoutError:
                    # 타임아웃은 정상 (메시지가 없을 때)
                    continue
                except KafkaError as e:
                    logger.error(f"Kafka error: {e}", exc_info=True)
                    await asyncio.sleep(1)  # 재시도 전 대기
                except Exception as e:
                    logger.error(f"Unexpected error in consume loop: {e}", exc_info=True)
                    await asyncio.sleep(1)
        
        except asyncio.CancelledError:
            logger.info("Consume loop cancelled")
        except Exception as e:
            logger.error(f"Fatal error in consume loop: {e}", exc_info=True)
            self.running = False
    
    async def _process_message(self, message) -> None:
        """메시지 처리"""
        try:
            event_data = message.value
            event_type = event_data.get("event_type")
            
            logger.debug(
                "Processing message",
                topic=message.topic,
                partition=message.partition,
                offset=message.offset,
                event_type=event_type,
            )
            
            # 핸들러 찾기
            handler = self.handlers.get(event_type)
            
            if handler:
                # 핸들러 실행
                await handler(event_data)
                
                logger.info(
                    "Message processed successfully",
                    topic=message.topic,
                    partition=message.partition,
                    offset=message.offset,
                    event_type=event_type,
                )
            else:
                logger.warning(
                    "No handler for event type",
                    event_type=event_type,
                    topic=message.topic,
                )
        
        except Exception as e:
            logger.error(
                "Error processing message",
                topic=message.topic,
                partition=message.partition,
                offset=message.offset,
                error=str(e),
                exc_info=True
            )
            # Dead Letter Queue로 전송하거나 재시도 로직 구현
            raise
```

### 2. Consumer 그룹 관리

```python
# src/infrastructure/messaging/consumer_manager.py

from typing import List, Dict
from src.infrastructure.messaging.consumer import MSKConsumer, EventHandler
from src.core.config import settings
from src.core.logging import logger


class ConsumerManager:
    """여러 Consumer 관리"""
    
    def __init__(self):
        self.consumers: List[MSKConsumer] = []
    
    def create_consumer(
        self,
        topics: List[str],
        group_id: str,
        handlers: Dict[str, EventHandler],
    ) -> MSKConsumer:
        """Consumer 생성 및 등록"""
        consumer = MSKConsumer(
            topics=topics,
            group_id=group_id,
            bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
            region=settings.KAFKA_REGION,
        )
        
        # 핸들러 등록
        for event_type, handler in handlers.items():
            consumer.register_handler(event_type, handler)
        
        self.consumers.append(consumer)
        return consumer
    
    async def start_all(self) -> None:
        """모든 Consumer 시작"""
        logger.info(f"Starting {len(self.consumers)} consumers...")
        
        tasks = [consumer.start() for consumer in self.consumers]
        await asyncio.gather(*tasks)
        
        logger.info("All consumers started")
    
    async def stop_all(self) -> None:
        """모든 Consumer 종료"""
        logger.info(f"Stopping {len(self.consumers)} consumers...")
        
        tasks = [consumer.stop() for consumer in self.consumers]
        await asyncio.gather(*tasks)
        
        logger.info("All consumers stopped")
```

### 3. 이벤트 핸들러 예시

```python
# src/application/handlers/order_handler.py

from typing import Dict, Any
from src.core.logging import logger
from src.domain.services.inventory_service import InventoryService
from src.domain.services.notification_service import NotificationService


async def handle_order_created(event_data: Dict[str, Any]) -> None:
    """주문 생성 이벤트 처리"""
    logger.info("Handling order.created event", event_data=event_data)
    
    order_id = event_data["order_id"]
    items = event_data["items"]
    
    # 재고 차감
    inventory_service = InventoryService()
    for item in items:
        await inventory_service.decrease_stock(
            product_id=item["product_id"],
            quantity=item["quantity"]
        )
    
    # 알림 발송
    notification_service = NotificationService()
    await notification_service.send_order_confirmation(
        user_id=event_data["user_id"],
        order_id=order_id
    )
    
    logger.info("Order created event processed", order_id=order_id)


async def handle_payment_completed(event_data: Dict[str, Any]) -> None:
    """결제 완료 이벤트 처리"""
    logger.info("Handling payment.completed event", event_data=event_data)
    
    order_id = event_data["order_id"]
    
    # 주문 상태 업데이트
    order_service = OrderService()
    await order_service.update_status(order_id, "paid")
    
    logger.info("Payment completed event processed", order_id=order_id)
```

### 4. 애플리케이션 통합

```python
# src/main.py

import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI

from src.infrastructure.messaging.producer_factory import get_producer, close_producer
from src.infrastructure.messaging.consumer_manager import ConsumerManager
from src.application.handlers.order_handler import (
    handle_order_created,
    handle_payment_completed,
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """애플리케이션 생명주기 관리"""
    # 시작 시
    producer = await get_producer()
    
    # Consumer 설정
    consumer_manager = ConsumerManager()
    
    # 주문 이벤트 Consumer
    consumer_manager.create_consumer(
        topics=["order.order.created", "order.order.status-changed"],
        group_id="order-service-consumer",
        handlers={
            "order.created": handle_order_created,
            "order.status-changed": handle_order_status_changed,
        }
    )
    
    # 결제 이벤트 Consumer
    consumer_manager.create_consumer(
        topics=["payment.payment.completed", "payment.payment.failed"],
        group_id="payment-service-consumer",
        handlers={
            "payment.completed": handle_payment_completed,
            "payment.failed": handle_payment_failed,
        }
    )
    
    await consumer_manager.start_all()
    
    yield
    
    # 종료 시
    await consumer_manager.stop_all()
    await close_producer()


app = FastAPI(lifespan=lifespan)
```

---

## 토픽 관리

### 1. 토픽 생성 (AWS CLI)

```bash
# 토픽 생성
aws kafka create-topic \
  --cluster-arn $CLUSTER_ARN \
  --topic-name "order.order.created" \
  --partitions 6 \
  --replication-factor 3 \
  --region ap-northeast-2

# 토픽 목록 조회
aws kafka list-topics \
  --cluster-arn $CLUSTER_ARN \
  --region ap-northeast-2

# 토픽 상세 정보
aws kafka describe-topic \
  --cluster-arn $CLUSTER_ARN \
  --topic-name "order.order.created" \
  --region ap-northeast-2
```

### 2. Python으로 토픽 관리

```python
# src/infrastructure/messaging/topic_manager.py

from typing import List, Dict, Any
from aiokafka import AIOKafkaAdminClient
from aiokafka.admin import NewTopic, ConfigResource, ConfigResourceType
from src.core.config import settings
from src.core.logging import logger


class TopicManager:
    """토픽 관리"""
    
    def __init__(self, bootstrap_servers: str, region: str = "ap-northeast-2"):
        self.bootstrap_servers = bootstrap_servers
        self.region = region
        self.admin_client: Optional[AIOKafkaAdminClient] = None
    
    async def start(self) -> None:
        """Admin 클라이언트 시작"""
        self.admin_client = AIOKafkaAdminClient(
            bootstrap_servers=self.bootstrap_servers,
            security_protocol="SASL_SSL",
            sasl_mechanism="AWS_MSK_IAM",
            sasl_plain_username="",
            sasl_plain_password="",
            ssl_context=self._create_ssl_context(),
        )
        await self.admin_client.start()
        logger.info("Topic manager started")
    
    async def stop(self) -> None:
        """Admin 클라이언트 종료"""
        if self.admin_client:
            await self.admin_client.close()
            self.admin_client = None
            logger.info("Topic manager stopped")
    
    def _create_ssl_context(self):
        """SSL 컨텍스트 생성"""
        import ssl
        from aiokafka.helpers import create_ssl_context
        
        context = create_ssl_context()
        context.check_hostname = False
        return context
    
    async def create_topic(
        self,
        topic_name: str,
        num_partitions: int = 6,
        replication_factor: int = 3,
        config: Optional[Dict[str, str]] = None,
    ) -> None:
        """
        토픽 생성
        
        Args:
            topic_name: 토픽 이름
            num_partitions: 파티션 수
            replication_factor: 복제 팩터 (MSK Serverless는 자동 관리)
            config: 토픽 설정
        """
        if not self.admin_client:
            raise RuntimeError("Topic manager not started")
        
        topic = NewTopic(
            name=topic_name,
            num_partitions=num_partitions,
            replication_factor=replication_factor,
            topic_configs=config or {},
        )
        
        try:
            result = await self.admin_client.create_topics([topic])
            await result[topic_name]
            logger.info(f"Topic created: {topic_name}")
        except Exception as e:
            logger.error(f"Failed to create topic {topic_name}: {e}")
            raise
    
    async def delete_topic(self, topic_name: str) -> None:
        """토픽 삭제"""
        if not self.admin_client:
            raise RuntimeError("Topic manager not started")
        
        try:
            result = await self.admin_client.delete_topics([topic_name])
            await result[topic_name]
            logger.info(f"Topic deleted: {topic_name}")
        except Exception as e:
            logger.error(f"Failed to delete topic {topic_name}: {e}")
            raise
    
    async def list_topics(self) -> List[str]:
        """토픽 목록 조회"""
        if not self.admin_client:
            raise RuntimeError("Topic manager not started")
        
        metadata = await self.admin_client.describe_topics()
        return list(metadata.topics.keys())
    
    async def get_topic_config(self, topic_name: str) -> Dict[str, Any]:
        """토픽 설정 조회"""
        if not self.admin_client:
            raise RuntimeError("Topic manager not started")
        
        config_resource = ConfigResource(
            ConfigResourceType.TOPIC,
            topic_name,
        )
        
        result = await self.admin_client.describe_configs([config_resource])
        configs = await result[config_resource]
        
        return {k: v for k, v in configs.items()}
```

### 3. 토픽 설정 권장사항

```python
# 토픽 설정 예시
topic_config = {
    # 메시지 보관 기간 (7일)
    "retention.ms": "604800000",
    
    # 압축 타입
    "compression.type": "gzip",
    
    # 최대 메시지 크기
    "max.message.bytes": "1048576",  # 1MB
    
    # 세그먼트 크기
    "segment.bytes": "1073741824",  # 1GB
}
```

---

## 모니터링 및 로깅

### 1. CloudWatch 메트릭

MSK Serverless는 자동으로 CloudWatch 메트릭을 제공합니다.

#### 주요 메트릭

| 메트릭 | 설명 |
|--------|------|
| `BytesInPerSec` | 초당 수신 바이트 수 |
| `BytesOutPerSec` | 초당 송신 바이트 수 |
| `MessagesInPerSec` | 초당 수신 메시지 수 |
| `SumOffsetLag` | Consumer lag 합계 |
| `SumPartitionCount` | 파티션 수 합계 |

#### CloudWatch 대시보드 생성

```python
# monitoring/cloudwatch_dashboard.py

import boto3

cloudwatch = boto3.client('cloudwatch')

dashboard_body = {
    "widgets": [
        {
            "type": "metric",
            "properties": {
                "metrics": [
                    ["AWS/Kafka", "BytesInPerSec", {"stat": "Sum"}],
                    [".", "BytesOutPerSec", {"stat": "Sum"}]
                ],
                "period": 300,
                "stat": "Average",
                "region": "ap-northeast-2",
                "title": "Kafka Throughput"
            }
        },
        {
            "type": "metric",
            "properties": {
                "metrics": [
                    ["AWS/Kafka", "MessagesInPerSec", {"stat": "Sum"}]
                ],
                "period": 300,
                "stat": "Average",
                "region": "ap-northeast-2",
                "title": "Message Rate"
            }
        },
        {
            "type": "metric",
            "properties": {
                "metrics": [
                    ["AWS/Kafka", "SumOffsetLag", {"stat": "Sum"}]
                ],
                "period": 300,
                "stat": "Average",
                "region": "ap-northeast-2",
                "title": "Consumer Lag"
            }
        }
    ]
}

cloudwatch.put_dashboard(
    DashboardName="MSK-Serverless-Dashboard",
    DashboardBody=json.dumps(dashboard_body)
)
```

### 2. 애플리케이션 레벨 모니터링

```python
# src/infrastructure/messaging/metrics.py

from typing import Dict
from dataclasses import dataclass
from datetime import datetime
import time

from src.core.logging import logger


@dataclass
class ProducerMetrics:
    """Producer 메트릭"""
    messages_sent: int = 0
    messages_failed: int = 0
    bytes_sent: int = 0
    avg_latency_ms: float = 0.0
    last_sent_at: Optional[datetime] = None


@dataclass
class ConsumerMetrics:
    """Consumer 메트릭"""
    messages_consumed: int = 0
    messages_failed: int = 0
    bytes_consumed: int = 0
    avg_processing_time_ms: float = 0.0
    current_lag: int = 0
    last_consumed_at: Optional[datetime] = None


class MetricsCollector:
    """메트릭 수집기"""
    
    def __init__(self):
        self.producer_metrics: Dict[str, ProducerMetrics] = {}
        self.consumer_metrics: Dict[str, ConsumerMetrics] = {}
    
    def record_producer_success(
        self,
        topic: str,
        bytes_sent: int,
        latency_ms: float,
    ) -> None:
        """Producer 성공 기록"""
        if topic not in self.producer_metrics:
            self.producer_metrics[topic] = ProducerMetrics()
        
        metrics = self.producer_metrics[topic]
        metrics.messages_sent += 1
        metrics.bytes_sent += bytes_sent
        metrics.last_sent_at = datetime.utcnow()
        
        # 평균 지연 시간 계산
        total = metrics.messages_sent + metrics.messages_failed
        metrics.avg_latency_ms = (
            (metrics.avg_latency_ms * (total - 1) + latency_ms) / total
        )
    
    def record_producer_failure(self, topic: str) -> None:
        """Producer 실패 기록"""
        if topic not in self.producer_metrics:
            self.producer_metrics[topic] = ProducerMetrics()
        
        self.producer_metrics[topic].messages_failed += 1
    
    def record_consumer_success(
        self,
        topic: str,
        bytes_consumed: int,
        processing_time_ms: float,
    ) -> None:
        """Consumer 성공 기록"""
        if topic not in self.consumer_metrics:
            self.consumer_metrics[topic] = ConsumerMetrics()
        
        metrics = self.consumer_metrics[topic]
        metrics.messages_consumed += 1
        metrics.bytes_consumed += bytes_consumed
        metrics.last_consumed_at = datetime.utcnow()
        
        # 평균 처리 시간 계산
        total = metrics.messages_consumed + metrics.messages_failed
        metrics.avg_processing_time_ms = (
            (metrics.avg_processing_time_ms * (total - 1) + processing_time_ms) / total
        )
    
    def record_consumer_failure(self, topic: str) -> None:
        """Consumer 실패 기록"""
        if topic not in self.consumer_metrics:
            self.consumer_metrics[topic] = ConsumerMetrics()
        
        self.consumer_metrics[topic].messages_failed += 1
    
    def get_producer_metrics(self, topic: str) -> ProducerMetrics:
        """Producer 메트릭 조회"""
        return self.producer_metrics.get(topic, ProducerMetrics())
    
    def get_consumer_metrics(self, topic: str) -> ConsumerMetrics:
        """Consumer 메트릭 조회"""
        return self.consumer_metrics.get(topic, ConsumerMetrics())
    
    def log_metrics(self) -> None:
        """메트릭 로깅"""
        logger.info("Producer Metrics", metrics=self.producer_metrics)
        logger.info("Consumer Metrics", metrics=self.consumer_metrics)
```

### 3. 알람 설정

```python
# monitoring/alarms.py

import boto3

cloudwatch = boto3.client('cloudwatch')

# Consumer Lag 알람
cloudwatch.put_metric_alarm(
    AlarmName='MSK-High-Consumer-Lag',
    ComparisonOperator='GreaterThanThreshold',
    EvaluationPeriods=2,
    MetricName='SumOffsetLag',
    Namespace='AWS/Kafka',
    Period=300,
    Statistic='Average',
    Threshold=10000,
    ActionsEnabled=True,
    AlarmActions=['arn:aws:sns:ap-northeast-2:123456789012:alerts'],
    AlarmDescription='Alert when consumer lag exceeds 10,000 messages'
)

# 메시지 처리 실패 알람
cloudwatch.put_metric_alarm(
    AlarmName='MSK-High-Error-Rate',
    ComparisonOperator='GreaterThanThreshold',
    EvaluationPeriods=1,
    MetricName='MessagesInPerSec',
    Namespace='Custom/Kafka',
    Period=60,
    Statistic='Sum',
    Threshold=100,
    ActionsEnabled=True,
    AlarmActions=['arn:aws:sns:ap-northeast-2:123456789012:alerts'],
    AlarmDescription='Alert when error rate is high'
)
```

---

## 비용 최적화

### 1. MSK Serverless 비용 구조

MSK Serverless는 다음 기준으로 과금됩니다:

- **프로듀서 쓰기**: GB당 $0.10
- **컨슈머 읽기**: GB당 $0.10
- **스토리지**: GB-월당 $0.10

### 2. 비용 최적화 전략

#### 2.1 메시지 압축

```python
# Producer에서 압축 활성화
producer = AIOKafkaProducer(
    # ...
    compression_type="gzip",  # 또는 "snappy", "lz4"
    # ...
)
```

#### 2.2 배치 발행

```python
# 여러 메시지를 배치로 발행하여 네트워크 오버헤드 감소
async def batch_publish(producer, messages: List[Tuple[str, str, dict]]):
    futures = [
        producer.send(topic, key, value)
        for topic, key, value in messages
    ]
    await asyncio.gather(*futures)
```

#### 2.3 메시지 크기 최적화

```python
# 불필요한 데이터 제거
def optimize_message(event: DomainEvent) -> dict:
    """메시지 크기 최적화"""
    data = event.to_dict()
    
    # 불필요한 필드 제거
    data.pop("internal_metadata", None)
    data.pop("debug_info", None)
    
    return data
```

#### 2.4 토픽 보관 정책

```python
# 오래된 메시지 자동 삭제
topic_config = {
    "retention.ms": "604800000",  # 7일
    "cleanup.policy": "delete",  # 또는 "compact"
}
```

### 3. 비용 모니터링

```python
# CloudWatch 비용 추정
import boto3

cloudwatch = boto3.client('cloudwatch')

# 일일 비용 추정
def estimate_daily_cost():
    # BytesInPerSec + BytesOutPerSec
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/Kafka',
        MetricName='BytesInPerSec',
        StartTime=datetime.utcnow() - timedelta(days=1),
        EndTime=datetime.utcnow(),
        Period=3600,
        Statistics=['Sum']
    )
    
    total_bytes = sum(point['Sum'] for point in response['Datapoints'])
    total_gb = total_bytes / (1024 ** 3)
    
    # 읽기 + 쓰기 비용
    cost = total_gb * 0.10 * 2  # 읽기와 쓰기
    
    return cost
```

---

## 트러블슈팅

### 1. 연결 문제

#### 증상: "Connection refused" 또는 "Timeout"

**원인 및 해결:**

```python
# 1. Bootstrap 서버 확인
bootstrap_servers = "b-1.my-cluster.xxxxx.c2.kafka-serverless.ap-northeast-2.amazonaws.com:9098"

# 2. VPC 및 보안 그룹 확인
# - 애플리케이션이 MSK와 같은 VPC에 있는지 확인
# - 보안 그룹이 올바르게 설정되었는지 확인

# 3. DNS 해석 확인
import socket
host, port = bootstrap_servers.split(":")
try:
    ip = socket.gethostbyname(host)
    print(f"DNS resolved: {host} -> {ip}")
except socket.gaierror as e:
    print(f"DNS resolution failed: {e}")
```

### 2. 인증 문제

#### 증상: "Authentication failed"

**원인 및 해결:**

```python
# 1. IAM 역할/정책 확인
# - 애플리케이션이 올바른 IAM 역할을 사용하는지 확인
# - MSK 클러스터에 대한 권한이 있는지 확인

# 2. 리전 확인
# - 클러스터 리전과 클라이언트 리전이 일치하는지 확인

# 3. 인증 메커니즘 확인
sasl_mechanism = "AWS_MSK_IAM"  # 올바른 메커니즘 사용
```

### 3. Consumer Lag 문제

#### 증상: Consumer가 메시지를 처리하지 못함

**원인 및 해결:**

```python
# 1. Consumer 그룹 상태 확인
from aiokafka import AIOKafkaConsumer

async def check_consumer_lag(consumer: AIOKafkaConsumer):
    """Consumer lag 확인"""
    partitions = consumer.assignment()
    
    for partition in partitions:
        committed = await consumer.committed(partition)
        high_water = await consumer.highwater(partition)
        
        lag = high_water - committed
        print(f"Partition {partition}: Lag = {lag}")
        
        if lag > 10000:
            print(f"WARNING: High lag on {partition}")

# 2. 처리 속도 개선
# - 병렬 처리
# - 배치 처리
# - 파티션 수 증가
```

### 4. 메시지 손실 문제

#### 증상: 발행한 메시지가 소비되지 않음

**원인 및 해결:**

```python
# 1. Producer 설정 확인
producer = AIOKafkaProducer(
    acks="all",  # 모든 복제본에 쓰기 완료 대기
    retries=3,  # 재시도 활성화
    enable_idempotence=True,  # 멱등성 활성화
)

# 2. Consumer offset 확인
# - auto_offset_reset="earliest"로 설정하여 모든 메시지 읽기
# - 수동 커밋 사용
```

### 5. 성능 문제

#### 증상: 처리량이 낮음

**원인 및 해결:**

```python
# 1. 배치 크기 조정
producer = AIOKafkaProducer(
    max_batch_size=32768,  # 32KB로 증가
    linger_ms=50,  # 배치 대기 시간 증가
)

# 2. 압축 활성화
producer = AIOKafkaProducer(
    compression_type="gzip",
)

# 3. 파티션 수 증가
# - 토픽의 파티션 수를 늘려 병렬 처리 증가
```

---

## 베스트 프랙티스

### 1. 메시지 설계

#### 1.1 스키마 버전 관리

```python
# 메시지에 버전 포함
@dataclass
class OrderCreatedEvent(DomainEvent):
    schema_version: str = "1.0"
    # ...
    
    def to_dict(self) -> dict:
        return {
            "schema_version": self.schema_version,
            # ...
        }
```

#### 1.2 멱등성 보장

```python
# 이벤트 ID를 사용한 중복 처리 방지
processed_events = set()

async def handle_event(event_data: dict):
    event_id = event_data["event_id"]
    
    if event_id in processed_events:
        logger.warning(f"Duplicate event: {event_id}")
        return
    
    processed_events.add(event_id)
    # 이벤트 처리
```

### 2. 에러 처리

#### 2.1 Dead Letter Queue

```python
# 실패한 메시지를 DLQ로 전송
async def handle_message_with_dlq(message, handler):
    try:
        await handler(message.value)
    except Exception as e:
        logger.error(f"Failed to process message: {e}")
        
        # DLQ로 전송
        await dlq_producer.send(
            topic="dlq.order.order.created",
            key=message.key,
            value={
                "original_message": message.value,
                "error": str(e),
                "failed_at": datetime.utcnow().isoformat(),
            }
        )
```

#### 2.2 재시도 전략

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
async def process_with_retry(event_data: dict):
    """지수 백오프 재시도"""
    await handle_event(event_data)
```

### 3. 보안

#### 3.1 최소 권한 원칙

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect"
      ],
      "Resource": "arn:aws:kafka:*:*:cluster/*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:WriteData"
      ],
      "Resource": "arn:aws:kafka:*:*:topic/my-cluster/specific-topic/*"
    }
  ]
}
```

#### 3.2 암호화

```python
# 전송 중 암호화 (TLS)
producer = AIOKafkaProducer(
    security_protocol="SASL_SSL",
    # ...
)

# 저장 시 암호화 (MSK Serverless가 자동 처리)
```

### 4. 모니터링

#### 4.1 핵심 메트릭 추적

- Consumer Lag
- 메시지 처리 속도
- 에러율
- 처리 지연 시간

#### 4.2 로깅

```python
# 구조화된 로깅
logger.info(
    "Message published",
    extra={
        "topic": topic,
        "partition": partition,
        "offset": offset,
        "event_type": event_type,
        "event_id": event_id,
    }
)
```

### 5. 테스트

#### 5.1 로컬 테스트

```python
# 로컬 Kafka 사용 (Docker)
# docker-compose.yml
version: '3'
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
```

#### 5.2 통합 테스트

```python
# pytest를 사용한 통합 테스트
@pytest.mark.asyncio
async def test_producer_consumer():
    producer = await create_test_producer()
    consumer = await create_test_consumer()
    
    # 메시지 발행
    await producer.publish("test-topic", "key", {"test": "data"})
    
    # 메시지 소비
    message = await consumer.get_message(timeout=5)
    assert message.value["test"] == "data"
```

---

## 결론

AWS MSK Serverless는 서버 관리 없이 Kafka를 사용할 수 있는 강력한 서비스입니다.
본 가이드에서 다룬 내용을 요약하면 다음과 같습니다:

### 핵심 포인트

1. **자동 확장**: 트래픽에 따라 자동으로 용량이 조정되어 용량 계획이 불필요합니다.
2. **비용 효율**: 사용한 만큼만 지불하는 Pay-as-you-go 모델로 초기 비용 부담이 적습니다.
3. **보안**: IAM 기반 인증과 VPC 격리로 안전한 메시징 환경을 제공합니다.
4. **관리 편의성**: 브로커 및 Zookeeper 관리가 필요 없어 운영 부담이 적습니다.

### 구현 체크리스트

#### 클러스터 설정
```
□ VPC 및 서브넷 구성 완료
□ 보안 그룹 설정 완료
□ MSK Serverless 클러스터 생성 완료
□ IAM 역할 및 정책 설정 완료
□ Bootstrap 서버 주소 확인 완료
```

#### Producer 구현
```
□ IAM 인증 설정 완료
□ Producer 초기화 및 시작 로직 구현 완료
□ 이벤트 발행 로직 구현 완료
□ 에러 처리 및 재시도 로직 구현 완료
□ 메트릭 수집 구현 완료
```

#### Consumer 구현
```
□ Consumer 그룹 설정 완료
□ 이벤트 핸들러 등록 완료
□ 메시지 소비 루프 구현 완료
□ 수동 커밋 로직 구현 완료
□ Dead Letter Queue 처리 구현 완료
```

#### 모니터링 및 운영
```
□ CloudWatch 대시보드 구성 완료
□ 알람 설정 완료
□ 로깅 전략 수립 완료
□ 비용 모니터링 설정 완료
□ 트러블슈팅 가이드 문서화 완료
```

### 다음 단계

1. **로컬 개발 환경 구축**: Docker Compose를 사용한 로컬 Kafka 환경 구성
2. **통합 테스트 작성**: Producer/Consumer 통합 테스트 작성
3. **성능 테스트**: 부하 테스트를 통한 처리량 및 지연 시간 측정
4. **운영 문서화**: 운영 매뉴얼 및 장애 대응 가이드 작성
5. **비용 최적화**: 실제 사용량 분석을 통한 비용 최적화

### 추가 리소스

- [AWS MSK Serverless 공식 문서](https://docs.aws.amazon.com/msk/latest/developerguide/serverless.html)
- [Apache Kafka 공식 문서](https://kafka.apache.org/documentation/)
- [aiokafka 라이브러리 문서](https://aiokafka.readthedocs.io/)
- [AWS MSK Best Practices](https://docs.aws.amazon.com/msk/latest/developerguide/best-practices.html)
- [Kafka 네이밍 컨벤션 가이드](https://www.confluent.io/blog/kafka-topic-naming-conventions/)

### 주의사항

1. **리전 제한**: MSK Serverless는 특정 리전에서만 사용 가능하므로 배포 리전을 확인해야 합니다.
2. **파티션 제한**: 토픽당 최대 1,000개 파티션 제한이 있으므로 토픽 설계 시 고려해야 합니다.
3. **메시지 크기**: 최대 1MB 메시지 크기 제한이 있으므로 대용량 데이터는 S3에 저장하고 참조만 전달하는 것을 권장합니다.
4. **비용 모니터링**: 사용량 기반 과금이므로 CloudWatch를 통해 지속적으로 비용을 모니터링해야 합니다.

---

## 부록

### A. 환경 변수 설정 예시

```bash
# .env 파일
KAFKA_BOOTSTRAP_SERVERS=b-1.my-msk-cluster.xxxxx.c2.kafka-serverless.ap-northeast-2.amazonaws.com:9098
KAFKA_REGION=ap-northeast-2
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=AWS_MSK_IAM
KAFKA_CONSUMER_GROUP_ID=my-service-consumer
KAFKA_AUTO_OFFSET_RESET=earliest
KAFKA_ENABLE_AUTO_COMMIT=false
```

### B. Docker Compose 로컬 개발 환경

```yaml
# docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
```

### C. 유용한 AWS CLI 명령어

```bash
# 클러스터 상태 확인
aws kafka describe-cluster \
  --cluster-arn $CLUSTER_ARN \
  --region ap-northeast-2

# 토픽 목록 조회
aws kafka list-topics \
  --cluster-arn $CLUSTER_ARN \
  --region ap-northeast-2

# Consumer 그룹 목록 조회
aws kafka list-consumer-groups \
  --cluster-arn $CLUSTER_ARN \
  --region ap-northeast-2

# Consumer 그룹 상세 정보
aws kafka describe-consumer-group \
  --cluster-arn $CLUSTER_ARN \
  --consumer-group-id $GROUP_ID \
  --region ap-northeast-2

# 클러스터 삭제
aws kafka delete-cluster \
  --cluster-arn $CLUSTER_ARN \
  --region ap-northeast-2
```

### D. 에러 코드 참조

| 에러 코드 | 설명 | 해결 방법 |
|----------|------|----------|
| `ConnectionRefusedError` | 연결 거부 | VPC 및 보안 그룹 설정 확인 |
| `AuthenticationFailed` | 인증 실패 | IAM 역할 및 정책 확인 |
| `TopicAuthorizationException` | 토픽 권한 없음 | IAM 정책에 토픽 권한 추가 |
| `ConsumerCoordinatorNotAvailableException` | Consumer 그룹 조정자 없음 | Consumer 그룹 재시작 |
| `OffsetOutOfRangeException` | Offset 범위 초과 | `auto_offset_reset` 설정 확인 |

### E. 성능 튜닝 가이드

#### Producer 성능 최적화

```python
# 고성능 Producer 설정
producer = AIOKafkaProducer(
    bootstrap_servers=bootstrap_servers,
    # 배치 설정
    max_batch_size=32768,  # 32KB
    linger_ms=50,  # 배치 대기 시간
    # 압축
    compression_type="gzip",
    # 재시도
    retries=3,
    request_timeout_ms=30000,
    # 버퍼
    buffer_memory=67108864,  # 64MB
)
```

#### Consumer 성능 최적화

```python
# 고성능 Consumer 설정
consumer = AIOKafkaConsumer(
    bootstrap_servers=bootstrap_servers,
    # 배치 처리
    max_poll_records=500,  # 한 번에 가져올 최대 레코드 수
    # 세션 관리
    session_timeout_ms=30000,
    heartbeat_interval_ms=3000,
    # Fetch 설정
    fetch_min_bytes=1048576,  # 1MB
    fetch_max_wait_ms=500,  # 최대 대기 시간
)
```

---

**문서 버전**: 1.0  
**최종 업데이트**: 2025-01-XX  
**작성자**: Development Team 