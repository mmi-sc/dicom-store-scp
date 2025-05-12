# DICOM Store SCP (PACS Server)

This container image provides a DICOM Store SCP service suitable for medical imaging workflows, compliant with DICOM C-STORE operations.

## 🧩 Overview

This is a containerized DICOM server that receives and stores DICOM files to an S3-compatible destination.
It supports deployment in local environments (`docker run`) or as part of an AWS Fargate ECS service.

---

## ⚙️ Environment Variables

| Variable                   | Required | Description                                                                  | Default    |
|----------------------------|----------|------------------------------------------------------------------------------|------------|
| `DICOM_BUCKET`             | ✅ Yes   | Name of the S3 bucket where received DICOM files will be stored              | N/A        |
| `SCP_PORT`                 | ⭕ No    | Port number to listen for DICOM C-STORE requests                             | `11112`    |
| `AE_TITLE`                 | ⭕ No    | AE title for the SCP                                                         | `STORESCP` |
| `REQUIRE_CALLED_AE_TITLE`  | ⭕ No    | Expected AE Title from sender (optional validation)                          | `false`    |
| `REQUIRE_CALLING_AE_TITLE` | ⭕ No    | Expected calling AE Title (optional validation)                              | empty      |
| `MAX_POOL_CONNECTIONS`     | ⭕ No    | Max pool connections for I/O                                                 | `50`       |
| `DIMSE_TIMEOUT`            | ⭕ No    | Timeout for DIMSE messages                                                   | `60`       |
| `MAXIMUM_ASSOCIATIONS`     | ⭕ No    | Max parallel DICOM associations                                              | `300`      |
| `MAXIMUM_PDU_SIZE`         | ⭕ No    | Max PDU size (in bytes)                                                      | `0`        |
| `NETWORK_TIMEOUT`          | ⭕ No    | Timeout for network connection                                               | `90`       |
| `SUPPORTED_SOP_CLASS_UIDS` | ⭕ No    | Comma-separated list of supported SOP Class UIDs                             | empty      |
| `LOG_LEVEL`                | ⭕ No    | Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`)              | `INFO`     |

---

## 🚀 Usage Examples

### 1. Run Locally with Docker

```bash
docker run -p 11112:11112 \
  -e DICOM_BUCKET=<your-dicom-bucket-name> \
  -e AWS_ACCESS_KEY_ID=<your-access-key-id> \
  -e AWS_SECRET_ACCESS_KEY=<your-secret-access-key> \
  -e AWS_DEFAULT_REGION=<your-region> \
  public.ecr.aws/mmi/dicom-store-scp:latest
```

### 2. Use in ECS/Fargate Task Definition

```json
{
  "containerDefinitions": [
    {
      "name": "dicom-scp",
      "image": "public.ecr.aws/mmi/dicom-store-scp:latest",
      "portMappings": [
        { "containerPort": 11112 }
      ],
      "environment": [
        { "name": "DICOM_BUCKET", "value": "your-dicom-bucket-name" },
        { "name": "AE_TITLE", "value": "YOUR-AE-TITLE" },
        { "name": "LOG_LEVEL", "value": "DEBUG" }
      ]
    }
  ]
}
```

---

## 📦 Health Check

This container supports a health check that validates whether the SCP port is open:

```dockerfile
HEALTHCHECK CMD nc -z localhost $SCP_PORT || exit 1
```

This helps ECS or Docker orchestrators detect when the service becomes unhealthy.

---

## 🛡️ Security

- Runs as non-root user (`dicomuser`)
- Uses `python:3.12-slim` for minimal attack surface
- Health check for stability

---

## 📄 License

MIT or custom EULA as applicable.

---

## 🔐 AWS Credentials

This container does not include AWS credentials.

To enable S3 access, you must configure valid credentials via one of the following methods:

1. **AWS ECS Task Role (Recommended)**
   - Attach an IAM role with `s3:PutObject` and `s3:ListBucket` permissions to your ECS task.

2. **Environment variables (for local testing only)**
   - If running locally with `docker run`, pass credentials explicitly:

   ```bash
   -e AWS_ACCESS_KEY_ID=<your-access-key-id> \
   -e AWS_SECRET_ACCESS_KEY=<your-secret-access-key> \
   -e AWS_DEFAULT_REGION=<your-region>
   ```

> ⚠️ NOTE: AWS CLI configuration via `aws configure` or `~/.aws/credentials` is not visible inside the container unless you explicitly mount that directory, which is not recommended for security reasons.

> ⚠️ NOTE: Never hard-code credentials or use environment variables in production.


---

## 🌍 AWS Region Behavior

This container checks for `AWS_REGION` as an environment variable. If not provided, the AWS SDK (boto3) will fall back to:

- `AWS_DEFAULT_REGION` environment variable
- Region setting in `~/.aws/config`
- EC2 or ECS instance metadata (if available)

This ensures compatibility for both container-only and CDK-integrated deployments.

## 📄 License

This product is licensed under the **Standard AWS Marketplace EULA**.  
See the AWS Marketplace listing for full terms and conditions.

## ⚠️ Disclaimer

This product is not a medical device.

It is designed to help healthcare institutions manage medical imaging data efficiently and enhance the quality of patient care. However, it is not intended to replace professional medical advice, diagnosis, or treatment.
