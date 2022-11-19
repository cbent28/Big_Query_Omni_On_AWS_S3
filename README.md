Using BigQuery Omni with AWS

![image](https://user-images.githubusercontent.com/104800728/202823320-1efe4092-24fc-4061-902a-cfcd36e114d5.png)

Overview
================
In this lab, you will learn how to use BigQuery Omni with AWS. BigQuery Omni lets you run BigQuery analytics on data stored in AWS S3. You will create an authorized connection between Google Cloud BigQuery and AWS S3, query data residing in S3 buckets without any data movement and write query results back to AWS S3 buckets. This lab is derived from a BigQuery Omni Guide published by Google.

High Level Architecture
----------------------

![image](https://user-images.githubusercontent.com/104800728/202823390-b982556e-69a8-4fc2-8373-14f5ecd9a179.png)

Objectives
====================
In this lab, you learn how to perform the following tasks:
- Create a connection between Google Cloud and AWS
- Authorize BigQuery Omni to read data in an AWS S3 bucket
- Create a BigQuery external table that references the raw data in AWS S3 bucket  
- Run queries on AWS S3 data
- Export query results to AWS S3 bucket


Create an AWS IAM policy for BigQuery
====================================
BigQuery Omni accesses Amazon S3 data through authorized connections from Google Cloud. Each connection has its own unique Amazon Web Services (AWS) Identity and Access Management (IAM) user. You grant permissions to users through AWS IAM roles. The policies within the AWS IAM roles determine what data BigQuery can access for each connection.

Create an AWS IAM policy for BigQuery 
=====================================
1. Sign in to the AWS Management Console. Click the Open AWS Console button on the lab pane, and log in with the provided username and password.
2. Search for Amazon S3 in the Search bar at the top and select S3. A regional bucket named S3 bucket name with data pre-populated is already available for this lab. Copy this bucket name for subsequent steps.
3. Search for AWS Identity and Access Management (IAM) in the Search bar at the top and select IAM.
4. From the left pane select Policies and click bigquery-omni-connection-policy.
5. Click Edit policy > JSON and paste the following into the editor. Replace all instances of <BUCKET_NAME> with your S3 bucket name copied from Step 2.

```
{
    "Statement": [
        {
            "Action": [
                "s3:ListBucket"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>"
            ],
            "Sid": "BucketLevelAccess"
        },
        {
            "Action": [
                "s3:GetObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>",
                "arn:aws:s3:::<BUCKET_NAME>/*"
            ],
            "Sid": "ObjectLevelGetAccess"
        },
        {
            "Action": [
                "s3:PutObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>",
                "arn:aws:s3:::<BUCKET_NAME>/*"
            ],
            "Sid": "ObjectLevelPutAccess"
        }
    ],
    "Version": "2012-10-17"
}
```

6. Click Review policy.
7. Click Save changes.

Validate the AWS IAM for BigQuery
=================================

1. From the left pane, select Roles.
2. Click the bigquery-omni-connection role.
3. Copy the Role ARN, it will be in the following format, where <ACCOUNT_ID> is your AWS Account ID.

```
arn:aws:iam::<ACCOUNT_ID>:role/bigquery-omni-connection
```

![image](https://user-images.githubusercontent.com/104800728/202824026-64bbf483-7c49-4c84-8b30-5a2cbbe3296d.png)

Create the BigQuery AWS connection
=============================

1. In the Google Cloud Console, from the Navigation Menu, go to BigQuery > SQL Workspace.
2. Click Add data +, then select Connections to external data sources.
3. In the External data source pane, enter the following information:
  - For Connection type, select BigLake on AWS (via BigQuery Omni).
  - For Connection ID, type ```bq-omni-aws-connector``` for an identifier for the connection resource.
  - For Connection location, select ```aws-us-east-1```.
- Optional: For Friendly name, enter a user-friendly name for the connection. The friendly name can be any value that helps you identify the connection resource if you need to modify it later.
- Optional: For Description, enter a description for this connection resource.
- For AWS role id, enter the full IAM Role ARN that you copied in the previous step in this format: ```arn:aws:iam::AWS_ACCOUNT_ID:role/ROLE_NAME```

4. Click Create connection.
5. In the BigQuery Explorer, click the dropdown next to your project name and navigate to the newly created connection in the External Connections list.

![image](https://user-images.githubusercontent.com/104800728/202824253-b0cd0f8f-81eb-4ed4-99c5-399aec3b14c5.png)

6. Note the BigQuery Google identity. This is a Google principle that is specific to each connection. Copy this BigQuery Google identity, it will be used in the next section.

Your BigQuery Google Identity should resemble the following:

```
BigQuery Google identity: 114999259451445753095
```
Click Check my progress to verify the objective.

Add a Trust Relationship to the AWS role
=====================================

The trust relationship lets the BigQuery AWS connection assume the role and access the S3 data as specified in the roles policy.

1. Navigate back to the AWS IAM console.
2. From the left pane, select Roles.
3. Select the bigquery-omni-connection role.
4. Click Edit and then do the following:
- Verify if Maximum session duration is set to 12 hours. As each query can run for up to six hours, this duration allows for one additional retry. Increasing the session duration beyond 12 hours will not allow for additional retries. For more information, see the query/multi-statement query execution-time limit.
- Click Save changes.
5. Select Trust Relationships tab and click Edit trust policy.

6. Replace the policy content with the following, replacing <IDENTITY_ID> with the BigQuery Google identity you copied in the previous section.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "accounts.google.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "accounts.google.com:sub": "<IDENTITY_ID>"
        }
      }
    }
  ]
}

