# Static Website Hosting on AWS (Serverless)
### Overview

This project demonstrates how to host a secure, globally distributed static website on AWS using a fully serverless architecture and Infrastructure as Code (IaC) with CloudFormation.

The solution uses managed AWS services only — no EC2, no servers to patch, and no manual configuration after deployment. It is designed to reflect real-world, production-ready patterns commonly used for marketing websites, documentation portals, and frontend applications.

#### Key goals of this project:
- Demonstrate AWS Solutions Architect fundamentals
- Show end-to-end automation using CloudFormation
- Build a scalable, secure, low-maintenance web hosting solution
- Provide a clean, repeatable deployment process (console + CLI)

### Architecture
#### Services Used

- Amazon S3    
    - Stores and serves static website content (HTML, CSS, JS)
    - Configured with index and error documents

- Amazon CloudFront
    - Global CDN for low-latency content delivery
    - Redirects HTTP → HTTPS
    - Uses S3 as the origin

- AWS Certificate Manager (ACM)
    - Provides an SSL/TLS certificate for the custom domain
    - Certificate is validated using Route 53 DNS records
    - Must be created in us-east-1 for CloudFront

- Amazon Route 53
    - Manages DNS for the custom domain
    - Routes traffic to CloudFront using Alias records
- AWS CloudFormation
    - Provisions the entire infrastructure in a single, repeatable deployment

#### Request Flow

1. User accesses https://example.com
2. Route 53 resolves the domain to the CloudFront distribution
3. CloudFront terminates HTTPS using ACM
4. CloudFront fetches content from the S3 bucket (origin)
5. Cached content is served from the nearest edge location

The following represents the architecture for hosting static website using AWS.

![Static website hosting using AWS S3 architecture](static-website-hosting-architecture.png)

### Deployment Options
You can deploy this project manually via the AWS Console or automatically using the AWS CLI.

### Option 1: Manual Deployment (AWS Console)
#### Step 1: Create Route 53 Hosted Zone
1. Go to Route 53 → Hosted Zones
2. Create a public hosted zone for your domain (e.g. example.com)
3. Update domain name servers at your registrar if required

#### Step 2: Request ACM Certificate
1. Go to ACM (us-east-1 region)
2. Request a public certificate for: example.com
3.Choose DNS validation
4. Allow Route 53 to create validation records automatically
5. Wait for certificate status to become Issued

#### Step 3: Create S3 Bucket
1. Create an S3 bucket named exactly as your domain (e.g. example.com)
2. Enable Static Website Hosting
3. Set:
    - Index document: index.html
    - Error document: error.html
4. Upload your static website files

#### Step 4: Create CloudFront Distribution
1. Create a new CloudFront distribution
2. Set origin to the S3 bucket
3. Set Viewer Protocol Policy to Redirect HTTP to HTTPS
4. Attach the ACM certificate
5. Set default root object to index.html

#### Step 5: Create Route 53 Record
1. Create an A Record (Alias)
2. Point it to the CloudFront distribution
3. Use CloudFront’s hosted zone ID (Z2FDTNDATAQYW2)

### Option 2: Automated Deployment (AWS CLI + CloudFormation)
#### Prerequisites
- AWS CLI configured (aws configure)
- Domain managed in Route 53
- Permissions for CloudFormation, S3, CloudFront, ACM, Route 53

##### Step 1: Clone Repository
```bash
git clone https://github.com/MohnishPophale/AWS-CloudFormation-Templates.git
cd <repo-name>
```
#### Step 2: Review Parameters
```yaml
Parameters:
  DomainName:
    Type: String
    Default: example.com
```
You can override it at deploy time.
#### Step 3: Deploy CloudFormation Stack
```bash
aws cloudformation deploy \
  --template-file static-website-cf-template.yaml \
  --stack-name StaticWebsiteStack \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides DomainName=example.com
```
How parameters work:
- Values passed using `--parameter-overrides` replace `!Ref DomainName`
- No hardcoding is required inside the template
#### Step 4: Upload Website Content
```bash
aws s3 sync ./site s3://example.com/ --delete
```
[aws s3 sync docu](https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html)
#### Step 5: Invalidate CloudFront Cache
```bash
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```
#### Step 6: Validate
- Open https://example.com
- Confirm HTTPS works
- Confirm content loads from CloudFront
### Cleanup (Important)
CloudFormation cannot delete a non-empty S3 bucket.

#### Step 1: Empty the S3 Bucket
```bash
aws s3 rm s3://example.com --recursive
```
#### Step 2: Delete CloudFormation Stack
```bash
aws cloudformation delete-stack \
  --stack-name StaticWebsiteStack
```
### Improvements & Extensions
This project intentionally focuses on fundamentals. Possible next steps:
- Use private S3 bucket + CloudFront Origin Access Control (OAC)
- Add AWS WAF for security
- Add CloudFront Response Headers Policy
- Enable CloudFront access logging
- Add CI/CD pipeline (GitHub Actions / CodePipeline)
- Add Lambda@Edge or CloudFront Functions
- Add monitoring with CloudWatch metrics and alarms

### Why This Architecture Matters

This is a real-world, production-grade pattern used by teams to:
- Launch websites quickly
- Scale globally without servers
- Reduce operational overhead
- Enforce HTTPS by default
- Automate infrastructure reliably

It reflects how AWS Solutions Architects design simple, scalable, and secure systems.

### Architectural Considerations & Trade-offs
#### Service Choice Trade-offs
1. **CloudFront vs ALB** 
    - CloudFront was chosen over an ALB because this workload serves static content and benefits from global edge caching rather than regional load balancing.

2. **Public S3 vs Private S3 + OAC**
    - For simplicity and learning purposes, the S3 bucket is public; in a production setup, this would be replaced with a private bucket using CloudFront Origin Access Control (OAC).
#### Failure Modes
1. **S3 Availability**
    - This architecture relies on Amazon S3’s high availability; for stricter resilience requirements, multi-origin or failover strategies could be introduced.
2. **Certificate Renewal**
    - ACM automatically manages certificate renewal for CloudFront distributions; DNS validation via Route 53 ensures no manual intervention is required.
#### Operational Considerations
1. **Monitoring & Observability**
    - Basic CloudFront and S3 metrics are available via CloudWatch; production systems would extend this with alarms, access logs, and request-level visibility.
2. **Rollback Strategy**
    - Content rollbacks can be performed by reverting S3 object versions or redeploying known-good static assets via the CI/CD pipeline.
#### Security Posture
1. **Public Access Acknowledgement**
    - Public read access is enabled to simplify static hosting; a production-grade design would restrict S3 access to CloudFront only and enforce additional security controls.
### Author
Mohnish Sunil Pophale \
AWS Certified | SDET → Solutions Architect \
[LinkedIn](https://linkedin.com/in/mohnish-pophale)