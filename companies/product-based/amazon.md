# Amazon Interview Questions

Complete preparation guide for Amazon SDE, DevOps, and Cloud roles.

---

## Interview Process

| Round | Duration | Focus |
|-------|----------|-------|
| Online Assessment | 90 min | 2 coding problems + Work Simulation |
| Phone Screen | 60 min | Coding + Leadership Principles |
| Loop Round 1 | 60 min | DSA + LP |
| Loop Round 2 | 60 min | System Design + LP |
| Loop Round 3 | 60 min | DSA + LP |
| Loop Round 4 | 60 min | Bar Raiser (any topic) |

---

## Amazon's 16 Leadership Principles

Every interview includes behavioral questions tied to Leadership Principles:

| Principle | What They Look For |
|-----------|-------------------|
| Customer Obsession | Decisions start with the customer |
| Ownership | Think long-term, act on behalf of the company |
| Invent and Simplify | Innovation and simplification |
| Are Right, A Lot | Good judgment and instincts |
| Learn and Be Curious | Always learning |
| Hire and Develop the Best | Raise the performance bar |
| Insist on the Highest Standards | Continually raising the bar |
| Think Big | Bold directions that inspire |
| Bias for Action | Speed matters, calculated risk-taking |
| Frugality | Do more with less |
| Earn Trust | Listen, speak candidly, treat others respectfully |
| Dive Deep | Stay connected to details |
| Have Backbone; Disagree and Commit | Challenge decisions respectfully |
| Deliver Results | Focus on key inputs, deliver with quality |
| Strive to be Earth's Best Employer | Work environment for success |
| Success and Scale Bring Broad Responsibility | Do better every day |

---

## DevOps/Cloud Interview Questions

### Question 1: AWS - Lambda Timeout in VPC

**Scenario:** Lambda functions are timing out when accessing RDS in a VPC. How do you debug this?

<details>
<summary><strong>View Answer</strong></summary>

**Common Causes:**

1. **No NAT Gateway**
   - Lambda in private subnet can't reach internet
   - Solution: Add NAT Gateway or use VPC endpoints

2. **Security Group Issues**
   - Lambda SG can't reach RDS SG
   - Solution: Allow Lambda SG in RDS inbound rules

3. **Subnet Routing**
   - No route to NAT Gateway
   - Solution: Update route table

**Debugging Steps:**
```bash
# Check Lambda configuration
aws lambda get-function --function-name myFunction

# Check VPC configuration
aws ec2 describe-subnets --subnet-ids subnet-xxx
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-xxx"
aws ec2 describe-security-groups --group-ids sg-xxx
```

**Best Practice Architecture:**
```
Lambda (Private Subnet) 
    → NAT Gateway (Public Subnet) 
    → Internet Gateway 
    → Internet

Lambda (Private Subnet) 
    → VPC Endpoint 
    → AWS Services (S3, DynamoDB, etc.)
```

</details>

---

### Question 2: Design DynamoDB for High-Throughput E-commerce

**Scenario:** DynamoDB is throttling requests and costs are high. Optimize the table design.

<details>
<summary><strong>View Answer</strong></summary>

**Problem: Hot Partition**
```
# Bad partition key - all today's orders hit same partition
PK: order_date (2024-01-15)
SK: order_id
```

**Solution: Write Sharding**
```
# Good partition key - distribute load across partitions
PK: order_date#shard_id (2024-01-15#3)
SK: order_id

# Shard ID = hash(order_id) % 10
```

**Cost Optimization:**
```python
# Use On-Demand for unpredictable traffic
# Use Provisioned with Auto Scaling for predictable patterns

# Enable TTL for automatic cleanup
aws dynamodb update-time-to-live \
    --table-name Orders \
    --time-to-live-specification "Enabled=true,AttributeName=ttl"

# Use GSI projection to reduce read costs
# Only project needed attributes, not ALL
```

**Access Patterns:**
| Access Pattern | Solution |
|----------------|----------|
| Get order by ID | PK = order_id |
| Get orders by customer | GSI: customer_id |
| Get recent orders | GSI: date + sort by timestamp |

</details>

---

### Question 3: ECS Task Failures - Exit Code 137

**Scenario:** ECS tasks are failing with exit code 137 and health check failures. Debug the issue.

<details>
<summary><strong>View Answer</strong></summary>

**Exit Code 137 = OOMKilled**

The container is running out of memory and being killed by the kernel.

**Debugging Steps:**

1. **Check task definition memory limits:**
```json
{
  "containerDefinitions": [{
    "memory": 512,
    "memoryReservation": 256
  }]
}
```

2. **Check CloudWatch Container Insights:**
```bash
aws logs get-log-events \
    --log-group-name /ecs/my-service \
    --log-stream-name ecs/my-container/task-id
```

3. **Check memory utilization:**
```bash
aws cloudwatch get-metric-statistics \
    --namespace AWS/ECS \
    --metric-name MemoryUtilization \
    --dimensions Name=ClusterName,Value=my-cluster
```

**Solutions:**

