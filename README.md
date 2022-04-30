# Data Engineering project with Youtube analytics data and AWS tools

In this repository, I'll show how to build a simple data pipeline with AWS. 

I'll also discuss about:
- Building a data lake from scratch in Amazon S3, joining semi-structured and structured data
- Good practices for lake house architecture design
- Data lake design in layers, partitioned in raw, cleansed and analytics version
- AWS data catalogue
- ETL in AWS glue jobs
- SQL with Amazon Athena and optimization
- Incremental ingestion of data

As a future repository, I'll present some BI dashboards that may be built with the data from this project.

The ideia for this project was found surfing the web, props to many websites that post about it.


# The dataset

The dataset for this project may be found on Kaggle, [in this link](https://www.kaggle.com/datasets/datasnaek/youtube-new).

## Considerations before starting the project

- I advise you to create a secondary IAM user, not the root one. There are many points regarding the motivation to do so, the main one is because the root account have all the permissions and a secondary one will have way less. (I may create a new repository showing how to do so);
- Install AWS CLI;
- Set AWS CLI with your user credentials, created in the first point of this section.

# Creating the first bucket

After logging in the console with the link provided when you download your credentials, look for Amazon S3.

Create a new bucket. By default, we name it using some specific rules. Mine in this case was named "de-youtube-raw-useast1-dev" (as it is the raw bucket and developing environment. The region doesn't matter that much as it's a personal project). The only setting changed was regarding server-side encryption, which was turned on.

## Uploading the data in the bucket

As I'm a Windows user, I did it with the cmd. Just get to the path were the files are stored and copy the following code (the code is also in this repository).

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/DataUploadToS3.PNG?raw=true)

Basically, this code line copies all of the json files to the S3 bucket in the specified path (also creating two new folders). As this process is completed, you may refresh the bucket to see its content, as presented below:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/Bucket1.PNG?raw=true)

Then, the same process is done to copy the data files to its own location with Hive-style patterns. You may find all the commands in this repo, I'll leave one below just as an example:
    aws s3 cp CAvideos.csv s3://de-youtube-raw-useast1-dev/youtube/raw_statistics/region=ca/
    
# First interaction with AWS Glue console

Before interacting with Glue, open the IAM page and create a new user role. Mine was "de-youtube-glue-s3-role". Its permissions should be:
- AmazonS3FullAccess (for general purposes of this project)
- AWSGlueRoleServiceRole

Open Glue console and look for "Crawlers" in the left menu. Click on add crawler.

- Name: "de-youtube-raw-glue-catalog-1"
- Leave the other settings as default an click on next two times
- Specify the S3 data path as: "s3://de-youtube-raw-useast1-dev/youtube/raw_statistics_reference_data/"
- Click on "next" two times once again
- Choose an existing IAM Role, the one created in this session (de-youtube-glue-s3-role)
- Run the crawler on demand
- Add a new database (mine was "de-youtube-raw")
- Finish the creation with the remaining default options
- Run the crawler


A table will be created as the product of the crawler execution. As we analyze the data, it's possible to see that the created table has three columns, as the normal structure of a json file. The issue with it is that this format of table won't permit us to query anything in Athena (SQL service).

## Getting the first Athena error

To see what I've just mentioned, we have to specify an output location for Athena. Clicking on "View settings" and "Manage" we may indicate the location of query results. For this reason, I've also created another bucket, named "de-youtube-raw-useast1-athena-job".

Once set, run Athena's default query. The error should be just like the image below:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/AthenaError1.PNG?raw=true)

The reason for this is because of the "kind" and "etag" keys, which are in the json file. We just need the "items" key and its values. For this reason, we need to preprocess and extract this particular item of the file and store it somewhere else.

Amazon Athena libraries also have an example of how the JSON data files should be to be read. Basically, AWS Glue needs it all in a single line.

# Data Cleansing

## Testing a prototype ETL, transforming JSON to Parquet (Apache) with AWS Lambda

The purpose of this section is to transform the JSON file into a row and column format, just like Apache's one.

