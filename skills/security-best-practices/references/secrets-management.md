# Secrets Management

## Environment Variables

### Local Development (.env)

```bash
# .env.example (committed to repo)
DATABASE_URL=postgresql://user:password@localhost/dbname
SECRET_KEY=your-secret-key-here
API_KEY=your-api-key-here
REDIS_URL=redis://localhost:6379
```

```bash
# .env (not committed, added to .gitignore)
DATABASE_URL=postgresql://devuser:devpass@localhost:5432/myapp
SECRET_KEY=super-secret-random-string-min-32-chars
API_KEY=sk_live_xxxxxxxxxxxxxxxxxxxxxx
```

### Production Environment

Never commit `.env` files. Use platform-specific secret management:

**Docker:**
```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}  # From host env
    secrets:
      - api_key
      - db_password

secrets:
  api_key:
    file: ./secrets/api_key.txt
  db_password:
    file: ./secrets/db_password.txt
```

**Kubernetes:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  # base64 encoded values
  database-url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0Bsb2NhbGhvc3QvZGI=
  api-key: c2tfbGl2ZV94eHh4eHh4eHh4eHh4eHh4eHh4eHh4eHh4eHh4eA==
```

## AWS Secrets Manager

### Python

```python
import boto3
import json
from botocore.exceptions import ClientError

def get_secret(secret_name: str, region_name: str = "us-east-1") -> dict:
    """Get secret from AWS Secrets Manager."""
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        
        # Parse secret string
        if 'SecretString' in response:
            return json.loads(response['SecretString'])
        else:
            # Binary secret
            import base64
            decoded_binary_secret = base64.b64decode(response['SecretBinary'])
            return json.loads(decoded_binary_secret)
            
    except ClientError as e:
        error_code = e.response['Error']['Code']
        if error_code == 'ResourceNotFoundException':
            print(f"Secret {secret_name} not found")
        elif error_code == 'InvalidRequestException':
            print(f"Invalid request for secret {secret_name}")
        elif error_code == 'InvalidParameterException':
            print(f"Invalid parameter for secret {secret_name}")
        raise

# Usage
secrets = get_secret("prod/myapp/database")
db_password = secrets['password']
db_username = secrets['username']
```

### JavaScript/Node.js

```javascript
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManagerClient({ region: 'us-east-1' });

async function getSecret(secretName) {
    try {
        const response = await client.send(
            new GetSecretValueCommand({
                SecretId: secretName,
                VersionStage: 'AWSCURRENT',
            })
        );
        
        if (response.SecretString) {
            return JSON.parse(response.SecretString);
        }
        
        // Handle binary secret
        const buff = Buffer.from(response.SecretBinary, 'base64');
        return JSON.parse(buff.toString('ascii'));
        
    } catch (error) {
        console.error('Error retrieving secret:', error);
        throw error;
    }
}

// Usage
const secrets = await getSecret('prod/myapp/database');
const dbPassword = secrets.password;
```

## HashiCorp Vault

### Python

```python
import hvac
import os

def get_vault_client():
    """Initialize Vault client with approle authentication."""
    client = hvac.Client(url=os.environ['VAULT_ADDR'])
    
    # Authenticate with AppRole
    client.auth.approle.login(
        role_id=os.environ['VAULT_ROLE_ID'],
        secret_id=os.environ['VAULT_SECRET_ID']
    )
    
    return client

def get_database_credentials(client, mount_point='database', role='myapp'):
    """Get dynamic database credentials from Vault."""
    response = client.secrets.database.generate_credentials(
        name=role,
        mount_point=mount_point
    )
    
    return {
        'username': response['data']['username'],
        'password': response['data']['password'],
        'lease_id': response['lease_id'],
        'lease_duration': response['lease_duration']
    }

def get_static_secret(client, path):
    """Get static secret from Vault KV store."""
    response = client.secrets.kv.v2.read_secret_version(path=path)
    return response['data']['data']

# Usage
client = get_vault_client()
db_creds = get_database_credentials(client)
secret = get_static_secret(client, 'myapp/config')
```

### Kubernetes Integration

```python
# Using Kubernetes service account token
def get_vault_client_k8s():
    """Initialize Vault client with Kubernetes authentication."""
    client = hvac.Client(url='https://vault.example.com')
    
    # Read JWT token from service account
    with open('/var/run/secrets/kubernetes.io/serviceaccount/token') as f:
        jwt = f.read()
    
    # Authenticate with Kubernetes auth method
    client.auth.kubernetes.login(
        role='my-app',
        jwt=jwt
    )
    
    return client
```

## Google Cloud Secret Manager

### Python

```python
from google.cloud import secretmanager

def get_secret(project_id: str, secret_id: str, version_id: str = "latest") -> str:
    """Get secret from Google Cloud Secret Manager."""
    client = secretmanager.SecretManagerServiceClient()
    
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    
    response = client.access_secret_version(request={"name": name})
    
    # Decode payload
    payload = response.payload.data.decode("UTF-8")
    return payload

# Usage
api_key = get_secret("my-project", "api-key")
```

### JavaScript/Node.js

```javascript
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');

