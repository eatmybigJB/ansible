```python
import base64
import smtplib
import requests
import ssl
import json
import os
import botocore
import aws_encryption_sdk
from aws_encryption_sdk.identifiers import CommitmentPolicy
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
import templates.dev.mdp as mdpconfig

mdp_one_token_url = mdpconfig.one_token_url
mdp_one_token_headers = mdpconfig.one_token_headers
mdp_one_token_payload = mdpconfig.one_token_payload

mdp_send_url = mdpconfig.send_url
mdp_send_headers = mdpconfig.send_headers
mdp_send_payload = mdpconfig.send_payload

def color_text(words, color_name):
    color_codes = {
        'green': 32,
        'yellow': 33
    }
    color_code = color_codes.get(color_name, 0)  # 默认颜色代码为0（无颜色）
    return f'\033[{color_code}m{words}\033[0m'

def get_one_token(url, headers, payload, ca_cert_path):
    response = requests.post(url=url, headers=headers, json=payload, verify=ca_cert_path)
    if response.status_code == 200:
        jwt_token = response.json().get("issued_token")
        print(f"{color_text('JWT Token', 'yellow')}: {color_text(jwt_token, 'green')}")
        return jwt_token
    else:
        error_message = f"Failed to get token: {response.status_code} - {response.text}"
        print(error_message)
        raise Exception(error_message)

def send(jwt_token, url, phone_number, decrypted_plaintext, ca_cert_path):
    mdp_send_headers['X-HSBC-E2E-Trust-Token'] = jwt_token
    mdp_send_payload['phoneNumber'] = phone_number
    mdp_send_payload['additionalDetails'][0]['value'] = decrypted_plaintext

    response = requests.post(url=url, json=mdp_send_payload, headers=mdp_send_headers, verify=ca_cert_path)

    if response.status_code == 200:
        print(f"{color_text('Success Response: ', 'yellow')}: {color_text(response.json(), 'green')}")
        print(mdp_send_payload)
        return response
    else:
        error_message = f"Failed to send SMS: {response.status_code} - {response.text}"
        print(error_message)
        raise Exception(error_message)

def send_email(subject, body, to_email, from_email, smtp_server, smtp_port, smtp_user, smtp_password):
    msg = MIMEMultipart()
    msg['From'] = from_email
    msg['To'] = to_email
    msg['Subject'] = subject
    msg.attach(MIMEText(body, 'plain'))

    context = ssl._create_unverified_context(check_hostname=False)

    try:
        with smtplib.SMTP(smtp_server, smtp_port, timeout=30) as server:
            server.starttls(context=context)
            server.login(smtp_user, smtp_password)
            server.sendmail(from_email, to_email, msg.as_string())
        print("Email sent successfully!")
    except Exception as e:
        print(f"Failed to send email: {str(e)}")

class DecryptionError(Exception):
    pass

def get_encrypted_code(encrypted_code, key_arn):
    try:
        encrypted_code_bytes = base64.b64decode(encrypted_code)
        print(f"Decoded Encrypted Code (Bytes): {encrypted_code_bytes}")

        client = aws_encryption_sdk.EncryptionSDKClient(
            commitment_policy=CommitmentPolicy.FORBID_ENCRYPT_ALLOW_DECRYPT
        )

        kms_key_provider = aws_encryption_sdk.StrictAwsKmsMasterKeyProvider(key_ids=[key_arn])

        decrypted_plaintext, _ = client.decrypt(
            source=encrypted_code_bytes,
            key_provider=kms_key_provider
        )
        return decrypted_plaintext.decode('utf-8')

    except (aws_encryption_sdk.exceptions.AWSEncryptionSDKClientError,
            botocore.exceptions.BotoCoreError,
            botocore.exceptions.ClientError) as e:
        error_message = f"Error decrypting verification code: {str(e)}"
        print(error_message)
        raise DecryptionError(error_message)

subject = "Test Email"
from_email = "ADRESTEST-SVC-B2B@adrestest.addev.hsbc"
smtp_server = "dynip-devapp-smtp-int-ext-relay.hk.hsbc"
smtp_port = 2525
smtp_user = "ADRESTEST-SVC-B2B"
smtp_password = "hcraM@evaeLenOsieiddE"

script_dir = os.path.dirname(os.path.realpath(__file__))
ca_cert_path = os.path.join(script_dir, 'cert.pem')
key_arn = 'arn:aws:kms:ap-east-1:730335524365:key/d1584f39-4fef-4c04-a396-fdb180ec04a8'

def lambda_handler(event, context):
    try:
        encrypted_code = event['request']['code']
        triggerSource = event['triggerSource']
        userAttributes = event['request']['userAttributes']
        print(event)

        try:
            decrypted_plaintext = get_encrypted_code(encrypted_code, key_arn)
            print(decrypted_plaintext)
        except DecryptionError as e:
            print(f"Verification code decryption failed: {str(e)}")
            return {
                'statusCode': 500,
                'body': json.dumps('Verification code decryption failed, unable to proceed')
            }

        if triggerSource == 'CustomEmailSender_Authentication':
            to_email = userAttributes['email']
            body = f"your code is {decrypted_plaintext}"
            print(to_email)
            try:
                send_email(subject, body, to_email, from_email, smtp_server, smtp_port, smtp_user, smtp_password)
            except Exception as e:
                print(f"Failed to send email: {str(e)}")
                return {
                    'statusCode': 500,
                    'body': json.dumps('Failed to send email')
                }

        if triggerSource == 'CustomSMSSender_Authentication':
            phone_number = userAttributes['phone_number']
            try:
                jwt_token = get_one_token(mdp_one_token_url, mdp_one_token_headers, mdp_one_token_payload, ca_cert_path)
                send(jwt_token, mdp_send_url, phone_number, decrypted_plaintext, ca_cert_path)
            except Exception as e:
                print(f"Failed to send SMS: {str(e)}")
                return {
                    'statusCode': 500,
                    'body': json.dumps('Failed to send SMS')
                }

        return {
            'statusCode': 200,
            'body': {
                'decryptedCode': decrypted_plaintext
            }
        }

    except Exception as e:
        print(f"Error during decryption: {str(e)}")
        return {
            'statusCode': 500,
            'body': 'Error during KMS decryption'
        }
