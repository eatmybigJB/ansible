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

