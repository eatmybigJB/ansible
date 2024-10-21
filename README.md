```python
import boto3
from botocore.exceptions import ClientError

# 创建 DynamoDB 客户端
dynamodb = boto3.client('dynamodb')

# 定义表的属性和配置
table_name = 'MyDynamoDBTable'

try:
    # 尝试创建 DynamoDB 表
    response = dynamodb.create_table(
        TableName=table_name,
        KeySchema=[
            {
                'AttributeName': 'PrimaryKey',
                'KeyType': 'HASH'  # 分区键
            }
        ],
        AttributeDefinitions=[
            {
                'AttributeName': 'PrimaryKey',
                'AttributeType': 'S'  # 字符串类型
            }
        ],
        ProvisionedThroughput={
            'ReadCapacityUnits': 5,
            'WriteCapacityUnits': 5
        }
    )
    print("Table created successfully:", response)

except ClientError as e:
    # 捕获 AWS 客户端错误并显示详细信息
    error_code = e.response['Error']['Code']
    error_message = e.response['Error']['Message']
    
    if error_code == 'ResourceInUseException':
        print(f"Error: Table {table_name} already exists.")
    elif error_code == 'LimitExceededException':
        print("Error: DynamoDB table creation limit exceeded.")
    elif error_code == 'ProvisionedThroughputExceededException':
        print("Error: Provisioned throughput limits exceeded.")
    elif error_code == 'RequestTimeout':
        print("Error: The request timed out.")
    else:
        print(f"Error: {error_code} - {error_message}")

except Exception as e:
    # 捕获其他异常并显示详细信息
    print(f"An unexpected error occurred: {str(e)}")