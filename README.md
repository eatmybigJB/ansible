```python
import boto3

# 创建 DynamoDB 资源
dynamodb = boto3.resource('dynamodb')

# 选择要查询的表
table_name = 'YourTableName'
table = dynamodb.Table(table_name)

# 通过主键查询
response = table.get_item(
    Key={
        'PrimaryKey': '123'  # 假设主键字段为 'PrimaryKey'，值为 '123'
    }
)

# 检查响应中是否包含 'Item'
if 'Item' in response:
    item = response['Item']
    print("Item found:", item)
else:
    print("Item not found")