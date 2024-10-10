```python
import boto3

# 创建 Cognito 客户端
client = boto3.client('cognito-idp')

# 用户池 ID
user_pool_id = 'us-west-2_example'

# 使用标准属性 email 进行查询
response = client.list_users(
    UserPoolId=user_pool_id,
    Filter='email = "test@example.com"'
)

# 输出用户属性，检查自定义属性是否存在
for user in response['Users']:
    print(f'Username: {user["Username"]}')
    for attribute in user['Attributes']:
        print(f'{attribute["Name"]}: {attribute["Value"]}')
