<div align="center">
  
![AWS PrivateLink](https://img.icons8.com/color/96/amazon-web-services.png)

## Secure Cross-Account RDS Access with Reader and Writer Endpoints

**Updated: January 2, 2026**

[![Follow @nicoleepaixao](https://img.shields.io/github/followers/nicoleepaixao?label=Follow&style=social)](https://github.com/nicoleepaixao)
[![Star this repo](https://img.shields.io/github/stars/nicoleepaixao/aws-rds-privatelink-nlb?style=social)](https://github.com/nicoleepaixao/aws-rds-privatelink-nlb)

</div>

---

<p align="center">
  <img src="img/aws-cross-account-rds-privatelink.png" alt="AWS Private Link Architecture" width="1200">
</p>

## **Overview**

This repository documents an end-to-end pattern to expose **RDS Aurora/MySQL writer and reader endpoints** from one AWS account (provider) to another AWS account (consumer) using Internal Network Load Balancer (NLB), IP-based Target Groups, separate listener ports, and AWS PrivateLink. All traffic remains **inside AWS' private network** with no internet exposure, VPC peering, or VPN complexity.

---

## **Important Information**

### **What This Solution Does**

| **Aspect** | **Details** |
|------------|-------------|
| **Architecture** | Cross-account RDS access via PrivateLink |
| **Traffic Flow** | Provider NLB → RDS endpoints → Consumer VPC endpoint |
| **Port Separation** | Reader (3306) and Writer (3307) on different ports |
| **Security** | No internet exposure, private IPs only |
| **No Complexity** | No VPC peering, VPN, or routing tables |
| **Managed Service** | AWS PrivateLink handles connectivity |

### **Why This Matters**

Traditional cross-account database access requires:

- **VPC Peering**: Complex routing and CIDR conflicts
- **VPN Connections**: Additional cost and maintenance
- **Public Endpoints**: Security risk and compliance concerns
- **Transit Gateway**: Expensive for simple use cases
- **Manual DNS Management**: Error-prone and fragile

### **Solution Benefits**

- **Zero Internet Exposure**: All traffic stays on AWS backbone
- **No VPC Peering**: Simplified network architecture
- **Clear Separation**: Provider controls access, consumer controls usage
- **Port-Based Routing**: Reader and writer on different ports
- **AWS Managed**: PrivateLink handles connectivity automatically
- **Scalable**: Add more consumers without provider-side changes

---

## **Architecture**

### **High-Level Flow**

<p align="center">
  <img src="img/aws-cross-account-rds-privatelink.png" alt="AWS Private Link Architecture" width="800">
</p>

</div>

### **Key Design Principles**

- **Consumer only sees NLB/PrivateLink DNS**: RDS endpoints remain private
- **Port 3306 on NLB**: Maps to reader endpoint
- **Port 3307 on NLB**: Maps to writer endpoint
- **IP-based target groups**: Direct IP registration for RDS endpoints
- **Security Groups**: Fine-grained access control at every layer

---

## **Use Cases**

| **Use Case** | **Application** |
|--------------|-----------------|
| **AWS Glue Jobs** | Read from RDS hosted in another account |
| **Cross-Account Analytics** | BI tools accessing centralized databases |
| **Multi-Account Architecture** | Strict network isolation with controlled access |
| **Partner Data Sharing** | Share databases without public exposure |
| **Data Platform** | Centralized data lake accessing source databases |
| **Disaster Recovery** | Cross-account database replication |

---

## **Prerequisites**

### **Provider Account**

| **Requirement** | **Purpose** |
|-----------------|-------------|
| RDS Aurora/MySQL Cluster | Database to be shared |
| VPC with private subnets | NLB deployment |
| Security Groups | Control access to RDS and NLB |
| IAM Permissions | Create NLB, Target Groups, PrivateLink |

### **Consumer Account**

| **Requirement** | **Purpose** |
|-----------------|-------------|
| VPC with private subnets | VPC Endpoint deployment |
| Security Groups | Control access from applications |
| IAM Permissions | Create VPC Endpoints |

---

## **Implementation Guide**

## **Step 1: Discover RDS Endpoints**

### **List All DB Clusters**

```bash
aws rds describe-db-clusters \
  --query "DBClusters[].{Cluster:DBClusterIdentifier,Writer:Endpoint,Reader:ReaderEndpoint}" \
  --output table
```

### **Set Cluster ID**

```bash
CLUSTER_ID=<your-db-cluster-id>
```

### **Get Writer Endpoint**

```bash
WRITER=$(aws rds describe-db-clusters \
  --db-cluster-identifier "$CLUSTER_ID" \
  --query "DBClusters[0].Endpoint" \
  --output text)

echo "Writer endpoint: $WRITER"
```

### **Get Reader Endpoint**

```bash
READER=$(aws rds describe-db-clusters \
  --db-cluster-identifier "$CLUSTER_ID" \
  --query "DBClusters[0].ReaderEndpoint" \
  --output text)

echo "Reader endpoint: $READER"
```

---

## **Step 2: Resolve Endpoints to Private IPs**

Target groups with `target-type=ip` require IP addresses. Run from an environment that can resolve RDS DNS (EC2 in same VPC, or via VPN/DirectConnect).

### **Resolve Writer IP**

```bash
WRITER_IP=$(dig +short "$WRITER")
echo "Writer IP: $WRITER_IP"
```

### **Resolve Reader IP**

```bash
READER_IP=$(dig +short "$READER")
echo "Reader IP: $READER_IP"
```

**Alternative (Linux):**

```bash
getent hosts "$WRITER"
getent hosts "$READER"
```

**Note:** You must see private IPs within the provider VPC CIDR range.

---

## **Step 3: Configure Security Groups**

### **RDS Security Group** (`sg-rds-privatelink`)

| **Direction** | **Type** | **Protocol** | **Port** | **Source/Destination** |
|---------------|----------|--------------|----------|------------------------|
| Inbound | MySQL/Aurora | TCP | 3306 | sg-nlb-rds-privatelink |
| Outbound | All traffic | All | All | 0.0.0.0/0 |

### **NLB Security Group** (`sg-nlb-rds-privatelink`)

| **Direction** | **Type** | **Protocol** | **Port** | **Source/Destination** |
|---------------|----------|--------------|----------|------------------------|
| Inbound | MySQL/Aurora | TCP | 3306-3307 | Provider VPC CIDR |
| Outbound | MySQL/Aurora | TCP | 3306 | sg-rds-privatelink |

### **Consumer Endpoint Security Group** (`sg-consumer-privatelink`)

| **Direction** | **Type** | **Protocol** | **Port** | **Source/Destination** |
|---------------|----------|--------------|----------|------------------------|
| Outbound | MySQL/Aurora | TCP | 3306-3307 | 0.0.0.0/0 |

---

## **Step 4: Create Target Groups**

### **Reader Target Group (Port 3306)**

```bash
TG_READER_ARN=$(aws elbv2 create-target-group \
  --name tg-rds-reader-privatelink \
  --protocol TCP \
  --port 3306 \
  --target-type ip \
  --vpc-id <PROVIDER_VPC_ID> \
  --health-check-protocol TCP \
  --health-check-port traffic-port \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

echo "Reader TG ARN: $TG_READER_ARN"
```

**Register Reader IP:**

```bash
aws elbv2 register-targets \
  --target-group-arn "$TG_READER_ARN" \
  --targets Id="$READER_IP",Port=3306
```

### **Writer Target Group (Port 3307)**

```bash
TG_WRITER_ARN=$(aws elbv2 create-target-group \
  --name tg-rds-writer-privatelink \
  --protocol TCP \
  --port 3307 \
  --target-type ip \
  --vpc-id <PROVIDER_VPC_ID> \
  --health-check-protocol TCP \
  --health-check-port traffic-port \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

echo "Writer TG ARN: $TG_WRITER_ARN"
```

**Register Writer IP:**

```bash
aws elbv2 register-targets \
  --target-group-arn "$TG_WRITER_ARN" \
  --targets Id="$WRITER_IP",Port=3307
```

---

## **Step 5: Create Internal Network Load Balancer**

```bash
NLB_ARN=$(aws elbv2 create-load-balancer \
  --name nlb-rds-privatelink \
  --type network \
  --scheme internal \
  --subnets <SUBNET_ID_1> <SUBNET_ID_2> <SUBNET_ID_3> \
  --security-groups <SG_NLB_ID> \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text)

echo "NLB ARN: $NLB_ARN"
```

**Get NLB DNS:**

```bash
aws elbv2 describe-load-balancers \
  --load-balancer-arns "$NLB_ARN" \
  --query "LoadBalancers[0].DNSName" \
  --output text
```

---

## **Step 6: Create Listeners**

### **Reader Listener (TCP 3306)**

```bash
aws elbv2 create-listener \
  --load-balancer-arn "$NLB_ARN" \
  --protocol TCP \
  --port 3306 \
  --default-actions Type=forward,TargetGroupArn="$TG_READER_ARN"
```

### **Writer Listener (TCP 3307)**

```bash
aws elbv2 create-listener \
  --load-balancer-arn "$NLB_ARN" \
  --protocol TCP \
  --port 3307 \
  --default-actions Type=forward,TargetGroupArn="$TG_WRITER_ARN"
```

**Validation:**
- `<NLB_DNS>:3306` → forwards to reader endpoint
- `<NLB_DNS>:3307` → forwards to writer endpoint

---

## **Step 7: Create VPC Endpoint Service (Provider)**

### **Create Endpoint Service**

```bash
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns "$NLB_ARN" \
  --acceptance-required
```

**With custom Private DNS (optional):**

```bash
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns "$NLB_ARN" \
  --acceptance-required \
  --private-dns-name rds-privatelink.example.com
```

### **Get Service Information**

```bash
SERVICE_ID=$(aws ec2 describe-vpc-endpoint-service-configurations \
  --query "ServiceConfigurations[0].ServiceId" \
  --output text)

SERVICE_NAME=$(aws ec2 describe-vpc-endpoint-service-configurations \
  --query "ServiceConfigurations[0].ServiceName" \
  --output text)

echo "Service ID:   $SERVICE_ID"
echo "Service Name: $SERVICE_NAME"
```

**Note:** Share the `SERVICE_NAME` with the consumer account.

---

## **Step 8: Restrict Access (Allow Principals)**

Restrict access so only the consumer account can create endpoints:

```bash
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id "$SERVICE_ID" \
  --add-allowed-principals arn:aws:iam::<CONSUMER_ACCOUNT_ID>:root
```

---

## **Step 9: Create Interface VPC Endpoint (Consumer)**

**Run in consumer account:**

```bash
aws ec2 create-vpc-endpoint \
  --vpc-endpoint-type Interface \
  --vpc-id <CONSUMER_VPC_ID> \
  --service-name "$SERVICE_NAME" \
  --subnet-ids <CONSUMER_SUBNET_1> <CONSUMER_SUBNET_2> \
  --security-group-ids <SG_CONSUMER_ENDPOINT_ID>
```

This returns a `VpcEndpointId` (e.g., `vpce-0abcd1234...`).

---

## **Step 10: Approve Connection (Provider)**

**Back in provider account:**

### **List Pending Connections**

```bash
aws ec2 describe-vpc-endpoint-connections \
  --service-id "$SERVICE_ID" \
  --query "VpcEndpointConnections[].VpcEndpointId" \
  --output text
```

### **Approve Connection**

```bash
aws ec2 accept-vpc-endpoint-connections \
  --service-id "$SERVICE_ID" \
  --vpc-endpoint-ids <VPCE_ID_FROM_ABOVE>
```

After approval, the endpoint becomes **Available** and traffic flows.

---

## **Testing Connectivity**

### **From Consumer VPC (EC2 or Glue)**

**Test Reader Connection:**

```bash
nc -vz <PRIVATELINK_DNS> 3306
```

**Test Writer Connection:**

```bash
nc -vz <PRIVATELINK_DNS> 3307
```

### **Database Connectivity**

**Reader:**

```bash
mysql -h <PRIVATELINK_DNS> -P 3306 -u <user> -p
```

**Writer:**

```bash
mysql -h <PRIVATELINK_DNS> -P 3307 -u <user> -p
```

### **AWS Glue JDBC URL**

```text
jdbc:mysql://<PRIVATELINK_DNS>:3306/<database>  # Reader
jdbc:mysql://<PRIVATELINK_DNS>:3307/<database>  # Writer
```

---

## **Operational Considerations**

### **RDS Failover Handling**

When Aurora/MySQL cluster fails over, the underlying IP addresses can change.

**Challenge:** IP-based target groups need IP updates after failover.

**Solution:** Implement automated IP monitoring and updates.

### **IP Update Automation Script**

```python
#!/usr/bin/env python3
import boto3

def update_target_ips():
    rds = boto3.client('rds')
    elbv2 = boto3.client('elbv2')
    
    # Get current RDS IPs
    cluster = rds.describe_db_clusters(DBClusterIdentifier='<CLUSTER_ID>')
    writer_endpoint = cluster['DBClusters'][0]['Endpoint']
    reader_endpoint = cluster['DBClusters'][0]['ReaderEndpoint']
    
    # Resolve to IPs (implement DNS resolution)
    writer_ip = resolve_dns(writer_endpoint)
    reader_ip = resolve_dns(reader_endpoint)
    
    # Get current target group IPs
    reader_targets = elbv2.describe_target_health(
        TargetGroupArn='<TG_READER_ARN>'
    )
    writer_targets = elbv2.describe_target_health(
        TargetGroupArn='<TG_WRITER_ARN>'
    )
    
    # Compare and update if changed
    # (implement comparison and registration logic)
```

**Deployment Options:**
- **Lambda function** triggered by EventBridge (every 5 minutes)
- **Cron job** on EC2 instance
- **Step Functions** workflow

---

## **Security Best Practices**

| **Practice** | **Implementation** |
|--------------|-------------------|
| **Least Privilege** | Use dedicated security groups for each component |
| **Network Isolation** | Keep RDS in private subnets only |
| **Encryption in Transit** | Enable SSL/TLS for database connections |
| **Audit Logging** | Enable VPC Flow Logs and CloudTrail |
| **IP Whitelisting** | Restrict consumer endpoint SG to known CIDRs |
| **Regular Review** | Audit VPC endpoint connections monthly |

---

## **Features**

| **Feature** | **Description** |
|-------------|-----------------|
| **Cross-Account Access** | Secure RDS sharing without VPC peering |
| **Port Separation** | Reader (3306) and Writer (3307) isolation |
| **Private Connectivity** | No internet exposure required |
| **Managed Service** | AWS PrivateLink handles routing |
| **Scalable** | Add multiple consumer accounts easily |
| **Flexible** | Works with Aurora, MySQL, PostgreSQL |
| **Secure** | Security groups at every layer |

---

## **Use Cases Summary**

- ✅ AWS Glue jobs reading from cross-account RDS
- ✅ Cross-account analytics and BI tools
- ✅ Multi-account architectures with strict isolation
- ✅ Sharing databases with partner AWS accounts
- ✅ Centralized data platforms accessing source databases
- ✅ Disaster recovery cross-account replication

---

## **Technologies Used**

| **Technology** | **Purpose** |
|----------------|-------------|
| AWS RDS Aurora/MySQL | Source database |
| Network Load Balancer | Traffic routing and load balancing |
| AWS PrivateLink | Secure cross-account connectivity |
| VPC Endpoint Service | Provider-side service configuration |
| Interface VPC Endpoint | Consumer-side connection |
| Security Groups | Network access control |
| Target Groups | IP-based routing to RDS endpoints |

---

## **Project Structure**

```text
aws-rds-privatelink-nlb/
│
├── README.md                      # Complete implementation guide
│
├── scripts/
│   ├── setup-provider.sh          # Provider account setup automation
│   ├── setup-consumer.sh          # Consumer account setup automation
│   └── monitor-failover.py        # IP update automation script
│
└── docs/
    ├── troubleshooting.md         # Common issues and solutions
    └── architecture-diagram.png   # Visual architecture reference
```

---

## **Additional Information**

For more details about AWS PrivateLink, NLB, and RDS best practices, refer to:

- [AWS PrivateLink Documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/) - Official guide
- [Network Load Balancer Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/) - NLB configuration
- [RDS Aurora Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.BestPractices.html) - Database optimization
- [VPC Endpoint Services](https://docs.aws.amazon.com/vpc/latest/privatelink/create-endpoint-service.html) - PrivateLink setup

---

## **Future Enhancements**

| **Feature** | **Description** | **Status** |
|-------------|-----------------|------------|
| Automated Failover | Lambda-based IP update on RDS failover | In Development |
| Multi-Region Support | Cross-region PrivateLink configuration | Planned |
| Monitoring Dashboard | CloudWatch metrics visualization | Future |
| Cost Tracking | PrivateLink and NLB cost analysis | Planned |
| PostgreSQL Support | Extend pattern to RDS PostgreSQL | Future |
| Health Check Automation | Automated target group health monitoring | Planned |

---

## **Connect & Follow**

Stay updated with AWS networking and PrivateLink best practices:

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/nicoleepaixao)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?logo=linkedin&logoColor=white&style=for-the-badge)](https://www.linkedin.com/in/nicolepaixao/)
[![Medium](https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white)](https://medium.com/@nicoleepaixao)

</div>

---

## **Disclaimer**

This implementation guide provides a production-ready pattern for cross-account RDS access via PrivateLink. AWS service configurations, pricing, and features may vary. Always test in non-production environments before deploying to production. Implement proper monitoring for RDS failover scenarios to ensure IP target groups remain synchronized. Consult official AWS documentation for the most current information.

---

<div align="center">

**Happy building secure AWS architectures!**

*Document last updated: January 2, 2026*

</div>