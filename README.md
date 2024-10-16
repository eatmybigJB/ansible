```python
import boto3
import hmac
import hashlib
import base64

# 创建 Cognito IDP 客户端
client = boto3.client('cognito-idp')

# 定义用户池和客户端信息
user_pool_id = '<Your_User_Pool_ID>'
client_id = '<Your_App_Client_ID>'
client_secret = '<Your_App_Client_Secret>'
username = 'new_user'
password = 'YourPassword123!'  # 密码需符合 Cognito 密码策略

# 生成 SecretHash
def generate_secret_hash(username, client_id, client_secret):
    message = username + client_id
    dig = hmac.new(
        client_secret.encode('utf-8'),
        msg=message.encode('utf-8'),
        digestmod=hashlib.sha256
    )
    return base64.b64encode(dig.digest()).decode()

secret_hash = generate_secret_hash(username, client_id, client_secret)

# 调用 sign_up 来创建用户
try:
    response = client.sign_up(
        ClientId=client_id,
        Username=username,
        Password=password,
        SecretHash=secret_hash,  # 包含 SecretHash
        UserAttributes=[
            {'Name': 'email', 'Value': 'user@example.com'},
            {'Name': 'email_verified', 'Value': 'true'}  # 如果邮箱已验证
        ]
    )
    
    print("Sign up successful:", response)

except client.exceptions.UsernameExistsException:
    print(f"Username {username} already exists.")
except client.exceptions.InvalidParameterException as e:
    print(f"Invalid parameter: {str(e)}")
except Exception as e:
    print(f"Error signing up user: {str(e)}")