In a previous repo I outlined a way of converting text files to compressed columnar formats like Parquet using Spark and 
Scala. The rationale behind doing this is was to leverage the large increase in querying speeds that a columnar file format 
allows.

I showed in another of my repos how summing a numeric column of a 350 million record file using AWS’s Athena took 20 seconds 
for a regular text file compared to 1 second for the same file in Parquet format.

Now, I called my conversion method “easy” and it’s true that it is easy if you have coding experience, but what if I told you 
that - using one of AWS’s newer tools - you can do the exact same text to parquet format conversion without having to do any 
coding at all. That is what AWS’s Glue product will let you do. Glue is a data categorisation and ETL product. It’s fully managed 
and scalable and you only pay for what processing you use. What follows is a step-by-step process that shows how you can convert 
a text file to a parquet file format without having to do any coding.

There are a few pre-requisites required before you start. First of all I would expect you have at least a passing knowledge of 
common AWS services and terms. So, I would expect for instance that you know what the S3 service is and what a bucket is. Also
that you’re aware of what IAM and IAM roles are. Secondly, you need access to AWS so I would expect that you have something like 
a free tier account already good to go. If you don’t please sign up for one as it’s a great way to start learning about AWS. You 
also need to have an input and output S3 bucket set up with a delimited text file present in your input bucket. This will be the 
file you will convert and you’ll store the converted output file in your output bucket.



### Set up an IAM role to allow Glue read/write from your S3 buckets

Sign in to the AWS Management Console and open the IAM console at https://console.aws.amazon.com/iam/.

In the left navigation pane, choose Roles.

Choose Create role.

For role type, choose AWS Service, find and choose Glue, and choose Next: Permissions. On the Attach permissions policy page, 
choose the policies that contain the required permissions; for example, the AWS managed policy AWSGlueServiceRole for 
general AWS Glue permissions and the AWS managed policy AmazonS3FullAccess for access to Amazon S3 resources. The 
following is an example of the type of policy you'll need:
```
{

   "Version": "2012-10-17",

      "Statement": [

                      {

                         "Effect": "Allow",

                         "Action": [

                            "s3:GetObject",

                            "s3:PutObject"

                                   ],

                          "Resource": [                                                                                                

                             "arn:aws:s3:::your-bucket-names-here*"

                                      ]

                       }

                   ]
}
```

For Role name, type a name for your role; for example, AWSGlueServiceRoleDefault. Create the role with the name prefixed 
with the string AWSGlueServiceRoleto allow the role to be passed from console users to the service. AWS Glue provided 
policies expect IAM service roles to begin with AWSGlueServiceRole. Otherwise, you must add a policy to allow your users 
the iam:PassRole permission for IAM roles to match your naming convention. Choose Create Role.


# Crawl your input text file.

Glue uses what it calls a crawler process to look at your data, infer a schema from it and create hive compatible tables 
for it. These tables then become available to other AWS services and tools such as Athena, EMR/Spark and the like to enable
them to do fast SQL like operations on the data.

Click on the Crawler link on the left hand pane then click on the Add Crawler button.

Enter a name for your crawler and choose the Next button.

Choose S3 for your data source and the “Specified path in my account” radio button

Click on the folder icon to the right of the Include Path field and select your input folder in the pop-up window.

Click on Next, then Next again.

Check the “Choose an existing IAM role” radio button and enter the name of the IAM role you previously created.

Click on Next, then Next again.

Choose default in the Database to contain Tables field and click on Next, then click the Finish button.

Click the Run It Now link at the “Crawler crawl-text was created to run on demand. Run it now?” message at the top of the screen.


After a minute or so you should see that the jobs has finished and created one table. Click on the Tables link at the left 
hand side of the screen to view your tables. Click on the newly created table and you should see some information about it 
such as the column names and data types and so on. At this stage you can go right ahead and start running ANSI SQL statements 
against your text file using Athena, but for now let’s crack on and run our file conversion.

NB You should note that a Glue Table does not contain any actual data. It’s just a definition of the schema of your data 
files on S3. But the the beauty is that other AWS tools such as Athena can take advantage of this schema to do super-fast
data extractions on your S3 data.

# Set up a file conversion job on AWS Glue

Click on Jobs link on the left-hand pane of the Glue home screen

Click on Add Jobs button. Enter convert-text-file-to-parquet in the job name field and select your previously created 
IAM role in the IAM role field. 

Click on the “A proposed script generated by AWS Glue” radio button and type in whatever name you want for the script that 
AWS will create. If you want you can choose your preferred ETL language for the script if you have one. Since we’re assuming 
zero code knowledge here just leave it at its default value of Python and give your script a name.

Leave everything else as is and click on the Next button at the bottom of the screen.

Choose your data source. This will be the name of the Glue table that your crawler process created. Click on Next.

Click the “Create tables in your data target” radio button.

Select S3 as the data store, format is Parquet. Click on the folder icon next to the Target Path field and choose 
your output S3 bucket. Click on the Next button.

At this stage you will be presented with a screen that allows you to map input fields to output, delete fields and 
change field data types, but for now just click the Next button.

Click on the Save Job and edit Script button.

At this stage if you were savvy at coding you could tweak your script but as we're assuming zero coding skills you just 
want to click on Run Job near the top of the screen. A popup window will appear giving you the option to change certain 
parameters but we can just click the Run Job button.

When the job has finished check your output S3 bucket. If all has went well you will see one or more parquet files there. 
If your input file was large it’s likely that your output bucket will contain a lot of parquet files. That’s not an issue 
though as your next likely step will be to run another Glue crawler process on your parquet output files and create a new 
Glue database table describing your Parquet data. Once that’s done any SQL you can run using Athena for instance will treat
your multiple input parquet files as if there was only one large contiguous Parquet file.

Although I've only described the process for running a crawler and ETL job for one input file you should be aware that Glue will 
scale to handle 1000's of input files. I think you’ll be pleased with the large speed increase on data extractions on your 
Parquet data file(s) as compared with the same operations on your text file(s).
