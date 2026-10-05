# CloudDataFlow --- An Event-Driven AWS Data Processing Pipeline

CloudDataFlow is a serverless/event-driven data processing application
built on AWS.

Users upload a CSV file through a web application. The file is stored in
Amazon S3, which automatically triggers an AWS Lambda function. Lambda
processes the CSV, stores the structured data in Amazon RDS for MySQL,
and saves the processed CSV back to S3. The web application then
provides the user with a temporary presigned download URL.

## Demo

**Demo video:** `demo-video/CloudDataFlow-Demo.mp4`

The final application flow is:

``` text
                         USER
                           │
                           ▼
                    Application
                  Load Balancer
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
              EC2 #1              EC2 #2
             Node.js             Node.js
                 │                   │
                 └─────────┬─────────┘
                           ▼
                      Amazon S3
                       input/
                           │
                    ObjectCreated Event
                           │
                           ▼
                    AWS Lambda
                    Python 3.x
                     /       \
                    ▼         ▼
             RDS MySQL     S3 processed/
                                │
                                ▼
                         Presigned URL
                                │
                                ▼
                              USER
```

## Key Features

-   CSV upload through a Node.js/Express web application
-   Application Load Balancer for the web tier
-   Auto Scaling Group with two EC2 instances across Availability Zones
-   Amazon S3 for input and processed files
-   Event-driven S3 → Lambda invocation
-   Python Lambda function for CSV processing
-   RDS MySQL for structured data storage
-   S3 presigned URLs for temporary processed-file downloads
-   IAM roles with bucket-specific S3 permissions
-   VPC with public and private subnets
-   Internet Gateway for public subnet connectivity
-   NAT Gateway for private subnet outbound connectivity
-   Private RDS database with security-group-controlled access

## AWS Architecture

### Network

The project uses a custom VPC:

``` text
VPC: CloudDataFlow-VPC
CIDR: 10.0.0.0/16

Public-A     10.0.1.0/24    ap-south-1a
Private-A    10.0.2.0/24    ap-south-1a
Public-B     10.0.3.0/24    ap-south-1b
Private-B    10.0.4.0/24    ap-south-1b
```

Public subnets use the Internet Gateway. Private subnets use the NAT
Gateway for outbound internet access.

### Application flow

1.  User opens the application through the Application Load Balancer.
2.  ALB distributes requests across the EC2 instances managed by the
    Auto Scaling Group.
3.  The Node.js application uploads the CSV to the S3 `input/` prefix.
4.  S3 generates an ObjectCreated event.
5.  The event invokes the Python Lambda function.
6.  Lambda downloads and processes the CSV.
7.  Lambda inserts the processed records into the private RDS MySQL
    database.
8.  Lambda writes the processed CSV to the S3 `processed/` prefix.
9.  The web application detects that the processed object exists.
10. The application generates a temporary S3 presigned URL.
11. The user downloads the processed CSV without the S3 bucket being
    public.

## AWS Services Used

  Service                     Purpose
  --------------------------- --------------------------------------------
  Amazon VPC                  Network isolation and architecture
  Subnets                     Public/private network separation
  Internet Gateway            Internet connectivity for public subnets
  NAT Gateway                 Outbound connectivity from private subnets
  Application Load Balancer   Distributes web traffic
  EC2                         Hosts the Node.js web application
  Auto Scaling Group          Maintains two application instances
  Amazon S3                   Stores input and processed CSV files
  AWS Lambda                  Event-driven CSV processing
  Amazon RDS for MySQL        Stores structured processed data
  IAM                         Service roles and access control

## Data Processing Example

Input:

``` csv
name,age,city,salary
Rahul,23,Pune,45000
Amit,25,Mumbai,52000
Priya,22,Nagpur,48000
Sneha,24,Pune,55000
```

Lambda adds a calculated `salary_after_bonus` field.

Output:

``` csv
name,age,city,salary,salary_after_bonus
Rahul,23,Pune,45000,49500
Amit,25,Mumbai,52000,57200
Priya,22,Nagpur,48000,52800
Sneha,24,Pune,55000,60500
```

