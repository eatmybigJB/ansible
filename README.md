```python
import requests
import os
import configparser
import json
import aws_encryption_sdk
from aws_encryption_sdk import CommitmentPolicy
from aws_encryption_sdk.key_providers.kms import KMSMasterKeyProvider
import botocore
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import ssl

def load_config():
    config = configparser.ConfigParser()
    env = os.environ.get('ENVIRONMENT', 'dev')
    config_path = f'config/{env}/config.ini'
    config.read(config_path)
    return config

CONFIG = load_config()

# Certificate file path
CERT_PATH = os.path.join(os.path.dirname(__file__), 'cert.pem')

# Initialize KMS Master Key Provider
kms_arn = CONFIG['AWS']['kms_arn']
kms_key_provider = KMSMasterKeyProvider(key_ids=[kms_arn])

def get_jwt_token():
    """
    Get JWT token
    
    :return: JWT token string or None (if failed to obtain)
    """
    auth_url = CONFIG['JWT_TOKEN']['url']
    username = CONFIG['JWT_TOKEN']['username']
    password = CONFIG['JWT_TOKEN']['password']
    
    headers = json.loads(CONFIG['JWT_TOKEN']['headers'])
    payload_str = CONFIG['JWT_TOKEN']['payload'].replace('{username}', username).replace('{password}', password)
    payload = json.loads(payload_str)
    
    try:
        response = requests.post(auth_url, json=payload, headers=headers, verify=CERT_PATH)
        response.raise_for_status()
        result = response.json()
        return result.get("output_token_state", {}).get("token")
    except requests.RequestException as e:
        print(f"Error obtaining JWT token: {e}")
        return None

def decrypt_verification_code(encrypted_code):
    """
    Decrypt verification code
    
    :param encrypted_code: Encrypted verification code
    :return: Decrypted verification code
    :raises: DecryptionError if decryption fails
    """
    try:
        client = aws_encryption_sdk.EncryptionSDKClient(commitment_policy=CommitmentPolicy.REQUIRE_ENCRYPT_ALLOW_DECRYPT)
        decrypted_code, _ = client.decrypt(
            source=encrypted_code,
            key_provider=kms_key_provider
        )
        return decrypted_code.decode('utf-8')
    except (aws_encryption_sdk.exceptions.AWSEncryptionSDKClientError,
            botocore.exceptions.BotoCoreError,
            botocore.exceptions.ClientError) as e:
        error_message = f"Error decrypting verification code: {str(e)}"
        print(error_message)
        raise DecryptionError(error_message)

class DecryptionError(Exception):
    """Custom exception to represent decryption failure"""
    pass

def send_verification_code(phone_number, verification_code):
    """
    Send verification code
    
    :param phone_number: Phone number to receive the verification code
    :param verification_code: Decrypted verification code
    :return: Boolean indicating whether sending was successful
    """
    jwt_token = get_jwt_token()
    if not jwt_token:
        print("Unable to obtain JWT token, verification code sending failed")
        return False
    
    api_url = CONFIG['VERIFICATION']['url']
    
    # Construct request headers
    headers_str = CONFIG['VERIFICATION']['headers'].replace('{jwt_token}', jwt_token)
    headers = json.loads(headers_str)
    
    # Construct request parameters
    payload_str = CONFIG['VERIFICATION']['payload'].replace('{phone_number}', phone_number).replace('{verification_code}', verification_code)
    payload = json.loads(payload_str)
    
    try:
        response = requests.post(api_url, json=payload, headers=headers, verify=CERT_PATH)
        response.raise_for_status()
        
        result = response.json()
        success = result.get("success", False)
        if success:
            print(f"Verification code sent to {phone_number}")
        return success
    except requests.RequestException as e:
        print(f"Error sending verification code: {e}")
        return False

def send_email_verification_code(email, verification_code):
    """
    Send verification code via email, ignoring server certificate
    
    :param email: Email address to receive the verification code
    :param verification_code: Decrypted verification code
    :return: Boolean indicating whether sending was successful
    """
    smtp_server = CONFIG['EMAIL']['smtp_server']
    smtp_port = int(CONFIG['EMAIL']['smtp_port'])
    smtp_username = CONFIG['EMAIL']['smtp_username']
    smtp_password = CONFIG['EMAIL']['smtp_password']
    sender_email = CONFIG['EMAIL']['sender_email']
    subject = CONFIG['EMAIL']['subject']
    body = CONFIG['EMAIL']['body'].format(verification_code=verification_code)
    
    message = MIMEMultipart()
    message['From'] = sender_email
    message['To'] = email
    message['Subject'] = subject
    
    message.attach(MIMEText(body, 'plain'))
    
    # Create SSL context that does not verify certificates
    context = ssl.create_default_context()
    context.check_hostname = False
    context.verify_mode = ssl.CERT_NONE
    
    try:
        with smtplib.SMTP(smtp_server, smtp_port) as server:
            server.starttls(context=context)  # Use custom SSL context
            server.login(smtp_username, smtp_password)
            server.send_message(message)
        print(f"Verification code email sent to {email}")
        return True
    except Exception as e:
        print(f"Error sending verification code email: {e}")
        return False

def extract_event_data(event):
    """
    Extract all required data from the event
    
    :param event: Lambda event object
    :return: Dictionary containing extracted data
    """
    return {
        'encrypted_code': event['request']['code'],
        'phone_number': event['request']['userAttributes']['phone_number'],
        'email': event['request']['userAttributes']['email'],
        'trigger_source': event['triggerSource'],
    }

# Lambda function entry point
def lambda_handler(event, context):
    try:
        # Extract event data
        event_data = extract_event_data(event)
        
        try:
            # Decrypt verification code
            verification_code = decrypt_verification_code(event_data['encrypted_code'])
        except DecryptionError as e:
            print(f"Verification code decryption failed: {str(e)}")
            return {
                'statusCode': 500,
                'body': json.dumps('Verification code decryption failed, unable to proceed')
            }
        
        # Choose sending method based on triggerSource
        if event_data['trigger_source'] == 'CustomSMSSender_Authentication':
            result = send_verification_code(event_data['phone_number'], verification_code)
            message = 'SMS verification code sent successfully' if result else 'SMS verification code sending failed'
        elif event_data['trigger_source'] == 'CustomEmailSender_Authentication':
            result = send_email_verification_code(event_data['email'], verification_code)
            message = 'Email verification code sent successfully' if result else 'Email verification code sending failed'
        else:
            print(f"Unsupported triggerSource: {event_data['trigger_source']}")
            return {
                'statusCode': 400,
                'body': json.dumps(f"Unsupported triggerSource: {event_data['trigger_source']}")
            }
        
        return {
            'statusCode': 200 if result else 400,
            'body': json.dumps(message)
        }
    except ValueError as e:
        print(f"Parameter error: {str(e)}")
        return {
            'statusCode': 400,
            'body': json.dumps(f'Parameter error: {str(e)}')
        }
    except Exception as e:
        print(f"Error during processing: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps('Internal server error')
        }