| Issue | Fix |
|-------|-----|
| Memory limit too low | Increase `memory` in task definition |
| Memory leak in app | Profile and fix application code |
| JVM heap too large | Set `-Xmx` to 75% of container memory |
| Health check too aggressive | Increase `healthCheckGracePeriodSeconds` |

**ECS Exec for debugging:**
```bash
aws ecs execute-command \
    --cluster my-cluster \
    --task task-id \
    --container my-container \
    --interactive \
    --command "/bin/sh"
```

</details>

---

### Question 4: Design SQS/SNS for Reliable Messaging

**Scenario:** Messages are being lost and processed multiple times. Implement reliable messaging.

<details>
<summary><strong>View Answer</strong></summary>

**Architecture:**
```
Producer → SNS Topic → SQS Queues (fan-out)
                           │
                           ├── Order Processing Queue
                           ├── Notification Queue
                           └── Analytics Queue
                           
Each Queue → DLQ (after 3 retries)
```

**SQS Configuration:**
```python
import boto3

sqs = boto3.client('sqs')

# Create queue with DLQ
sqs.create_queue(
    QueueName='order-processing',
    Attributes={
        'VisibilityTimeout': '300',  # 5 minutes
        'ReceiveMessageWaitTimeSeconds': '20',  # Long polling
        'RedrivePolicy': json.dumps({
            'deadLetterTargetArn': dlq_arn,
            'maxReceiveCount': '3'
        })
    }
)
```

**Prevent Duplicate Processing:**
```python
# Use message deduplication ID for FIFO queues
sqs.send_message(
    QueueUrl=fifo_queue_url,
    MessageBody=json.dumps(order),
    MessageGroupId='order-group',
    MessageDeduplicationId=order_id  # Prevents duplicates within 5 min
)

# Or implement idempotency in consumer
def process_message(message):
    order_id = message['order_id']
    
    # Check if already processed
    if redis.get(f'processed:{order_id}'):
        return  # Skip duplicate
    
    # Process order
    process_order(order_id)
    
    # Mark as processed (with TTL)
    redis.setex(f'processed:{order_id}', 3600, 'true')
```

</details>

---

## System Design Questions

### Design Amazon.com Product Page

<details>
<summary><strong>View Answer</strong></summary>

**Requirements:**
- 100M+ products
- Millions of concurrent users
- Real-time inventory
- Personalized recommendations

**High-Level Design:**
```
User → CloudFront (CDN) → ALB → Product Service
                                      │
                     ┌────────────────┼────────────────┐
                     │                │                │
                     ▼                ▼                ▼
              Product DB      Inventory Cache    Recommendation
              (DynamoDB)         (Redis)          Service (ML)
```

**Key Components:**

1. **Product Catalog Service**
   - DynamoDB for product data
   - Elasticsearch for search
   - S3 + CloudFront for images

2. **Inventory Service**
   - Redis for real-time stock
   - SQS for inventory updates
   - Eventually consistent

3. **Recommendation Engine**
   - Collaborative filtering
   - Personalize service
   - Real-time + batch processing

4. **Caching Strategy**
   - CloudFront for static assets
   - ElastiCache for API responses
   - DynamoDB DAX for DB caching

</details>

---

## Behavioral Questions (STAR Format)

Use STAR format: **Situation, Task, Action, Result**

### Customer Obsession

**Question:** Tell me about a time you went above and beyond for a customer.

**Example Answer:**
- **S:** Customer reported data loss after server migration
- **T:** Recover data and prevent future occurrences
- **A:** Worked overnight, wrote backup verification tool, implemented automated testing
- **R:** Recovered 100% of data, tool prevented 3 similar issues

### Ownership

**Question:** Tell me about a time you took ownership of something outside your job scope.

### Bias for Action

**Question:** Tell me about a time you made an important decision without complete information.

### Dive Deep

**Question:** Tell me about a complex problem you solved by getting into the details.

---

## Interview Tips for Amazon

1. **STAR format is mandatory** - Practice every behavioral question in STAR
2. **Quantify results** - Use numbers ("reduced latency by 50%")
3. **Prepare 10-15 stories** - Map them to different LPs
4. **Be specific** - They will dive deep, so know your details
5. **Show ownership** - Use "I" not "we" for your contributions
6. **Ask clarifying questions** - Even for behavioral questions

---

## Preparation Timeline

```
Month 1-2: DSA + Leadership Principles
├── LeetCode: Focus on Amazon-tagged problems
├── Write 15 STAR stories
└── Practice explaining technical decisions

Month 3: System Design
├── Read DDIA
├── Practice AWS-based designs
└── Focus on scalability

Month 4: Mock Interviews
├── Pramp for coding
├── Practice LP stories out loud
└── Record yourself and review
```

---

## Resources

- [AWS Interview Questions](../../interview-questions/cloud/aws.md)
- [Docker Questions](../../interview-questions/devops/docker.md)
- [System Design Roadmap](../../roadmaps/cloud-engineer.md)
- [DevOps Roadmap](../../roadmaps/devops-engineer.md)

---

*Build AWS projects for your interview portfolio? [Try DeployU](https://deployu.ai?ref=github) - Deploy on real AWS infrastructure with zero billing risk.*
