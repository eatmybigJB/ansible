```python
import smtplib
import ssl
from email.mime.text import MIMEText

# 配置 SMTP 服务器信息
smtp_server = "smtp.example.com"
port = 465  # 如果是使用 SSL，通常是465
sender_email = "your_email@example.com"
receiver_email = "receiver@example.com"
password = "your_password"

# 邮件内容
message = MIMEText("This is a test email")
message["Subject"] = "Test Email"
message["From"] = sender_email
message["To"] = receiver_email

# 创建自定义的 SSL 上下文，并加载 CA 证书
context = ssl.create_default_context(cafile="/path/to/ca_certificate.pem")

try:
    # 使用 SMTP_SSL 连接到服务器，并使用自定义的 SSL 上下文
    with smtplib.SMTP_SSL(smtp_server, port, context=context) as server:
        server.login(sender_email, password)
        server.sendmail(sender_email, receiver_email, message.as_string())
    print("Email sent successfully!")
except Exception as e:
    print(f"Failed to send email: {e}")