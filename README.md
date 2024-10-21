```python
import os
import boto3

# 从环境变量中获取访问密钥和秘密密钥
access_key = os.environ['AWS_ACCESS_KEY_ID']
secret_key = os.environ['AWS_SECRET_ACCESS_KEY']
region_name = "your-region"

# 创建 DynamoDB 客户端
dynamodb = boto3.client(
    'dynamodb',
    aws_access_key_id=access_key,
    aws_secret_access_key=secret_key,
    region_name=region_name
)

# 示例操作 - 列出 DynamoDB 表
response = dynamodb.list_tables()
print("DynamoDB Tables:", response['TableNames'])