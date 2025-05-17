# DICOM Store SCP (PACS Server)

This container image provides a DICOM Store SCP service suitable for medical imaging workflows, compliant with DICOM C-STORE operations.

## 🧩 Overview

This is a containerized DICOM server that receives and stores DICOM files to an S3-compatible destination.
It supports deployment in local environments (`docker run`) or as part of an AWS Fargate ECS service.

---

## 🚀 Usage

### Run Locally with Docker

#### 1. **Prepare an Amazon S3 bucket to store DICOM files, and note the bucket name.**

For instructions, refer to:  
[Creating a general purpose bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html)

Example: Set the bucket name as an environment variable:

```bash
DICOM_BUCKET="your-dicom-bucket-name"
```

#### 2. **Create AWS credentials that allow the container to upload files to S3.**

You will need an **Access Key ID** and **Secret Access Key** for an IAM user with `s3:PutObject` and `s3:ListBucket` permissions.  
For help:  
[Create an access key for an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-keys-admin-managed.html#admin-create-access-key)

If you have already configured a default profile using `aws configure`, you can load the credentials like this:

```bash
AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

#### 3. **Run the container using the following command.**

Make sure you have set the `$CONTAINER_IMAGES` variable and pulled the container image accordingly.

```bash
CONTAINER_IMAGES="709825985650.dkr.ecr.us-east-1.amazonaws.com/man-machine-interface/dicom-store-scp:v1.1.1"
```

```bash
docker run -p 11112:11112 \
  -e DICOM_BUCKET=$DICOM_BUCKET \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  $CONTAINER_IMAGES
```

#### 4. **Use your preferred DICOM Store SCU to send STORE requests to the SCP.**

The following example uses [DCMTK](https://dicom.offis.de/en/dcmtk/dcmtk-tools/) to send all DICOM files in the current directory to the server:

```bash
storescu localhost 11112 *.dcm
```

### Use in ECS/Fargate Task Definition

```json
{
  "containerDefinitions": [
    {
      "name": "dicom-scp",
      "image": "709825985650.dkr.ecr.us-east-1.amazonaws.com/man-machine-interface/dicom-store-scp:v1.1.1",
      "portMappings": [
        { "containerPort": 11112 }
      ],
      "environment": [
        { "name": "DICOM_BUCKET", "value": "your-dicom-bucket-name" }
      ]
    }
  ]
}
```

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

## 📦 Health Check

This container supports a health check that validates whether the SCP port is open:

```dockerfile
HEALTHCHECK CMD nc -z localhost $SCP_PORT || exit 1
```

This helps ECS or Docker orchestrators detect when the service becomes unhealthy.

---

## 🛡️ Security

- Runs as non-root user (`dicomuser`)
- Built on python:3.12-alpine, a lightweight and security-hardened base image with minimal surface area
- Includes a health check to ensure SCP port availability and container stability

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
