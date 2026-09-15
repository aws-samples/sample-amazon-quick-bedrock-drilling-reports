# Daily Drilling Report Intelligence with Amazon Bedrock and Amazon Quick

A single CloudFormation template that turns daily drilling report (DDR) PDFs from any operator into a canonical dataset and an executive dashboard. Upload a PDF, and within a few minutes the record appears in Amazon Quick.

This repository accompanies the blog post [link to be added].

## How it works

```
PDF upload (s3://<bucket>/raw/)
  -> S3 event -> Lambda -> Claude on Amazon Bedrock (Converse API, PDF as document input)
  -> one TSV row per report (s3://<bucket>/curated/)
  -> AWS Glue table -> Amazon Athena workgroup
  -> Amazon Quick dataset (SPICE or direct query) -> dashboard
```

The Lambda function prompts Claude to return a fixed JSON schema (well, rig, depths, footage, ROP, NPT hours, incidents, cost, and a 24-hour summary), normalizes units to feet and dates to ISO format, writes the row, and in SPICE mode starts a dataset ingestion so the dashboard refreshes immediately. An hourly refresh schedule acts as a safety net.

## Prerequisites

- An Amazon Quick subscription in the deployment Region, with Amazon Athena access enabled once under Security and permissions
- Amazon Bedrock model access enabled for the chosen Claude model (default: Claude Sonnet 4.6 through the US cross-Region inference profile)
- The ARN of the Amazon Quick user who will own the dashboard:

  ```
  aws quicksight list-users --aws-account-id <account-id> --namespace default --region <identity-region>
  ```

## Deploy

```
aws cloudformation deploy \
  --template-file ddr-pipeline-Cloudformation.yaml \
  --stack-name ddr-intel \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides QuickSightUserArn=<user-arn>
```

| Parameter | Default | Purpose |
|---|---|---|
| `QuickSightUserArn` | (required) | Owner of the theme, data source, dataset, and dashboard |
| `BedrockModelId` | `us.anthropic.claude-sonnet-4-6` | Inference profile ID for extraction; use the profile for your geography |
| `DataSetImportMode` | `SPICE` | `SPICE` for in-memory with scheduled refresh, `DIRECT_QUERY` for live Athena queries |
| `BucketPrefix` | `ddr-intel` | Data bucket name prefix; the account ID is appended |

## Use

Upload one or more DDR PDFs to the `raw/` prefix (the `UploadExample` stack output has the exact command), then open the URL in the `DashboardUrl` output. The first upload takes a minute or two; later uploads appear after the next ingestion.

## Security configuration

The data bucket has default encryption (SSE-S3), versioning, server access logging to a separate log bucket (which also logs its own access), public access blocked, and a bucket policy that denies non-HTTPS requests. Amazon Quick reaches the bucket through a bucket policy scoped to the Quick service roles in your account, so no bucket selection is needed in the Quick console. The Lambda role is limited to reading `raw/`, writing `curated/`, invoking the chosen model, and starting ingestions for this dataset.

## Cost

The stack incurs charges for Bedrock inference per PDF, Lambda, Athena queries, S3 storage (including retained object versions in the data bucket), and Amazon Quick capacity. Access logs expire after 90 days. Delete the stack when you are done testing.

## Cleanup

Both buckets must be empty before the stack can be deleted, and the data bucket is versioned, so `aws s3 rm` alone is not enough:

```
python3 -c "import boto3,sys; [boto3.resource('s3').Bucket(b).object_versions.delete() for b in sys.argv[1:]]" \
  <prefix>-<account-id> <prefix>-<account-id>-logs
aws cloudformation delete-stack --stack-name ddr-intel
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
