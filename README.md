# TrackMan Data Enigneer System Design Challenge

This is my solution for the [TrackMan Data Engineering System Design Challenge](http://designchallenge.trackmandata.com/). In this README file, I'll discuss the solution I came up with, which components I chose, and why I chose them.

![Full System Design](/images/01_Design_Full.jpg)

## The Challenge

The challenge states that it can be approached as a green field project, with only 3 components in place:

* A relation database called `trackman-backend`.
* An S3 bucket called `trackman-lake`.
* Another relational database called `dataengineering-db`, which serves as the backend for an internal application (this internal application can be ignored for the purpose of the challenge).

The solution needs to read data from multiple different tables in `trackman-backend` and ingest that data into both `dataengineering-db` and `trackman-lake` with a delay of no more than 30 and 60 minutes respectively.

The solution should also take into account that more data soruces might be added in the future.

## Solution Approach

Having worked with data pipelines before, I knew the main challenge was in figuring out which tools to use for the ETL process, and how to integrate it with the source and destination. So I spent a lot of time figuring out what tools AWS provides for handling _Data Ingestion_, _Monitoring_, _Cost Analysis_, _Data Analysis_, and _Data Warehousing_. Once I had a decent grasp of the main tools, I started putting the solution together, trying to account for future sources.

## Data Ingestion and Orchestration

![Data Ingestion and Orchestration](/images/02_Ingestion_and_Orchestration.jpg)

AWS offers a lot of tools for moving data between systems. For this solution, I decided to utilize _Glue_, _Database Migration Service_ (DMS), and _Kinesis_, each of which have different use cases. On top of that, I would use a combination of _Step Functions_ and _Lambda_ to create and manage workflows if they start to grow in complexity.

* Database Migration Service (DMS)

_DMS_ is a cloud service that allows for migrating data from one relational database to another. On top of one-time migration, _DMS_ can be set up to constantly sync changes from the source to the target, resulting in near-real-time updates to the target. For this solution, _DMS_ would be used for moving data from `trackman-backend` to `dataengineering-db`.

* Glue

_Glue_ is a serverless data integration service which can be used to create, run, and monitor ETL pipelines. It can connect to more than 70 data sources, making it a solid choice for general-purpose ETL jobs. Gl_ue also supports running custom Python scripts, so if a source is not supported, custom code can be written to handle it anyway. _Glue_ also has a _Job Bookmark_ functionality, which lets an ETL job persist state information between runs, allowing for a _Glue_ job to only load the newest data from the source. This is great for both cost efficiency (as the job runs for a shorter time) and storage (as data duplication is heavily reduced).

For this solution, _Glue_ would be used to move data from `trackman-backend` to `trackman-lake` with a job that runs every 20 minutes, loading only the newest data due to Job Bookmarks.

Due to its versatility, _Glue_ can also be utilized for future sources.

* Kinesis

_Kinesis_ is a a collection of tools that can collect, process, and analyze real-time data streams. It can handle a varity of data streams, and can output to, among others, an S3 bucket.

This challenge doesn't involve live-streaming data, but future sources might raise the need for the functionality, which is why I mention it.

* Step Function & Lambda

_Step Function_ is a serverless orchestration service, which can integrate with _Lambda_, _Glue_, and other AWS services to build complex processes and applications. It allows a developer to examine each individual state in the workflow to make sure everything is okay, and contains features for building workflows featuring failure handling, branching workflows, paralell processing, and human interaction if the need arises.

_Lambda_ is a compute service that can run code without the need for managing services, while taking care of overhead of administrating resources, handling system maintenance, etc.. It's useful for a variety of tasks, such as processing files or functioning as backends for applications.

_Step Function_ and _Lambda_ might not be necessary for the data flows this challenge focuses on, but once more sources are added and the data flows become more complex, they're valuable assets for managing the data pipeline.

## Monitoring

![Monitoring](/images/03_Monitoring.jpg)

When the data starts flowing, it is good to be able to monitor the services themselves and audit any change made to the systems. This is where _CloudWatch_ and _CloudTrail_ comes into the picture.

* CloudWatch

_CloudWatch_ monitors the AWS resources and applications that are running. It can collect and track metrics, which can be used to see how the services are doing. It can be used to trigger alarms if something is misbehaving, or make changes to resources based on user-defined rules. It basically provides all the monitoring that is necessary for making sure that the solution is running properly.

* CloudTrail

Where _CloudWatch_ monitors resources and services, _CloudTrail_ monitors everything users are doing to those resources and serviecs. Any action taken by a user is recorded as an event in CloudTrail. These events can be audited to provide full visibility in regards to security and best practices, something that is important for any solution, not just data pipelines.

## Cost Analysis

![Cost Analysis](/images/04_Cost_Analysis.jpg)

Another aspect of data pipelines is the cost of running them. While we will be paying for the services, it is good to know exactly what we're paying for. The solution uses _Cost Explorer_ and _Budgets_ to analyze and manage spending.

* Cost Explorer

_Cost Explorer_ is a tool that allows for viewing and analyzing costs and usage. It contains historical data up to 12 months back, and can make a forecast for the next 12 months. With _Cost Explorer_, one can identify exactly where the money goes and take proper action.

* Budgets

_Budgets_ allows for taking action based on the cost of the services running. Various types of budgets can be set up, with the option to raise warnings if a budget is about to be exceeded. _Budgets_ can discover anomalies in spending and usage patterns, a capability that is good for the solution, as we want to understand the causes of such things.

## Destinations

![Destinations](/images/05_Destinations.jpg)

The final part of the solution is where the data ends up. Given that `dataengineering-db` and `trackman-lake` are already set up, there's not much to be said on that topic. However, the challenge states that at some point, the ingested data should be made available to analysts in the company through a data warehouse. For this purpose, we'll use _Athena_ to analyze the data, then move it into _Redshift_ in a structured manner, where the analysts can then work with the data.

* Athena & Redshift

_Athena_ is a serverless interactive query service that makes it easy to analyze data directly in S3 using standard SQL. Considering that `trackman-lake` will contain a huge amount of more-or-less structured data, we want to use _Athena_ to identify and understand data that is relevant to the analysts, but that is all. _Athena_ should not be considered a data warehouse, as it only offers a view into the data lake.

_Redshift_, on the other hand, is a serverless, fully-managed, massive-scale data warehouse service. Resources are automatically provisioned and the capacity is automatically scaled in order to deliver fast performance. This is where the analysts will go to do their work.

My suggestion is to use _Athena_ to understand the data in the lake, then move it into _Redshift_ in a more organized fashion (this can be accomplished using _Glue_.). The analysts can then do all their work on the data in _Redshift_, not having to worry about understanding _everything_ the data lake contains.

## The Full Design

![Full System Design](/images/01_Design_Full.jpg)

Putting all of the above components together results in a data pipeline that can handle a large variety of data sources, with the ability to construct as complex workflows as is required for the data that is being handled. The solution has options for monitoring both the running services as well as any changes made to them, while also providing the ability to analyse cost and monitor spending. Finally, the data can be structured into a highly-scalable data warehouse, once it is time to expose the data to the company's analysts.