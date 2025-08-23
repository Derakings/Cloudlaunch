# CloudLaunch AWS Infrastructure Project

This repository contains the implementation of CloudLaunch, a lightweight platform showcasing a basic company website with secure storage for internal documents using AWS core services (S3, IAM, and VPC).

## Task 1: Static Website Hosting Using S3 + IAM User with Limited Permissions

### S3 Buckets Configuration

I created three S3 buckets with different access levels:

1. **Cloudlaunch-site-bucket**
   - Configured for static website hosting
   - Contains HTML, CSS, and font-awesome resources
   - Publicly accessible (read-only)
   - Website URL: [http://cloudluanch--site-bucket.s3-website-us-east-1.amazonaws.com/](http://cloudluanch--site-bucket.s3-website-us-east-1.amazonaws.com)

2. **Cloudlaunch-private-bucket**
   - Not publicly accessible
   - The IAM user has GetObject and PutObject permissions

3. **Cloudlaunch-visible-only-bucket**
   - Not publicly accessible
   - IAM user can only list the bucket but cannot access its contents

### CloudFront Distribution (Bonus)

I configured a CloudFront distribution in front of the site bucket for HTTPS and global content delivery:

- CloudFront URL: [https://d2a2v3p788qry3.cloudfront.net/](https://d2a2v3p788qry3.cloudfront.net/)
- Features:
  - HTTPS security (using default CloudFront certificate)
  - Global content delivery for reduced latency
  - Edge caching for improved performance

### IAM User and Policy

Created an IAM user named with limited permissions to the S3 buckets. The user can:
- List all three buckets
- GetObject/PutObject on cloudlaunch-private-bucket
- GetObject on cloudlaunch-site-bucket
- No DeleteObject permissions anywhere
- No access to cloudlaunch-visible-only-bucket content

**IAM Policy JSON:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListBuckets",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": [
                "arn:aws:s3:::cloudlaunch-site-bucket",
                "arn:aws:s3:::cloudlaunch-private-bucket",
                "arn:aws:s3:::cloudlaunch-visible-only-bucket"
            ]
        },
        {
            "Sid": "AllowGetPutPrivateBucket",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::cloudlaunch-private-bucket/*"
        },
        {
            "Sid": "AllowGetObjectSiteBucket",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::cloudlaunch-site-bucket/*"
        }
    ]
}
```

## Task 2: VPC Design for CloudLaunch Environment

Created a secure, logically separated network environment with the following components:

### VPC Configuration
- **VPC Name**: cloudlaunch-vpc
- **CIDR Block**: 10.0.0.0/16
- **Region**: us-east-1

### Subnet Configuration

- **Public Subnet**:
  - Name: cloudlaunch-public-subnet
  - CIDR: 10.0.1.0/24
  - Availability Zone: us-east-1a
  - Purpose: For future load balancers/public services

- **Application Subnet**:
  - Name: cloudlaunch-app-subnet
  - CIDR: 10.0.2.0/24
  - Availability Zone: us-east-1b
  - Purpose: For application servers (private)

- **Database Subnet**:
  - Name: cloudlaunch-db-subnet
  - CIDR: 10.0.3.0/28
  - Availability Zone: us-east-1a
  - Purpose: For database services (private)

### Internet Gateway
- **Name**: cloudlaunch-igw
- **Status**: Attached to cloudlaunch-vpc
- **Purpose**: Provides internet access to public subnet

### Route Tables

- **Public Route Table (cloudlaunch-public-rt)**:
  - Associated with: cloudlaunch-public-subnet
  - Routes: 
    - 10.0.0.0/16 - Local
    - 0.0.0.0/0 - cloudlaunch-igw (Internet Gateway)

- **Application Route Table (cloudlaunch-app-rt)**:
  - Associated with: cloudlaunch-app-subnet
  - Routes: 
    - 10.0.0.0/16 - Local
    - No internet access (fully private)

- **Database Route Table (cloudlaunch-db-rt)**:
  - Associated with: cloudlaunch-db-subnet
  - Routes: 
    - 10.0.0.0/16 → Local
    - No internet access (fully private)

### Security Groups

- **cloudlaunch-app-sg**:
  - **Description**: Allow HTTP access within VPC only
  - **Inbound Rules**:
    - Type: HTTP (80), Source: 10.0.0.0/16 (VPC CIDR)
  - **Outbound Rules**:
    - Type: HTTP (80), Destination: 0.0.0.0/0
    - Type: HTTPS (443), Destination: 0.0.0.0/0
    - Type: MySQL (3306), Destination: 10.0.3.0/28

- **cloudlaunch-db-sg**:
  - **Description**: Allow MySQL access from app subnet only
  - **Inbound Rules**:
    - Type: MySQL/Aurora (3306), Source: 10.0.2.0/24 (App subnet)
  - **Outbound Rules**:
    - Type: HTTPS (443), Destination: 0.0.0.0/0 (for updates only)

### IAM VPC Read-Only Permissions

Added VPC read-only permissions to the cloudlaunch-user to allow them to view but not modify VPC resources:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CloudLaunchVPCReadOnly",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVpcs",
                "ec2:DescribeSubnets",
                "ec2:DescribeRouteTables",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeInternetGateways",
                "ec2:DescribeNetworkAcls",
                "ec2:DescribeTags"
            ],
            "Resource": "*"
        }
    ]
}
```

## AWS Account Information

- **AWS Console URL**: https://779846823004.signin.aws.amazon.com/console
- **User Password**: [Temporary password - user must change on first login]
- **Access Type**: AWS Management Console access only

### User Credentials Setup

The IAM user has been configured with:
- **Console Password**: Temporary password requiring change on first login
- **Password Requirements**: Minimum 8 characters, mixed case, numbers, symbols
- **Password Reset**: User must reset password upon initial login
- **MFA**: Not configured (can be enabled for additional security)

## Implementation Highlights

### Security Best Practices Implemented

1. **Principle of Least Privilege**: IAM policies grant only necessary permissions
2. **Network Segmentation**: Separate subnets for different tiers (public, app, database)
3. **Security Groups**: Restrictive inbound/outbound rules based on business requirements
4. **Private Subnets**: Application and database subnets have no direct internet access
5. **HTTPS Enforcement**: CloudFront distribution enforces HTTPS for website access

### Architecture Benefits

1. **High Availability**: Resources distributed across multiple Availability Zones
2. **Scalability**: VPC design supports future expansion and load balancing
3. **Security**: Multi-layered security with network and application-level controls
5. **Monitoring Ready**: Architecture supports future implementation of CloudWatch and logging

### Website Features

- **Modern Design**: Professional, responsive website built with HTML and CSS
- **Performance**: Optimized loading with CloudFront CDN integration
- **Accessibility**: Clean typography and color contrast for better user experience
- **Mobile Responsive**: Optimized for viewing on all device types

## Technical Specifications

- **Frontend**: HTML, CSS, Font Awesome icons
- **Hosting**: AWS S3 Static Website Hosting
- **CDN**: AWS CloudFront for global content delivery
- **Network**: Custom AWS VPC with public and private subnets
- **Security**: IAM-based access control with custom policies
- **Availability Zones**: Multi-AZ deployment (us-east-1a, us-east-1b)
