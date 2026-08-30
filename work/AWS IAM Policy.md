**Context:** The Atomity AWS connector uses a cross-account IAM role with read-only access (FR-AWS-001). The base policy already covers cost data ingestion and provider-native recommendations, with the **additional** statements required for the resource inventory, CloudWatch metrics fetchers introduced in INSIGHT-2, and the Organizations account discovery used by the provider-native recommendation fetch.

---
## Trust Policy

```json
{

	"Version": "2012-10-17",	
	"Statement": [
		{	
			"Effect": "Allow",	
			"Principal": {
				"AWS": "arn:aws:iam::%s:root"
			},
			"Action": "sts:AssumeRole",
			"Condition": {	
				"StringEquals": {
					"sts:ExternalId": "%s"
				}
			}
		}
	]
}
```

## Complete Atomity Cross-Account Role Permission Policy (reference)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AtomityCostData",
      "Effect": "Allow",
      "Action": [
        "cur:DescribeReportDefinitions",
        "bcm-data-exports:GetExport",
        "bcm-data-exports:ListExports",
        "ce:GetCostAndUsage",
        "ce:GetCostForecast",
        "ce:GetAnomalyMonitors",
        "ce:GetAnomalySubscriptions",
        "ce:GetAnomalies"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AtomityS3Export",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::CUSTOMER_BILLING_BUCKET",
        "arn:aws:s3:::CUSTOMER_BILLING_BUCKET/*"
      ]
    },
    {
      "Sid": "AtomityProviderRecommendations",
      "Effect": "Allow",
      "Action": [
        "cost-optimization-hub:ListRecommendations",
        "cost-optimization-hub:GetRecommendation",
        "cost-optimization-hub:ListRecommendationSummaries",
        "compute-optimizer:GetEC2InstanceRecommendations",
        "compute-optimizer:GetAutoScalingGroupRecommendations",
        "compute-optimizer:GetRecommendationSummaries"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AtomityOrganizations",
      "Effect": "Allow",
      "Action": [
        "organizations:ListAccounts",
        "organizations:DescribeOrganization",
        "organizations:DescribeAccount"
      ],
      "Resource": "*"
    },for both the customer
    {
      "Sid": "AtomityResourceTags",
      "Effect": "Allow",
      "Action": [
        "tag:GetResources"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AtomityResourceInventory",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "rds:DescribeDBInstances",
        "elasticloadbalancing:DescribeLoadBalancers"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AtomityCloudWatchMetrics",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AtomityS3BucketDiscovery",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    }
  ]
}
```

Replace `CUSTOMER_BILLING_BUCKET` with the customer's actual S3 bucket name during the AWS connection wizard.

### Notes

- All actions are **read-only**. The role must never be granted `Create*`, `Delete*`, `Stop*`, `Modify*`, or `Purchase*` actions (SEC-003).
    
- `Resource: "*"` is required because `Describe*`, `GetMetricData`, and Organizations calls do not support resource-level ARN restrictions. AWS documents this in the [EC2 actions table](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html), [CloudWatch actions table](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazoncloudwatch.html), and [Organizations actions table](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsorganizations.html).
    
- For organisations using AWS Organizations SCPs, ensure the SCP does not deny `cloudwatch:GetMetricData`, the `Describe*` actions, or the `organizations:*` actions in member accounts that Atomity should access.
    
- `organizations:ListAccounts` is required for multi-account recommendation coverage (FR-AWS-003). If denied — standalone account or SCP restriction — the connector falls back gracefully to the caller's own account via STS `GetCallerIdentity`. Recommendations will be scoped to one account instead of the full organization. This is a degraded but functional state, not a connection failure.

---