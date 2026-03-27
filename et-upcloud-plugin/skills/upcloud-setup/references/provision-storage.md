# Provision Object Storage Playbook

## Step 1: Determine the Correct Region

Object storage regions differ from server zones. Use this EU-only mapping:

| Server Zone | Object Storage Region |
|-------------|----------------------|
| `fi-hel1`   | `FI-HEL2`           |
| `se-sto1`   | `SE-STO1`           |
| `de-fra1`   | `DE-FRA1`           |

Non-EU regions (`SG-SIN1`, `US-CHI1`) must NOT be used — EU data residency policy.

Verify with:
```bash
upctl object-storage regions
```

## Step 2: Create Object Storage Instance

```bash
upctl object-storage create \
  --name "{project}-uploads" \
  --region "{region}" \
  --network type=public,name=public,family=IPv4 \
  --wait
```

## Step 3: Create Service User

```bash
upctl object-storage user create \
  --service-uuid "{storage_uuid}" \
  --username "{project}-app"
```

## Step 4: Create Access Keys

```bash
ACCESS_KEY_INFO=$(upctl object-storage access-key create \
  --service-uuid "{storage_uuid}" \
  --username "{project}-app" \
  -o json)

ACCESS_KEY=$(echo "$ACCESS_KEY_INFO" | jq -r '.access_key_id')
SECRET_KEY=$(echo "$ACCESS_KEY_INFO" | jq -r '.secret_access_key')
```

## Step 5: Create Default Bucket

```bash
upctl object-storage bucket create \
  --service-uuid "{storage_uuid}" \
  --name "{project}-uploads"
```

## Step 6: Store Credentials in Infisical

```
S3_ENDPOINT=https://{project}-uploads.{region}.upcloudobjects.com
S3_ACCESS_KEY={access_key}
S3_SECRET_KEY={secret_key}
S3_BUCKET={project}-uploads
S3_REGION={region}
```

## Usage with AWS SDK

UpCloud Object Storage is S3-compatible. Use any S3 SDK:

```typescript
import { S3Client } from "@aws-sdk/client-s3";

const s3 = new S3Client({
  endpoint: process.env.S3_ENDPOINT,
  region: process.env.S3_REGION,
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY,
    secretAccessKey: process.env.S3_SECRET_KEY,
  },
  forcePathStyle: true,
});
```

## Notes

- Object Storage is S3-compatible — works with any S3 SDK or tool (aws-cli, rclone, etc.)
- Access keys are scoped per user — create separate users for different access levels
- Pricing: ~EUR 5/month for 250GB
