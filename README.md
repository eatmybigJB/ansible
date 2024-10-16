```python
import boto3

# 创建 Cognito Identity Provider (IDP) 客户端
client = boto3.client('cognito-idp')

# 定义用户池 ID 和要确认的用户名
user_pool_id = '<Your_User_Pool_ID>'
username = '<Username_To_Confirm>'

try:
    # 调用 admin_confirm_sign_up 来确认用户
    response = client.admin_confirm_sign_up(
        UserPoolId=user_pool_id,
        Username=username
    )
    
    print("User confirmed successfully:", response)

except client.exceptions.UserNotFoundException:
    print(f"User {username} does not exist.")
except client.exceptions.NotAuthorizedException:
    print(f"User {username} is already confirmed or cannot be confirmed.")
except Exception as e:
    print(f"Error confirming user: {str(e)}")