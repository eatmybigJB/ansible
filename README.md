```python
import boto3

# 创建 DynamoDB 资源
dynamodb = boto3.resource('dynamodb')

# 选择要插入数据的表
table_name = 'YourTableName'
table = dynamodb.Table(table_name)

# 插入的数据
item = {
    'PrimaryKey': '123',  # 假设你的主键字段是 'PrimaryKey'
    'Attribute1': 'Value1',
    'Attribute2': 'Value2',
    'Attribute3': 'Value3'
}

# 向 DynamoDB 表插入数据
response = table.put_item(Item=item)

# 打印响应
print("PutItem succeeded:", response)