const client = new SecretManagerServiceClient();

async function getSecret(projectId, secretId, versionId = 'latest') {
    const name = `projects/${projectId}/secrets/${secretId}/versions/${versionId}`;
    
    const [response] = await client.accessSecretVersion({ name });
    const payload = response.payload.data.toString('utf8');
    
    return payload;
}

// Usage
const apiKey = await getSecret('my-project', 'api-key');
```

## Azure Key Vault

### Python

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

def get_secret(vault_url: str, secret_name: str) -> str:
    """Get secret from Azure Key Vault."""
    credential = DefaultAzureCredential()
    client = SecretClient(vault_url=vault_url, credential=credential)
    
    secret = client.get_secret(secret_name)
    return secret.value

# Usage
vault_url = "https://my-keyvault.vault.azure.net/"
db_password = get_secret(vault_url, "database-password")
```

## Secret Rotation

### Automatic Rotation with AWS Lambda

```python
import boto3
import secrets

def lambda_handler(event, context):
    """Rotate database credentials."""
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']
    
    # Get the secret manager client
    client = boto3.client('secretsmanager')
    
    if step == 'createSecret':
        # Generate new secret
        new_password = generate_password()
        
        # Store pending version
        client.put_secret_value(
            SecretId=arn,
            ClientRequestToken=token,
            SecretString=json.dumps({
                'username': 'admin',
                'password': new_password
            }),
            VersionStages=['AWSPENDING']
        )
        
    elif step == 'setSecret':
        # Update database with new credentials
        secret = client.get_secret_value(
            SecretId=arn,
            VersionStage='AWSPENDING'
        )
        secret_dict = json.loads(secret['SecretString'])
        
        update_database_password(
            secret_dict['username'],
            secret_dict['password']
        )
        
    elif step == 'testSecret':
        # Verify new credentials work
        secret = client.get_secret_value(
            SecretId=arn,
            VersionStage='AWSPENDING'
        )
        secret_dict = json.loads(secret['SecretString'])
        
        test_connection(
            secret_dict['username'],
            secret_dict['password']
        )
        
    elif step == 'finishSecret':
        # Promote pending version to current
        metadata = client.describe_secret(SecretId=arn)
        current_version = None
        
        for version in metadata['VersionIdsToStages']:
            if 'AWSCURRENT' in metadata['VersionIdsToStages'][version]:
                current_version = version
                break
        
        # Move previous version to AWSPREVIOUS
        if current_version:
            client.update_secret_version_stage(
                SecretId=arn,
                VersionStage='AWSPREVIOUS',
                MoveToVersionId=current_version,
                RemoveFromVersionId=token
            )
        
        # Promote new version to AWSCURRENT
        client.update_secret_version_stage(
            SecretId=arn,
            VersionStage='AWSCURRENT',
            MoveToVersionId=token
        )

def generate_password(length=32):
    """Generate secure random password."""
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*"
    return ''.join(secrets.choice(alphabet) for _ in range(length))
```

## Best Practices

### 1. Never Hardcode Secrets

```python
# ❌ BAD
API_KEY = "sk_live_1234567890abcdef"

# ✅ GOOD
import os
API_KEY = os.environ.get('API_KEY')
if not API_KEY:
    raise ValueError("API_KEY environment variable not set")
```

### 2. Use Short-Lived Credentials

```python
# Get dynamic credentials from Vault
credentials = get_database_credentials()
# credentials expire automatically after TTL
```

### 3. Encrypt Secrets at Rest

```python
from cryptography.fernet import Fernet

def encrypt_secret(secret: str, key: bytes) -> bytes:
    """Encrypt secret before storing."""
    f = Fernet(key)
    return f.encrypt(secret.encode())

def decrypt_secret(encrypted: bytes, key: bytes) -> str:
    """Decrypt secret."""
    f = Fernet(key)
    return f.decrypt(encrypted).decode()
```

### 4. Audit Secret Access

```python
import logging

def get_secret_with_audit(secret_name: str, user: str) -> str:
    """Get secret with audit logging."""
    logger.info(f"User {user} accessing secret {secret_name}")
    
    secret = get_secret(secret_name)
    
    logger.info(f"User {user} successfully retrieved secret {secret_name}")
    return secret
```

### 5. Implement Least Privilege

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:region:account:secret:prod/myapp/*"
      ]
    }
  ]
}
```

### 6. Use Separate Secrets per Environment

```
prod/myapp/database
staging/myapp/database
dev/myapp/database
```

### 7. Rotate Regularly

- Set up automatic rotation (AWS: every 30 days)
- Manual rotation after employee departure
- Rotation after security incident
- Rotation if secret might be compromised

### 8. Monitor Secret Access

```python
# Set up CloudWatch alarms for unusual access
import boto3

cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_alarm(
    AlarmName='SecretAccessAnomaly',
    MetricName='GetSecretValue',
    Namespace='AWS/SecretsManager',
    Statistic='Sum',
    Period=300,
    EvaluationPeriods=1,
    Threshold=100,
    ComparisonOperator='GreaterThanThreshold'
)
```
