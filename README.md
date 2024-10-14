```python
import boto3

def lambda_handler(event, context):
    # 初始化 S3 客户端
    s3 = boto3.client('s3')
    
    # 从事件中获取 S3 存储桶名称和要删除的对象键
    bucket_name = event['bucket_name']  # S3 存储桶名称
    object_key = event['object_key']    # 要删除的对象键（文件路径）
    
    # 删除对象
    try:
        response = s3.delete_object(Bucket=bucket_name, Key=object_key)
        
        # 打印出删除操作的 response
        print("Delete Object Response:", response)
        
        return {
            'statusCode': 200,
            'body': f"文件 '{object_key}' 已从存储桶 '{bucket_name}' 中成功删除。",
            'response': response
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': f"删除文件时出错: {str(e)}"
        }