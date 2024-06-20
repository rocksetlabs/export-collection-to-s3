# export-collection-to-s3
This is a text generation script. The generated text contains a CURL POST command to execute against Rockset’s endpoint API. The endpoint API used is the query endpoint because the CURL command will be issuing a SQL statement to Rockset. The SQL query will be a variation of INSERT INTO S3 based on the input paramters provided at run time.

# Purpose

The purpose of the program is not to directly run a query against Rockset. Rather, it’s purpose is to build a CURL POST command and a SQL statement for you. You can choose to adjust it as needed or execute it directly if the generated output is acceptable. That decision is yours.

This gives you flexibility to create and run this program however you wish. Perhaps you want to generate scripts and pass them onto teammates for execution as part of an ops effort. You could name the output file as a `myoutput.sh` and another team member could `chmod +x` and run it `./myoutput.sh`.

As an example, your generated output might look something like this:
```shell
curl --request POST \
     --url https://api.usw2a1.rockset.com/v1/orgs/self/queries \
     --header 'Authorization: ApiKey yourRSAPIKey' \
     --header 'accept: application/json' \
     --header 'content-type: application/json' \
     --data @- <<EOF
{
  "sql": {
    "query": "INSERT INTO 's3://mybucket' INTEGRATION = 'my-RS-intg' FORMAT = (TYPE='JSON', INCLUDE_QUERY_ID=true) SELECT * FROM myRSwsdotcoll"
  }
}
EOF
```
You could choose to further refine your script to include portioning or any other adjustments you require.

# Prerequisites

- Before exporting data from Rockset, stop all ingestion and processes (e.g. Query Lambdas)
- Python 3.x installed
- Access to an API Key for your Rockset org
- Write access to an S3 bucket and appropriate IAM Policy/Roles configured

# General recommendations

- Before exporting data from Rockset, stop all ingestion and processes (e.g. Query Lambdas)
- For most, it may be easiest to create 1 single S3 bucket for all exported Rockset collections
    - Different workspaces and collections can be exported to different paths in that bucket
- Use “async” mode for large amounts of data
- The query must execute within 30 minutes
- S3 Bucket must be in the same region as the Rockset collection
- Export to Parquet will generally be faster for larger datasets so thats something to consider whereas JSON may obviously be easier to read

# Running the Script

Open Terminal, navigate to the directory where `export_RSCollection_to_AWSS3.py` is saved and run the script with the required command-line arguments.

### Basic Usage

This program execution will create an output file named `export_mycollection_script.sh`.

Example: This example uses a read/write Rockset S3 Integration name (you can obtain this from the Rockset Console UI in the Integrations tab). You can use an existing valid one or create a new one in the UI and use this integration name here. Creating a new integration will require you to create the appropriate AWS IAM Policy and Role. This is typically our recommended option.
```shell
python3 export_RSCollection_to_AWSS3.py \
    --output_file export_mycollection_script.sh \
    --param_RS_region usw2a1 \
    --param_RS_apikey myRSAPIKey \
    --param_RS_wsdotcollectionname prod.mycollection \
    --param_RS_outputformat JSON \
    --param_AWS_S3bucketuri s3://myS3bucket \
    --param_RS_integrationname myRS_S3_IntegrationName
```

Example: This example does not use a Rockset Integration name, but rather a AWS IAM Role and AWS External ID. You can use those values directly. This can be useful if you do not want to use an existing Rockset S3 integration or create a new Rockset S3 integration. In this case, it assumes you have already setup an IAM Role and Policy that has read/write access. You can use the Rockset S3 setup wizard to help you create that Policy and Role and stop short of creating the integration, but the easiest path is to the example above.
```shell
python3 export_RSCollection_to_AWSS3.py \
    --output_file export_mycollection_script.sh \
    --param_RS_region usw2a1 \
    --param_RS_apikey myRSAPIKey \
    --param_RS_wsdotcollectionname prod.mycollection \
    --param_RS_outputformat JSON \
    --param_AWS_S3bucketuri s3://myS3bucket \
    --param_RS_AWSROLE_credentials myIAMroleARN \
    --param_RS_AWSEXTID_credentials myExternalID
```

### Command Line Arguments

- `--output_file` (required): Name of the output file that will contain the generated text
- `--param_RS_region` (required): The Rockset region your collection exists.
- `--param_RS_apikey` (required): Your Rockset API key.
- `--param_RS_wsdotcollectionname` (required): The name of your Rockset workspace.dot.collection. E.g. myproductionenv.mycollection
- `—param_RS_outputformat` (required): JSON or PARQUET
- `—param_AWS_S3bucketuri` (required): Your S3 bucket uri
- `—param_RS_AWSROLE_credentials (optional)`: Either param_RS_integrationname or param_RS_AWSROLE_credentials must be provided.
- `-param_RS_AWSEXTID_credentials (optional)`: Used in conjunction with param_RS_AWSROLE_credentials, Any external ID that is the AWS IAM Role
- `—param_RS_integrationname (optional)`: Either param_RS_integrationname or param_RS_AWSROLE_credentials must be provided.
- `—param_AWS_S3outputchunksize` (optional): 1000 is the default if not specified
- `—param_RS_querysynchronous` (optional): FALSE is the default if you provide nothing; that is to say that an asynchronous query is the default and your script will return you a query_id immediately while the query is still running at Rockset in the background; but if you provide TRUE your query will run synchronously and block until success or failure
- `—param_RS_adv_filtercollection_byID` (optional): This is advanced feature and only accepts an integer of 1 or 2. If you have a really large dataset, you might need to filter the collection to export it by _id using one of the 16 hex values starting the GUID. If you select 1, then the output file will contain 16 queries that will run sequentially. If you select 2, then the output file will contain 256 queries (16^2) that will run sequentially. Keep this in mind if you are running against large datasets.

# Output

The script will create a file named with the generated CURL POST command and SQL statement.

When you run your program, it will export data from Rockset and write the results directly into S3.

If you run in synchronous mode (not the default), your console will wait until the script completes. However, it may timeout which is why asynchronous is the default. You will see output like this:
```JSON
{"query_id":"a63d329a-964e-45d3-a314-30ca0f43ed6a:kBotPiX:0","status":"QUEUED","stats":{"elapsed_time_ms":9,"throttled_time_micros":9000}}
```

Once it’s completed, the results in S3 will look like the following:

`Amazon S3 → Buckets → myS3bucket → prod.mycollection → a63d329a-964e-45d3-a314-30ca0f43ed6a_kBotPiX_0 → Documents`

The “a63d329a-964e-45d3-a314-30ca0f43ed6a_kBotPiX_0” is a randomly generated query ID GUID from Rockset that identifies your query with the exported results in S3.

# Code

[Python program](export_RSCollection_to_AWSS3.py)
