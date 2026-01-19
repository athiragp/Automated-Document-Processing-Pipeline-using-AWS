# Automated-Document-Processing-Pipeline-using-AWS
Serverless KYC document processing pipeline using AWS. Documents uploaded to S3 trigger Lambda for OCR with Textract, intelligent cleaning and classification using Amazon Bedrock, metadata storage in DynamoDB, and archival to a processed S3 bucket. Fully automated via CLI + cron.
