```python
import boto3

# 创建 DynamoDB 客户端
dynamodb = boto3.client('dynamodb')

# 定义表的属性和配置
table_name = 'MyDynamoDBTable'

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

# 打印创建表的响应
print("Table created successfully:", response)