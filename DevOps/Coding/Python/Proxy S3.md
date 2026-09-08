Proxy S3 is used to connect to S3 bucket
```python
from flask import Flask, request, jsonify, Response
import boto3
from botocore.exceptions import ClientError

app = Flask(__name__)

AWS_ACCESS_KEY = 'K63801Xga0kykQ_qvs'
AWS_SECRET_KEY = 'LC8dLtI76zF0O7kbgzr_y9_UUDxBu1yEO9hqXuFesJQ'
AWS_REGION = 'us-east-1'
S3_BUCKET_NAME = 'dashboard-doanhthu-cattiensanet'
S3_ENDPOINT = 'https://34992.oss.swiftserve.com'

s3_client = boto3.client(
    's3',
    aws_access_key_id=AWS_ACCESS_KEY,
    aws_secret_access_key=AWS_SECRET_KEY,
    region_name=AWS_REGION,
    endpoint_url=S3_ENDPOINT
)

@app.route('/<path:filename>', methods=['GET'])
def get_s3_file(filename):
    try:
        response = s3_client.get_object(Bucket=S3_BUCKET_NAME, Key=filename)
        content = response['Body'].read()
        content_type = response.get('ContentType', 'application/octet-stream')
        return Response(content, content_type=content_type)
    except ClientError as e:
        print(f"[ERROR] {filename} -> {e}")
        return jsonify({'error': str(e)}), 404

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```