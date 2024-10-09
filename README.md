```python
import boto3

# 创建 Cognito 客户端
client = boto3.client('cognito-idp')

# 设置用户池 ID
user_pool_id = 'us-west-2_example'  # 替换为你的用户池 ID

# 要查找的自定义属性的值
employee_id_to_search = '12345'  # 替换为要查找的 employeeId 值

# 查询用户池中的用户
response = client.list_users(
    UserPoolId=user_pool_id,
    Filter=f'custom:employeeId = "{employee_id_to_search}"'
)

# 输出查询结果
for user in response['Users']:
    print(f'Username: {user["Username"]}')
    print(f'Attributes: {user["Attributes"]}')