Before going to Lambda, create a new role as "de-youtube-raw-useast1-lambda-role". Set Use Case as Lambda. Permissions should be AmazonS3FullAccess and AWSGlueServiceRole.

Open AWS Lambda and create a new function from scratch:
- I've named it "teste"
- Runtime: python 3.9
- Use the role you've just created
- Keep the other settings as default
- Create your function

Now we need to write a function code which will get only the content we want from the JSON file. For that, we'll use AWS Wrangler, which is an Amazon library that basically reads file stored in S3.

The code may be found in this repository, the original code was gathered around the internet, I've just adapted it. As a good practice and AWS feature, we won't be naming the OS inputs in the file.

On Configuration and Environment variables, add the following:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/EnvironmentVariables.PNG?raw=true)

Before running the code, remember to create the database and table on Glue and the bucket in S3 with the respective names as in the image above. Regarding the operation of writing data, we need to choose append as we don't want to overwrite it.

To test it, there's the need to create a test event. I've used s3-put template. Changes that need to be made:
- I replaced "example-bucket" with "de-youtube-raw-useast1-dev"
- Also replaced "test%2Fkey" with "youtube/raw_statistics_reference_data/US_category_id.json" (as it's just a test, any of the files would work)


Some errors I got on this part:

**Unable to import module "lambda_function"** <br>
Rolled the page down and clicked on "Add layer". <br>
Added "AWSDataWrangler-Python39", version 5. With this, the function will start supporting Wrangler, which is called in the beginning of the code.

**Timeout error** <br>
Increased the timeout duration to 5 minutes, in the following path: Configuration -> General Configuration.

Once everything was correct, Lambda returns the following:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/LambdaTest.PNG?raw=true)

## Going back to Athena

Now we can go to Tables on AWS Glue and find the "cleaned_statistics_reference_data" table filled. On "Actions", it's possible to view data directly on Athena.<br>
Running the Query return something like:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/FirstQueryResults.PNG?raw=true)


# Creating new crawler 

Moving back to Glue, I created a new crawler:
- Name: "de-youtube-raw-csv-crawler-01"
- Path: s3://de-youtube-raw-useast1-dev/youtube/raw_statistics/
- Role: the Glue one
- Database: de-youtube-raw
- All other settings as default

Ran the crawler, which created a new table to be visualized in Athena. The data in this table is partitioned by region and that's the main reason of the "folders" created in the S3 bucket.

Also, now is possible to JOIN both the cleaned database and the raw statistics. However, id column in the cleaned_statistics_reference_data is read as a string. There are many ways to change it and materialize the INNER JOIN between category_id and id as the use of the cast function, where id would be casted as an integer.<br>
As a concrete solution, as the right way to deal with it is a preprocess, I went to Glue and changed the schema, by transforming the id column to bigint.

<span style="color:red">**A new error**</span><br>
As I used the INNER JOIN query between both tables, the following error message appeared:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/HiveBadDataError.PNG?raw=true)

The reason behind it is that Apache Parquet files data don't change, besides the change I made in the Glue Catalog.

I went to the cleansed bucket and deleted the parquet file on youtube folder. Then ran the Lambda function again, so it would recreate the file in the bucket.

The INNER JOIN now works (I present my query below):

    SELECT * FROM "de-youtube-raw"."raw_statistics" a
    INNER JOIN "db_youtube_cleaned"."cleaned_statistics_reference_data" b on a.category_id=b.id
    

Now there's the need to process the raw version that is stored into the raw bucket to move all the transformed data into one single bucket.

*Glue ETL Job

Glue ETL Jobs may be used to automatically do the process mentioned above.

Job properties:
- name: de-youtube-cleansed-useast1-csv-to-parquet
- role: glue one
- glue version: 2.0 or 3.0
- Enabled job bookmark (to see what is happening inside the job)
- Enabled Job metrics to see what is happening with the job on CloudWatch
- Other parameters as default
- Data source: raw_statistics (as this is the data we want to process)
- Change schema
- Create data target (Amazon S3 - Parquet - inserted the target path as the cleansed bucket with raw_statistics folder)

