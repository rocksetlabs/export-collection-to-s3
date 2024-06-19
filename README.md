# export-collection-to-s3
This is a text generation script. The generated text contains a CURL POST command to execute against Rockset’s endpoint API. The endpoint API used is the query endpoint because the CURL command will be issuing a SQL statement to Rockset. The SQL query will be a variation of INSERT INTO S3 based on the input paramters provided at run time.