The same processed records are inserted into the `employee_data` table
in RDS MySQL.

Example query:

``` sql
USE clouddataflow;

SELECT * FROM employee_data;
```

## S3 Structure

``` text
clouddataflow-prathmesh-2026/
│
├── input/
│   └── uploaded CSV files
│
└── processed/
    └── processed CSV files
```

The Lambda trigger is configured for:

``` text
Event: ObjectCreated
Prefix: input/
Suffix: .csv
```

Processed files are written to `processed/`, preventing them from
triggering the Lambda function again.

## IAM Design

The project uses IAM roles instead of hard-coded AWS access keys.

### Lambda role

`CloudDataFlow-Lambda-Role`

Used by Lambda for:

-   CloudWatch logging
-   VPC networking
-   Reading input objects from the project S3 bucket
-   Writing processed objects to the project S3 bucket

### EC2 role

`CloudDataFlow-EC2-Role`

Used by the Node.js application for:

-   Uploading objects to `input/`
-   Reading objects from `processed/`
-   Checking for processed files

S3 access is restricted to the project bucket and relevant prefixes.

## Security

-   RDS has public access disabled.
-   RDS is placed in private subnets.
-   Security groups restrict traffic between application components.
-   S3 Block Public Access remains enabled.
-   Processed files remain private in S3.
-   Users receive temporary presigned download URLs.
-   EC2 uses an IAM instance role instead of storing AWS access keys in
    the application.
-   Lambda uses an IAM execution role.
-   SSH access is restricted to the administrator's IP.

## Application Technology

### Backend

-   Node.js
-   Express.js
-   Multer
-   AWS SDK for JavaScript

### Processing

-   Python
-   AWS Lambda
-   PyMySQL

### Database

-   MySQL
-   Amazon RDS

## Project Screenshots

Suggested evidence to include in the repository:

``` text
screenshots/
├── 01-vpc.png
├── 02-subnets.png
├── 03-public-route-table.png
├── 04-private-route-table.png
├── 05-nat-gateway.png
├── 06-ec2-asg.png
├── 07-target-group.png
├── 08-alb.png
├── 09-s3-bucket.png
├── 10-s3-processed.png
├── 11-lambda.png
├── 12-s3-trigger.png
├── 13-rds.png
├── 14-rds-data.png
└── 15-iam.png
```

Do not upload passwords, access keys, private keys, database
credentials, or other secrets.

## Project Outcome

The project demonstrates an end-to-end event-driven AWS workflow:

``` text
CSV Upload
    ↓
Application Load Balancer
    ↓
EC2 + Auto Scaling
    ↓
S3
    ↓
S3 Event
    ↓
Lambda
    ├──→ RDS MySQL
    └──→ S3 Processed File
              ↓
       Presigned Download
```

The architecture separates the web tier, file storage, event-driven
processing, and database layer while using IAM and VPC security
controls.

## Cleanup

This project is intended as a temporary AWS learning and portfolio
project.

After recording the demo and capturing screenshots, remove unused
resources to avoid unnecessary AWS charges.

Recommended cleanup order:

1.  Delete/terminate Auto Scaling instances through the Auto Scaling
    Group.
2.  Delete the Auto Scaling Group.
3.  Delete the Application Load Balancer and target groups.
4.  Delete the EC2 Launch Template versions if no longer needed.
5.  Delete the RDS database after taking any required evidence.
6.  Remove S3 objects and delete the S3 bucket.
7.  Delete the Lambda function and layers.
8.  Delete Lambda IAM policies/roles if no longer needed.
9.  Delete EC2 IAM policies/role if no longer needed.
10. Delete the NAT Gateway.
11. Release any Elastic IPs associated with the project.
12. Delete route tables, subnets, and the Internet Gateway.
13. Delete the VPC.
14. Verify that no billable project resources remain.

## Author

**Prathmesh**

B.Tech Computer Science & Engineering

AWS / Cloud / DevOps Learning Project

------------------------------------------------------------------------

## License

This project is intended primarily as a learning and portfolio project.