```
7. Click Update Policy.
The connection is now ready to use.

Run queries on the AWS S3 external table
==============================

BigQuery Omni does not manage data stored in Amazon S3. To access S3 data, define an external table. This table is called an external table because the data is not stored in BigQuery managed storage. For more information about external tables, see External tables.

Create a BigQuery dataset
---------------------
In this section, you will create a BigQuery dataset in the same region as your AWS S3 bucket.
1. In the Google Cloud Console, go to the BigQuery page to create a dataset.
2. Click the 3 dots next to your project name and select Create dataset.
3. On the Create dataset page, enter the following information:
- For Dataset ID, enter ```bq_omni_demo```.
- For Data location, choose ```aws-us-east-1```.
4. Click Create dataset.

Create an external table
In this section you will create an external table in the above dataset.
1. In this BigQuery explorer, expand your project and select the ```bq_omno_demo``` dataset created.
2. In the details panel, click Create table.
3. On the Create table page, in the Source section, do the following:
- For Create table from, select Amazon S3.
- For Select S3 path, enter ```s3:// S3 bucket name /taxi-data_green_trips_table.csv```.
- For File format, select CSV. Note: supported formats are AVRO, PARQUET, ORC, CSV, NEWLINE_DELIMITED_JSON, and Google Sheets.
4. On the Create table page, in the Destination section, do the following:
- For Dataset name, choose ```bq_omni_demo```.
- In the Table name field, use ```bq-omni-table```.
- Verify that Table type is set to External table.
- For Connection ID, choose the appropriate Connection ID from the dropdown.
- In the Schema section, select the Auto detect checkbox.
5. Click Create table.
Click Check my progress to verify the objective.

Create an external table and query AWS S3 data
===================================

BigQuery Omni lets you query the external table like any BigQuery table. The maximum result size for interactive queries is 10 GB (preview). For more information, see Limitations. If your query result is larger than 10 GB, then we recommend that you export it to Amazon S3. The query result is stored in a BigQuery temporary table.

Query the external table
---------------
1. From the bq-omni-table details page, select Query > In new tab.
query bigquery omni table with in new tab selction highlighted

![image](https://user-images.githubusercontent.com/104800728/202824974-2bd6174f-e17b-4726-a865-0078a8a4d164.png)

2. In the Query editor, execute the following statement:

```
SELECT * FROM `S3 bucket name.bq_omni_demo.bq-omni-table`
``` 
 
3. Click Run.
You should see the following output:

![image](https://user-images.githubusercontent.com/104800728/202825044-6140ab75-1b00-48e2-abbd-37293566a144.png)

Click Check my progress to verify the objective.

Export query results to AWS S3
===============

BigQuery Omni lets you export the result of a query against a BigQuery external table to Amazon S3.

Export Query Results
----------------------
BigQuery Omni writes to the specified Amazon S3 location irrespective of any existing content. The export query can overwrite existing data or mix the query result with existing data. In the Query editor field, you will need to run a Google Standard SQL export query. Google Standard SQL is the default syntax in the Google Cloud console. The following is the template for what you will need to write:

```
EXPORT DATA WITH CONNECTION `CONNECTION_REGION.CONNECTION_NAME` \
OPTIONS(uri="s3://BUCKET_NAME/PATH", format="FORMAT", ...) \
AS QUERY
```
You will need to replace the following:
- ```CONNECTION_REGION```: the region where the connection was created.
- ```CONNECTION_NAME```: the connection name that you created with the necessary permission to write to the S3 bucket.
- ```BUCKET_NAME```: the Amazon S3 bucket where you want to write the data.
- ```PATH```: the path where you want to write the exported file to
- ```FORMAT```: supported formats are JSON, AVRO, and CSV.
- ```QUERY```: the query to analyze the data that is stored in a BigQuery external table.

For this lab, the query has been pre-populated for you. Paste this query into the editor:

```
EXPORT DATA WITH CONNECTION `aws-us-east-1.bq-omni-aws-connector`
OPTIONS(uri="s3://S3 bucket name/exports/*", format="CSV")
AS SELECT * FROM `S3 bucket name.bq_omni_demo.bq-omni-table`
```

2.Click Run.
You should see the following output:

```
Successfully exported 15417 rows into 1 files.
```

3. Navigate to your S3 bucket and verify the data has been exported in the exports directory.

![image](https://user-images.githubusercontent.com/104800728/202825279-3af1343a-098b-4a53-b0cb-fbcd39c1dc2c.png)

Great! You have successfully executed an export query and created a file in your S3 bucket.

Congratulations!
===============

In this lab you created a connection between Google Cloud and AWS and authorized BigQuery Omni to read the data in an AWS S3 bucket. You then created a BigQuery external table that references the raw data in the AWS S3 bucket, ran queries on the data, and exported query results back to an AWS S3 bucket.
