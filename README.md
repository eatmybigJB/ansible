```python
import boto3

client = boto3.client('cognito-idp')

def initiate_auth(username, password, client_id):
    response = client.initiate_auth(
        AuthFlow='USER_PASSWORD_AUTH',
        AuthParameters={
            'USERNAME': username,
            'PASSWORD': password
        },
        ClientId=client_id
    )
    return response

