# DICOM Store SCP (PACS Server)

This container image provides a DICOM Store SCP service suitable for medical imaging workflows, compliant with DICOM C-STORE operations.

## 🧩 Overview

This is a containerized DICOM server that receives and stores DICOM files to an S3-compatible destination.
It supports deployment in local environments (`docker run`) or as part of an AWS Fargate ECS service.

---

## 🚀 Usage

### Run Locally with Docker

#### 1. **Run the container using the following command.**

Ensure the `$CONTAINER_IMAGES` environment variable is set and the image has been pulled.

```bash
CONTAINER_IMAGES="709825985650.dkr.ecr.us-east-1.amazonaws.com/man-machine-interface/dicom-store-scp:v1.2.0"

docker run -p 11112:11112 $CONTAINER_IMAGES
```

---

#### 2. **Send DICOM files using your preferred Store SCU.**

For example, you can use [DCMTK](https://dicom.offis.de/en/dcmtk/dcmtk-tools/) to send all DICOM files in the current directory:

```bash
storescu localhost 11112 *.dcm
```

DICOM files will be saved under `/app/storage` inside the container's file system.

To store DICOM files in an Amazon S3 bucket, deploy the container using **Amazon ECS** with appropriate **IAM permissions**.

---

### ECS Launch Instructions (via AWS Management Console)

This section explains how to deploy the container using **Amazon ECS with Fargate** through the AWS Management Console. This is the recommended method for running the application securely using an IAM role (no access keys needed).

#### 1. Prerequisites

##### 1.1 Create an Amazon S3 bucket

This container stores incoming DICOM files in an S3 bucket.
If you don't have one already, follow the steps here to create one:
[How to create an S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html)

1. Open the [Amazon S3 Console](https://console.aws.amazon.com/s3/)
2. Click **"Create bucket"**
3. Give the bucket a unique name (e.g., `my-dicom-storage-bucket`)
4. Choose a region and leave other settings as default unless you have specific needs
5. Click **"Create bucket"** at the bottom of the form

Take note of the bucket name — you'll need it in later steps as `DICOM_BUCKET`.

---

##### 1.2 Create an IAM Role for ECS Tasks

The container requires permission to write files to your S3 bucket. This permission should be granted via an **IAM role** that is assigned to the ECS task.

Follow these steps:

1. Open the [IAM Console](https://console.aws.amazon.com/iam/)
2. In the left navigation pane, choose **Roles** > **Create role**
3. Under **Trusted entity type**, select **AWS service**
4. Under **Use case**, choose **Elastic Container Service > Elastic Container Service Task**
5. Click **Next**
6. Click **Next** again to skip adding a managed policy for now
7. Enter a **Role name** (e.g., `DicomScpTaskRole`) and click **Create role**

After the role is created:

8. Click the newly created role to open its details
9. Under the **Permissions** tab, choose **Add permissions** > **Create inline policy**
10. In the **Policy editor**, select the **JSON** tab
11. Paste the following policy document:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-dicom-storage-bucket",
        "arn:aws:s3:::my-dicom-storage-bucket/*"
      ]
    }
  ]
}
```

12. Click **Next**, enter a **Policy name** (e.g., `DicomScpTaskPolicy`)
13. Click **Create policy** to finish

---

#### 2. Create an ECS Task Definition

1. Open the [ECS Console](https://console.aws.amazon.com/ecs/) and go to **Task Definitions**
2. Click **Create new task definition** > **Create new task definition**
3. In the **Task definition configuration** section:
   - Enter a **Task definition family name** (e.g., `dicom-scp-task`)
4. In the **Infrastructure requirements** section:
   - Ensure **Launch type** is set to **AWS Fargate**
   - Under **Task Role**, select the IAM role you created earlier (e.g., `DicomScpTaskRole`)
5. In the **Container** section:
   - Fill in the container details:
     - **Name**: `dicom-store-scp`
     - **Image URI**:
       `709825985650.dkr.ecr.us-east-1.amazonaws.com/man-machine-interface/dicom-store-scp:v1.2.0`
     - **Essential container**: `Yes`
   - Remove any default **Container port**
   - Click **Add port mapping** and set the following:
     - **Container port**: `11112`
     - **Protocol**: `TCP`
     - **App protocol**: `None`
6. Scroll down to the **Environment variables** section:
   - Click **Add environment variable** and enter:
     - **Key**: `DICOM_BUCKET`
     - **Value type**: `Value`
     - **Value**: your S3 bucket name (e.g., `my-dicom-storage-bucket`)
7. Leave all other settings at their defaults unless specific changes are required, and click **Create**.

---

#### 3. Run the Task in a Fargate Service

1. Go to the **Clusters** in the ECS console, click **Create cluster**
2. Enter a name for the cluster (e.g., `dicom-scp-cluster`), and click **Create**
3. Once the cluster is created, open the cluster
4. With the Services tab selected, click **Create**
5. In the **Service details** section:
   - **Task Definition family**: Select the one you created earlier (e.g., `dicom-scp-task`)
   - **Service name**: (e.g., `dicom-scp-service`)
6. In the **Networking** section:
   - **VPC**: Choose the default VPC
   - **Subnets**: Select at least one public subnet
   - **Security groups**: Create or select one that allows TCP port 11112 from `0.0.0.0/0`
   - **Auto-assign public IP**: `ENABLED`
7. Click **Create**

After a few minutes, the task should be running.  
You can verify that the container is listening on port `11112` and ready to accept DICOM C-STORE requests.

---

## ⚙️ Environment Variables

| Variable                   | Optional | Description                                                     | Default    |
|----------------------------|----------|-----------------------------------------------------------------|------------|
| `DICOM_BUCKET`             | Yes      | Name of the S3 bucket where received DICOM files will be stored | N/A        |
| `SCP_PORT`                 | Yes      | Port number to listen for DICOM C-STORE requests                | `11112`    |
| `AE_TITLE`                 | Yes      | AE title for the SCP                                            | `STORESCP` |
| `REQUIRE_CALLED_AE_TITLE`  | Yes      | Expected AE Title from sender                                   | `false`    |
| `REQUIRE_CALLING_AE_TITLE` | Yes      | Expected calling AE Title                                       | empty      |
| `MAX_POOL_CONNECTIONS`     | Yes      | Max pool connections for I/O                                    | `50`       |
| `DIMSE_TIMEOUT`            | Yes      | Timeout for DIMSE messages                                      | `60`       |
| `MAXIMUM_ASSOCIATIONS`     | Yes      | Max parallel DICOM associations                                 | `300`      |
| `MAXIMUM_PDU_SIZE`         | Yes      | Max PDU size (in bytes)                                         | `0`        |
| `NETWORK_TIMEOUT`          | Yes      | Timeout for network connection                                  | `90`       |
| `SUPPORTED_SOP_CLASS_UIDS` | Yes      | Comma-separated list of supported SOP Class UIDs                | empty      |
| `LOG_LEVEL`                | Yes      | Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`) | `INFO`     |

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
