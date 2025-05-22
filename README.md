# 📦 DICOM Store SCP for AWS Marketplace (ECS)

This containerized application provides a robust DICOM C-STORE SCP service, enabling healthcare organizations to store medical imaging data reliably. DICOM (Digital Imaging and Communications in Medicine) is an established standard widely used to manage and transfer medical imaging data.

---

## 🧩 Overview

This containerized DICOM server receives and stores DICOM files either in an Amazon S3 bucket or within the container file system at `/app/storage`.
It supports running via the `docker run` command and deployment as part of an AWS Fargate ECS service.

---

## 🚀 Run the Container

1. Ensure you have pulled the container image from Amazon ECR following the **Pull container image** instructions.  
   As part of those steps, the `CONTAINER_IMAGES` environment variable should be set with the appropriate image URL.

2. **Run the container using the command below:**

```bash
docker run -p 11112:11112 $CONTAINER_IMAGES
```

The service is now operational.

By default, DICOM files are stored in `/app/storage`.  
*To store files in Amazon S3, deploy the container using ECS (Fargate). Local deployment with S3 is not supported.*

---

## ☁️ Deploy on AWS ECS (Fargate)

### Step 1: Create an Amazon S3 Bucket

Follow these steps to create your bucket named **`my-dicom-storage-bucket`**:

1. Open the [Amazon S3 Console](https://console.aws.amazon.com/s3/).
2. Select your desired AWS region from the top-right region menu first.
3. Click **"Create bucket"**.
4. Name the bucket `my-dicom-storage-bucket`.
5. Click **"Create bucket"**.

### Step 2: Create an IAM Role for ECS Tasks

This IAM role enables ECS tasks to write to your bucket.

1. Open the [IAM Console](https://console.aws.amazon.com/iam/).
2. Select **Roles**, then click **Create role**.
3. In the **Trusted entity type** section, select **AWS service**.
4. In the **Use case** section:
   - From the **Service or use case** dropdown menu, select **Elastic Container Service**.
   - Choose **Elastic Container Service Task** as the use case.
5. Click **Next**.
6. Click **Next** without selecting additional policies.
7. Enter `DicomScpTaskRole` as the role name, then click **Create role**.

Add an inline policy:

1. Select your new role, click **Add permissions** > **Create inline policy**.
2. Select the **JSON** tab, paste the policy below:

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
3. Click **Next**, name it `DicomScpTaskPolicy`, and click **Create policy**.

### Step 3: ECS Task Definition

1. Open [ECS Console](https://console.aws.amazon.com/ecs/) > **Task Definitions**.
2. Click **Create new task definition** > **Create new task definition**.
3. In the **Task definition configuration** section, enter `dicom-scp-task` as the task family.
4. In the **Infrastructure requirements** section, select `DicomScpTaskRole` as **Task role**.
5. In the **Container** section, configure the container:
   - **Name**: `dicom-store-scp`
   - **Image URI**: Container image URL from ECR.
   - **Port mapping**: TCP `11112`  
                       Remove any default Container port
   - Environment variables:
     - `DICOM_BUCKET`: `my-dicom-storage-bucket`
6. Click **Create**.

### Step 4: Launch ECS Service

1. Navigate to the **Clusters** page in the [ECS console](https://console.aws.amazon.com/ecs/) and click **Create cluster**.
2. Enter a name for your cluster (e.g., `dicom-scp-cluster`) and click **Create**.
3. After the cluster has been created, open it.
4. On the **Services** tab, click **Create**.
5. In the **Service details** section:
   - **Task Definition family**: Select the one you created earlier (e.g., `dicom-scp-task`).
   - **Service name**: Enter a suitable name (e.g., `dicom-scp-service`).
6. In the **Networking** section:
   - **VPC**: Select your default VPC.
   - **Subnets**: Select at least one public subnet.
   - **Security groups**: Create or select a security group allowing inbound TCP port `11112` from `0.0.0.0/0`.
   - **Auto-assign public IP**: Set to **ENABLED**.
7. Click **Create**.

After a few minutes, your task should display a **RUNNING** status.  
You can verify the container is listening on port `11112` and ready to accept DICOM C-STORE requests.

---

## 📡 Verifying SCP Connectivity (Example Store SCU Operation)

After successful deployment:

1. Confirm ECS task status is `RUNNING`.
2. Send a DICOM file with an SCU client such as `storescu` from [DCMTK](https://dicom.offis.de/dcmtk.php.en):

```bash
storescu ecs-public-ip 11112 example.dcm
```

3. Confirm file storage in your S3 bucket (`my-dicom-storage-bucket`) via AWS S3 Console.

---

## ⚙️ Environment Variables Configuration

| Variable                   | Required | Description                                                      |
|----------------------------|----------|------------------------------------------------------------------|
| `DICOM_BUCKET`             | No       | Amazon S3 bucket for file storage. If omitted, files will be stored under `/app/storage`. |
| `SCP_PORT`                 | No       | Listening port for DICOM (default: `11112`).                     |
| `AE_TITLE`                 | No       | AE Title of SCP server (default: `STORESCP`).                    |
| `REQUIRE_CALLED_AE_TITLE`  | No       | Accept connections matching the AE Title if `true`.              |
| `REQUIRE_CALLING_AE_TITLE` | No       | Accept connections only from this AE Title.                      |
| `MAX_POOL_CONNECTIONS`     | No       | Max concurrent S3 connections (default: `50`).                   |
| `DIMSE_TIMEOUT`            | No       | Timeout for DIMSE operations (default: `60` seconds).            |
| `MAXIMUM_ASSOCIATIONS`     | No       | Max simultaneous DICOM connections (default: `300`).             |
| `MAXIMUM_PDU_SIZE`         | No       | Max PDU size (default: `0`, unlimited).                          |
| `NETWORK_TIMEOUT`          | No       | Network timeout (default: `90` seconds).                         |
| `SUPPORTED_SOP_CLASS_UIDS` | No       | Supported SOP Class UIDs, comma-separated (default: all classes).|
| `LOG_LEVEL`                | No       | Log verbosity (DEBUG, INFO, WARNING, ERROR, CRITICAL; default: INFO). |

---

## 🔍 Health Check

Integrated health check confirms service availability:

```dockerfile
HEALTHCHECK CMD nc -z localhost $SCP_PORT || exit 1
```

---

## 🔐 Credentials and Security

- Secure non-root execution.
- AWS ECS Task Role recommended for credentials.
- Avoid embedding credentials.

---

## 📄 License

This product is licensed under the **Standard AWS Marketplace EULA**.  
See the AWS Marketplace listing for full terms and conditions.

---

## ⚠️ Disclaimer

This product is not a medical device.

It is designed to help healthcare institutions manage medical imaging data efficiently and enhance the quality of patient care. However, it is not intended to replace professional medical advice, diagnosis, or treatment.
