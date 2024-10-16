```python
import boto3
import boto3

# 创建 Cognito Identity Provider (IDP) 客户端
client = boto3.client('cognito-idp')

# 定义要使用的 Cognito 用户池 ID 和用户信息
user_pool_id = '<Your_User_Pool_ID>'
username = 'new_user'
temporary_password = 'TempPassword123!'  # 临时密码（需要符合 Cognito 密码策略）

try:
    # 调用 admin_create_user 来创建用户
    response = client.admin_create_user(
        UserPoolId=user_pool_id,
        Username=username,
        TemporaryPassword=temporary_password,
        UserAttributes=[
            {'Name': 'email', 'Value': 'user@example.com'},
            {'Name': 'email_verified', 'Value': 'True'}
        ],
        MessageAction='SUPPRESS'  # 禁止 Cognito 自动发送邮件或短信（可选）
    )
    
    print("User created successfully:", response)
    
except client.exceptions.UsernameExistsException:
    print(f"Username {username} already exists.")
except Exception as e:
    print(f"Error creating user: {str(e)}")