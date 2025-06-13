# Secure Juice Shop Deployment on AWS

This CloudFormation template deploys OWASP Juice Shop, a deliberately vulnerable web application, in a secure AWS environment suitable for security assessment and penetration testing training.

## Architecture Overview

The template creates a comprehensive AWS infrastructure with the following components:

### Network Infrastructure
- **VPC** with configurable CIDR block (default: 10.0.0.0/16)
- **Public Subnets** (2) for load balancer deployment across multiple AZs
- **Private Subnet** (1) for EC2 instances to enhance security
- **Internet Gateway** for public internet access
- **NAT Gateway** for outbound internet access from private subnet
- **Route Tables** properly configured for public and private subnets

### Security Components
- **Application Load Balancer (ALB)** with internet-facing configuration
- **Security Groups** with principle of least privilege
- **AWS WAF** with managed rule sets for SQL injection and common attacks
- **Rate limiting** (100 requests per IP per 5 minutes)
- **CloudTrail** for API logging and audit trail
- **GuardDuty** for threat detection
- **Encrypted EBS volumes** for data at rest encryption
- **MariaDB encryption** at rest and in transit

### Compute and Storage
- **Auto Scaling Group** with single instance configuration
- **Launch Template** with t2.medium instance type
- **Encrypted MariaDB** database for application data
- **AWS Secrets Manager** for database password management

### Monitoring and Management
- **AWS Systems Manager (SSM)** for secure instance access
- **CloudWatch** integration for monitoring
- **IAM roles** with minimal required permissions

## Prerequisites

Before deploying this template, ensure you have:

1. **AWS CLI configured** with appropriate permissions
2. **EC2 Key Pair** created in your target region
3. **Appropriate IAM permissions** to create all resources
4. **VPC quotas** sufficient for the deployment

### Required AWS Permissions
Your IAM user/role needs permissions for:
- EC2 (instances, VPC, security groups, load balancers)
- IAM (roles, policies, instance profiles)
- AutoScaling
- CloudFormation
- CloudTrail
- GuardDuty
- WAF
- Secrets Manager
- SSM
- S3 (for CloudTrail logs)

## Parameters

| Parameter | Description | Default Value | Notes |
|-----------|-------------|---------------|-------|
| `KeyPairName` | EC2 Key Pair for SSH access | `test1` | Must exist in target region |
| `InstanceName` | Name tag for EC2 instance | `VulnerableInstance` | For identification |
| `AMIId` | Amazon Linux 2023 AMI ID | Latest AL2023 | Auto-updated |
| `VpcCidr` | VPC CIDR block | `10.0.0.0/16` | Adjust for your network |
| `PublicSubnetCidr1` | First public subnet CIDR | `10.0.1.0/24` | Must be within VPC CIDR |
| `PublicSubnetCidr2` | Second public subnet CIDR | `10.0.3.0/24` | Must be within VPC CIDR |
| `PrivateSubnetCidr` | Private subnet CIDR | `10.0.2.0/24` | Must be within VPC CIDR |
| `AllowedSSHCidr` | CIDR for SSH access | `0.0.0.0/0` | **Restrict in production** |
| `AllowedWebCidr` | CIDR for web access | `0.0.0.0/0` | Restrict as needed |

## Deployment Instructions

### Using AWS CLI

1. **Clone or download** the CloudFormation template
2. **Deploy the stack**:
   ```bash
   aws cloudformation create-stack \
     --stack-name juice-shop-security-lab \
     --template-body file://juice-shop-template.yaml \
     --parameters ParameterKey=KeyPairName,ParameterValue=your-key-pair \
     --capabilities CAPABILITY_IAM \
     --region us-east-1
   ```

3. **Monitor deployment**:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name juice-shop-security-lab \
     --query 'Stacks[0].StackStatus'
   ```

### Using AWS Console

1. Navigate to **CloudFormation** in AWS Console
2. Click **Create Stack** → **With new resources**
3. Upload the template file
4. Configure parameters (especially `KeyPairName`)
5. Acknowledge IAM resource creation
6. Click **Create Stack**

## Post-Deployment Configuration

### Accessing Juice Shop

1. **Get the Load Balancer URL** from CloudFormation outputs:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name juice-shop-security-lab \
     --query 'Stacks[0].Outputs[?OutputKey==`ALBDNS`].OutputValue' \
     --output text
   ```

2. **Access the application** via HTTP (typically takes 5-10 minutes for full deployment):
   ```
   http://[ALB-DNS-NAME]
   ```

### Connecting to EC2 Instance

The instance is deployed in a private subnet for security. Use AWS Systems Manager Session Manager:

```bash
# List instances in the Auto Scaling Group
aws ec2 describe-instances \
  --filters "Name=tag:aws:autoscaling:groupName,Values=*JuiceShopASG*" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text

# Connect using Session Manager
aws ssm start-session --target [INSTANCE-ID]
```

### Database Access

The MariaDB root password is stored in AWS Secrets Manager:

```bash
# Retrieve the password
aws secretsmanager get-secret-value \
  --secret-id "enpm665/DBRootPassword" \
  --query SecretString \
  --output text
```

## Security Features

### Network Security
- **Private subnet deployment** prevents direct internet access to instances
- **Security groups** restrict traffic to necessary ports only
- **NAT Gateway** provides controlled outbound internet access

### Application Security
- **WAF protection** against common web attacks
- **Rate limiting** prevents DoS attacks
- **HTTPS termination** at load balancer (configure SSL certificate separately)

### Data Security
- **EBS encryption** protects data at rest
- **MariaDB encryption** with file-based key management
- **Secrets Manager** for secure password storage

### Monitoring and Compliance
- **CloudTrail** logs all API activities
- **GuardDuty** provides threat detection
- **VPC Flow Logs** can be enabled for network monitoring

## Customization Options

### Security Hardening

1. **Restrict CIDR blocks**:
   ```yaml
   AllowedWebCidr: "YOUR.OFFICE.IP/32"
   AllowedSSHCidr: "YOUR.OFFICE.IP/32"
   ```

2. **Enable additional WAF rules**:
   - Add IP reputation lists
   - Configure geo-blocking
   - Add custom rules for your use case

3. **Configure SSL/TLS**:
   - Add ACM certificate to ALB
   - Redirect HTTP to HTTPS
   - Configure security headers

### Scaling Configuration

```yaml
# Modify Auto Scaling Group parameters
MinSize: '2'
MaxSize: '5'
DesiredCapacity: '2'
```

### Instance Type Adjustment

```yaml
# In Launch Template
InstanceType: t3.large  # For better performance
```

## Monitoring and Troubleshooting

### Log Locations

- **User Data logs**: `/var/log/user_data.log`
- **Juice Shop logs**: Check with `sudo journalctl -u juice-shop`
- **MariaDB logs**: `/var/log/mariadb/mariadb.log`

### Common Issues

1. **Application not accessible**:
   - Check Auto Scaling Group health
   - Verify target group health in ALB
   - Review security group rules

2. **Database connection issues**:
   - Verify MariaDB service status
   - Check Secrets Manager access
   - Review IAM permissions

3. **SSM connection fails**:
   - Confirm SSM agent is running
   - Verify instance profile permissions
   - Check VPC endpoints if using private connectivity

### Health Checks

```bash
# Check application status
curl -I http://[ALB-DNS-NAME]

# Verify database
mysql -u root -p[PASSWORD] -e "SHOW DATABASES;"

# Check services
systemctl status mariadb
systemctl status amazon-ssm-agent
```

## Cost Considerations

### Estimated Monthly Costs (us-east-1)
- **EC2 t2.medium**: ~$30/month
- **Application Load Balancer**: ~$20/month
- **NAT Gateway**: ~$32/month
- **EBS storage (20GB)**: ~$2/month
- **Other services**: ~$10/month
- **Total**: ~$94/month

### Cost Optimization
- Use **t3.micro** for development (free tier eligible)
- Consider **NAT Instance** instead of NAT Gateway for lower costs
- Enable **CloudWatch billing alerts**

## Security Best Practices

### For Production Use
1. **Never use default passwords**
2. **Restrict CIDR blocks** to known IP ranges
3. **Enable MFA** for AWS console access
4. **Regular security updates** of instances
5. **Monitor CloudTrail logs** regularly
6. **Configure backup strategies**

### For Security Testing
1. **Use dedicated AWS account** for testing
2. **Set up proper boundaries** and access controls
3. **Document all testing activities**
4. **Clean up resources** after testing
5. **Follow responsible disclosure** practices

## Cleanup

To avoid ongoing charges, delete the stack when finished:

```bash
aws cloudformation delete-stack --stack-name juice-shop-security-lab
```

**Note**: Some resources like S3 buckets (CloudTrail logs) have retention policies and may need manual deletion.

## Support and Troubleshooting

### Resource Limits
Ensure your AWS account has sufficient limits for:
- VPC (5 per region default)
- Security Groups (60 per VPC default)
- Load Balancers (50 per region default)

### Common CloudFormation Errors
- **InsufficientCapabilities**: Add `--capabilities CAPABILITY_IAM`
- **KeyPair not found**: Verify key pair exists in target region
- **Subnet conflicts**: Ensure CIDR blocks don't overlap

## License and Disclaimer

This template is provided for educational and security testing purposes. OWASP Juice Shop is intentionally vulnerable and should never be deployed in a production environment without proper security controls.

- **OWASP Juice Shop**: [MIT License](https://github.com/bkimminich/juice-shop/blob/master/LICENSE)
- **This Template**: Use at your own risk, following AWS best practices

## Contributing

To improve this template:
1. Fork the repository
2. Make your changes
3. Test thoroughly
4. Submit a pull request with detailed description

---

**⚠️ Important**: This deployment creates intentionally vulnerable infrastructure for security testing. Always use in isolated environments and follow your organization's security policies.