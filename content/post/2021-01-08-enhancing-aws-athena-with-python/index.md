---
title: Enhancing AWS Athena with Python
author: Dan
date: '2021-01-08'
slug: []
categories: []
tags:
  - AWS
  - cloud computing
  - tips
description: ''
lastmod: ''
image: ''
author_twitter: ''
---

AWS Athena allows you to run SQL queries against a data lake on an ad-hoc basis. Many analysts begin using Athena in the workbench in AWS console. This setup is ideal for iterating as you explore your data and refine your queries. But this setup is not ideal if you need to run queries on a regular basis or if you would like to run a similar query but change certain parameters, such as which data you analyze or which variables are extracted. These scenarios are better suited for using Athena in Python or another scripting language that allows you to dynamically update parameters before executing your queries. Here is a basic tutorial on how to migrate from Athena on the AWS console to Athena in Python.

This tutorial assumes you have access to an AWS account and you have permissions to use Athena. Please ensure you have [installed](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) the AWS Command Line Interface (CLI), downloaded access keys from the IAM section of AWS, and configured the AWS CLI to use the access keys. For a walkthrough on these steps, see [this article](https://amolkokje.medium.com/aws-command-line-interface-8bc6bb1c8775). 

## Create a table with sample data to analyze
Follow [these instructions](https://docs.aws.amazon.com/athena/latest/ug/getting-started.html) to set up a table with sample data to use in this tutorial.  

The query at the bottom of the instructions should work for you before you proceed. We will be using this query to understand how Python can simplify and augment your work. The query is:
```
SELECT os, COUNT(*) count
FROM cloudfront_logs
WHERE date BETWEEN date '2014-07-05' AND date '2014-08-05'
GROUP BY os;
```

## Connecting to Athena in Python

Using Athena in the AWS Console is one way of connecting to AWS: 
1. login to the AWS website (a process known as authentication)
2. navigate the Athena service
3. execute a SQL query
4. receive the results

The AWS CLI is another way to call on AWS. The access keys you downloaded and configured serve to authenticate your computer with AWS in the same way you do when you login. AWS has created packages of code that allow you to access AWS from your local computer in a variety of languages; these packages are known as Software Development Kits (SDKs) because they assist you in developing software that uses AWS. THE AWS SDK for Python is called [boto3](https://aws.amazon.com/sdk-for-python/). Boto3 allows you to complete the steps outlined above all within Python.

To install boto3, run the following command from your command line or terminal: 
```pip install boto3```

Running this command will install the code necessary for your computer to tell AWS how to execute your Python code. 

Let's walk through how we complete each of steps outlined above with Python.

## 1. Login to AWS

This step is automatically done when you configure the AWS CLI, as you did earlier. By running the `aws configure` command and entering your access key information, your computer will know where to look when it wants to communicate with AWS. 

## 2. Navigate to the Athena service

To access AWS Athena, you need to tell your computer that you want to talk to Athena, rather than any other AWS service. Boto3 allows you to specify which service you would like to use through a client. To do this for Athena, you run the following code:

```
import boto3

athena_client = boto3.client('athena')
```
The `athena_client` object knows how to use AWS Athena in the same way you do when you execute queries in the console. This client comes with a number of functions that make it easy for you to do common Athena tasks, such as starting a query or listing the tables in a database. To use one of these functions, you call the function from the client and add parameters that specify the information would like to retrieve. For example, to list the databases you have access to in your default Athena Data Catalog (called 'AwsDataCatalog'), you would run the code:

```
athena_client.list_databases(CatalogName='AwsDataCatalog')
```

Here, `athena_client` is executing the `list_databases` function to list the databases available in the 'AWSDataCatalog' Data Source. This command is the same as if you navigated to Athena in the AWS console, selected 'AWSDataCatalog' from the Data Source dropdown, and looked at the databases listed in the Database dropdown menu. You can find a full list of the functions available for AWS Athena in the [documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/athena.html#Athena.Client.list_databases).

## 3. execute a SQL query

Now let's try to execute the same Athena query from before in Python. If you were in the AWS console, you would simply enter your query in the workbench and press "run_query". You only need piece of information to do this, the query you want executed. In programming, the query is known as a "parameter" because changing this value will change what the client asks Athena to do for you.

To do this in Python, we need to identify the right client function, which is called `start_query_execution`. This function requires just one parameter, `QueryString`, which is the query you would like to execute. 

Let's create a query string in python and tell the Athena client to execute the query for us:
```
query_string = """
SELECT os, COUNT(*) count
FROM cloudfront_logs
WHERE date BETWEEN date '2014-07-05' AND date '2014-08-05'
GROUP BY os;
"""

response = athena_client.start_query_execution(QueryString=query_string)

query_execution_id = response['QueryExecutionId`]
print(query_execution_id)

```

## 4. Retrieve the results

One of the advantages of working in the console is that your results are printed right on the screen for you to inspect. This is great for data exploration and query development, when you are not quite sure that the query meets your needs yet. Retrieving the results is not as simple in Python.

Each time you query Athena, AWS attaches a unique identifier to your query so that they can connect your query to the results it produces. The `response` object in the code snippet above includes this unique identifier as part of a dictionary. The unique identifier is identified by the key `QueryExecutionId`. Every time you execute an Athena query, it saves the results in AWS Simple Storage Service (s3). If you are working in the Athena Console, you will see the results right on screen, but remember that AWS also stores the results in s3 so you can retrieve them later.

To retrieve your results when using Python, you will need to know where in s3 AWS saves your results. To figure this out, you can go to the Athena Console, click on "Settings" and look for the "Query result location" in the popup. If you navigate to this location in s3, you will find a CSV file with the file name matching your query execution ID. 

However, we want to do all of this in Python, so let's use a client for s3 to retrieve the results. Here, we create variables for the s3 location where Athena saves the results (`query_result_location`) and we create the full path to the csv file for this query by adding the `query_execution_id` to `query_result_location` and adding the '.csv' file extension. We then create a new client specifically for s3, which includes all of the functions you need to interact with s3. 

To create the parameters needed to get the csv in s3, we split up the `query_result_file_name` into the Bucket and the Key (a Bucket is like a folder on your computer, while a Key is like the specific file in a folder). With these two new variables (`bucket` and `key`), we can use the s3 client function `get_object` to retrieve the csv file in s3 and read it into python as a Pandas Data Frame. Running this code will output the Data Frame containing your query results. 

```
import io
import pandas as pd

query_result_file_name = query_result_location + query_execution_id + '.csv'
  
bucket = query_result_file_name.replace('s3://').split('/', 1)[0]
key = query_result_file_name.replace('s3://').split('/', 1)[-1]
  
obj = s3.get_object(Bucket=bucket, Key=s3_file_key)
result_df = pd.read_csv(io.BytesIO(obj['Body'].read()))
```

Because Athena queries take time to run, we need to add one more function that will force our Python code wait until the query is completed before it retrieves the results. Without this function, you might get your results because your Python code might look for a result file before the query has finished executing.
```
def wait_for_query_completion(query_execution_id: str):
  """
  determines if the query has finished excuting. If it has not, the function waits for the query to finished.
  Parameters:
    query_execution_id: the query execution id (as a string)
  """
  while (max_execution > 0 and state in ['RUNNING', 'QUEUED']):
          max_execution = max_execution - 1
          response = client.get_query_execution(QueryExecutionId = execution_id)
  
          if 'QueryExecution' in response and \
                  'Status' in response['QueryExecution'] and \
                  'State' in response['QueryExecution']['Status']:
              state = response['QueryExecution']['Status']['State']
              if state == 'FAILED':
                  return False
              elif state == 'SUCCEEDED':
                  s3_path = response['QueryExecution']['ResultConfiguration']['OutputLocation']
                  filename = re.findall('.*\/(.*)', s3_path)[0]
                  return filename
          time.sleep(1)
  return False
```

Let's put everything we have done into a single function:
```
import boto3
import io
import pandas as pd

def run_query(query_string: str, query_result_location: str):
  """
  Executes an Athena query and returns the results from s3.
  
  Parameters:
    query_string: the query to be executed (as a string)
    query_result_location: the s3 location where Athena saves the results (as a string)
  """
  athena_client = boto3.client('athena')
  s3_client = boto3.client('s3')

  response = athena_client.start_query_execution(QueryString=query_string)
  query_execution_id = response['QueryExecutionId`]
  
  wait_for_query_completion(query_execution_id)
  
  query_result_file_name = query_result_location + query_execution_id + '.csv'
  
  bucket = query_result_file_name.replace('s3://').split('/', 1)[0]
  key = query_result_file_name.replace('s3://').split('/', 1)[-1]
  
  obj = s3.get_object(Bucket=bucket, Key=s3_file_key)
  result_df = pd.read_csv(io.BytesIO(obj['Body'].read()))
  
  return result_df

### example using the function
query_string = """
SELECT os, COUNT(*) count
FROM cloudfront_logs
WHERE date BETWEEN date '2014-07-05' AND date '2014-08-05'
GROUP BY os;
"""
query_result_location = 's3://EXAMPLE-BUCKET/EXAMPLE-PATH/'

query_result_df = run_query(query_string, query_result_location)
print(query_result_df)

```

## Leveraging Python For Repetitive Tasks
This seems like a lot more work than just using Athena in the Console; however now that we know how to complete all of the steps in Python, we can see how Python can help speed up our work. 

The original query filtered data between two dates in the `WHERE` statement:
```
SELECT os, COUNT(*) count
FROM cloudfront_logs
WHERE date BETWEEN date '2014-07-05' AND date '2014-08-05'
GROUP BY os;
```
What if we want to run the query every week for data in the previous week? In the Console, we could come back to the Console every week, change the dates in the query, and run the query. However, if we know we are going to want to do this task repetitively, we can use Python to change the values every time the script is run, this frees up our time and reduces the potential for errors when updating the query:

```
from datetime import datetime, timedelta

query_string_template = """
SELECT os, COUNT(*) count
FROM cloudfront_logs
WHERE date BETWEEN date {} AND date {}
GROUP BY os;"""


today = datetime.today().strftime('%Y-%m-%d')
last_week = (datetime.today() - timedelta(days=7)).strftime('%Y-%m-%d')

new_query_string = query_string_template.format(last_week, today)
print(new_query_string)

query_result_df = run_query(query_string, query_result_location)
print(query_result_df)

```

You will notice the variable `query_string_template`, the date values have been replaced with `{}`. These curly brackets allow us to insert values into the string later on using the `format()` function. By creating variables for today and last week using the Python `datetime` package, we then generate a new query string with these values (`new_query_string`). We then call our `run_query` function using the new query string, and we get new results based on the date variables we inserted into the query.

Next week, when we want to run this query again using new dates, we don't have to update the code because Python is computing the date values at the time that you execute the Python script. 

If you wanted to run the same query against different tables, you could create a list of table names and loop through them to run a query for each one:

```
query_string_template = """
SELECT *
FROM {}
"""
tables = ['table_a', 'table_b']

for table in tables:
  new_query_string = query_string_template.format(table)
  result_df = run_query(new_query_string, query_result_location)
  print(df)
```

In this example, we have replaced the table name in `query_string_template` with `{}`, so when we run `query_string_template.format()`, it will insert a value where the curly brackets are to produce a new query string. In the for-loop above, we create a new query string in each iteration of the loop (via the `format()` function) and then use the new query string as a parameter in an Athena query, and print the resulting Data Frame. 

This method of creating template strings and inserting values when you run a Python script can be expanded to more complex queries. Perhaps you would like to aggregate data for multiple tables but each table has a different name for the same variable, you could write a template query string that inserts different variables in a `GROUP BY` statement depending on the table name. 