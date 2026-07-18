For your work on the cloud connector, these are the AWS references I'd recommend, in priority order.

|Topic|Why it matters|Reference|
|---|---|---|
|**GetCostAndUsage API**|Your `FetchCostData()` directly calls this API. Learn request/response structure, metrics, dimensions, filters, grouping, pagination, and the inclusive/exclusive date rules.|[GetCostAndUsage API Reference](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_GetCostAndUsage.html?utm_source=chatgpt.com) ([AWS Documentation](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_GetCostAndUsage.html?utm_source=chatgpt.com "GetCostAndUsage - AWS Billing and Cost Management"))|
|**Using the Cost Explorer API**|Explains service endpoint, IAM permissions, and how Cost Explorer is intended to be used.|[Using the Cost Explorer API](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-api.html?utm_source=chatgpt.com) ([AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-api.html?utm_source=chatgpt.com "Using the AWS Cost Explorer API - AWS Cost Management"))|
|**GetCallerIdentity API**|Your `ValidateCredential()` uses this operation. Explains exactly what is returned and why it succeeds even without explicit permissions.|[GetCallerIdentity API Reference](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html?utm_source=chatgpt.com) ([AWS Documentation](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html?utm_source=chatgpt.com "GetCallerIdentity - AWS Security Token Service"))|
|**AssumeRole API**|Your mapper optionally performs `AssumeRole`. Learn request parameters, trust relationships, session duration, session name, and returned credentials.|[AssumeRole API Reference](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html?utm_source=chatgpt.com)|
|**IAM Roles User Guide**|Explains trust policies, identity policies, and cross-account role assumption.|[IAM Roles User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html?utm_source=chatgpt.com)|
|**AWS Organizations User Guide**|Explains management accounts, member accounts, consolidated billing, and linked accounts, which affect Cost Explorer visibility.|[AWS Organizations User Guide](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html?utm_source=chatgpt.com)|
|**Cost Allocation Tags**|Relevant when you later populate FOCUS `Tags`. Explains activation and billing behavior.|[Cost Allocation Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html?utm_source=chatgpt.com)|
|**Cost Categories**|Useful if your product eventually groups costs by business units or environments rather than AWS services.|[Cost Categories User Guide](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/manage-cost-categories.html?utm_source=chatgpt.com)|
|**AWS SDK for Go v2 – Cost Explorer**|Documents the Go client you are already using.|[AWS SDK for Go v2 Cost Explorer Package](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/costexplorer?utm_source=chatgpt.com)|
|**AWS SDK for Go v2 – STS**|Documents the Go STS client used by your validator and AssumeRole flow.|[AWS SDK for Go v2 STS Package](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sts?utm_source=chatgpt.com)|

## Read these sections carefully in `GetCostAndUsage`

These are directly applicable to your current implementation:

- `TimePeriod` (start is inclusive, end is exclusive)
    
- `Metrics`
    
- `GroupBy`
    
- `Filter`
    
- `Granularity`
    
- `NextPageToken`
    
- `ResultsByTime`
    

Reference: [GetCostAndUsage API Reference](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_GetCostAndUsage.html?utm_source=chatgpt.com) ([AWS Documentation](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_GetCostAndUsage.html?utm_source=chatgpt.com "GetCostAndUsage - AWS Billing and Cost Management"))

## Read these sections in `AssumeRole`

Since your code already implements cross-account access, focus on:

- `RoleArn`
    
- `RoleSessionName`
    
- `DurationSeconds`
    
- Response `Credentials`
    
- Trust relationships
    
- Cross-account example
    

Reference: [AssumeRole API Reference](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html?utm_source=chatgpt.com)

## Read these sections in `GetCallerIdentity`

Understand exactly what your validator returns:

- `Account`
    
- `Arn`
    
- `UserId`
    

Reference: [GetCallerIdentity API Reference](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html?utm_source=chatgpt.com) ([AWS Documentation](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html?utm_source=chatgpt.com "GetCallerIdentity - AWS Security Token Service"))

These documents cover essentially everything your current AWS provider implementation relies on, without branching into unrelated AWS services.