Once the Output Schema definition screen appeared, I matched all of the bigint on Source with the Target, as the default was string.

Saved the job and AWS opened the pyspark script. As the default script saves it in one file, I had to customize it to be partitioned. For this step, I used some info I found in the internet (the whole code is in the repository):

- imported DynamicFrame from awsglue.dynamicframe (it works like pandas, but more efficient for this job)
- inserted the datasink1 code line to add it into the dinamic data frame
- built the final_output
- added the partition key, which is region, to the path, on datasink4 line

Once I ran the job, the following errors appeared:

**Language errors**
There are some characters of Japanese and Russian language, which are not encoded as UTF-8. For the sake of this project, I've filtered out the data.<br>
First I've added a predicate_pushdown to filter only Canada, Great Britain and USA files on datasource. I'm not ignoring the data itself, but firstly I need to know if the job works.

After I suceeded, I went to create a new crawler, named "de-youtube-cleaned-csv-to-parquet-etl", with path "s3://de-youtube-cleansed-useast1-dev/youtube/raw_statistics/", and database "db_youtube_cleaned". I've also ran the job to see if it was working indeed.


## Adding a trigger to the Lambda function

The main purpose of adding a trigger to the Lambda function is to make it run automatically, everytime there's data ingestion. By the time I got to this point, I had only tested the function with the s3-put test event.

In summary, I wanted to execute all the json files in the raw_statistics_reference_data and whenever a file is uploaded, AWS environment should process all of them by itself. As an example, if I added a new region to the files, it should automatically be prompted in there.

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/Trigger.PNG?raw=true)

On Lambda screen, I've added the trigger with the following settings:
- bucket: de-youtube-raw-useast1-dev
- event type: all object create events
- path: youtube/raw_statistics_reference_data/
- suffix: .json

To test this trigger, I had to delete all of the contents of the raw bucket and also the output file of the cleaned version (de-youtube-cleansed-useast1-dev / parquet file) and re-upload just as I did in the beginning of the project.

Everything went ok, there wasn't any kind of error.

# Building a reporting layer (analytics bucket)

In this step, I built a reporting layer with some ETL from the Cleansed bucket. This is generally a step that is done everyday in data pipelines.

As the data gets larger everytime, preprocessing it is really needed because the costs to query the data everytime become exponencially bigger. Once again, I used Glue to create the new ETL pipeline.

## Creating a Glue Job

Once in Glue Studio, I've opened the jobs page to create a new one, with the following settings:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/GlueJob.PNG?raw=true)

For this step, I created a new bucket, called "de-youtube-analytics-useast1-dev".

I've also created a new database in Athena, with the following query command:

        CREATE DATABASE db_youtube_analytics;

Back to the image of this section, the settings of the job are:
- 1st table is raw_statistics from db_youtube_cleaned
- 2nd table is cleaned_statistics_reference_data from db_youtube_cleaned
- Join type is INNER JOIN
- Condition is category_id = id
- Data target is parquet format, snappy compression and the analytics bucket
- "Create a table in the Data Catalog..."
- Table name: final_analytics
- Partition keys: region and category_id (as both are being used to analyze the data, category_id will have its own folders in the destiny bucket)

Once I set the role and ran the job, everything went ok. The final ETL pipeline is completed.

## Final considerations

Basically, the steps for this project are presented on one of AWS documentation pictures, that may be seen below:

![alt text](https://github.com/jack3DX/Data-Engineering-Youtube_analytics-AWS/blob/main/images/AWSDataFlow.PNG?raw=true)

I just didn't use Redshift as it wasn't necessary.

After all, it was a great project, provided some good experience and made me flow even deeper into AWS applications. I myself find AWS easier and simpler than GCP platform.

As a future repo, I intend to present the analytics bucket data on some of the AWS apps for Data Visualization.

Feel free to contact me in case you have any questions.
