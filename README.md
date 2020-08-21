# spark-etl-on-aws-emr
This basic example uses data sources stored in S3. Using Spark SQL, it aggregates the different datasets and loads that data into DynamoDB as a full ETL process.

### Notes 

* ElasticMapReduce cluster will be created using AWS web console with Spark on Hadoop YARN and Zeppelin Notebook applications
* SSH tunnel will be established to access Zeppelin Notebook on cluster master node 
* Tables on HDFS will be created and Text formatted data located in S3 will be imported to tables
* Queries will be executed to generate reports and results will be stored on Dynamo DB
* Spark Cache feature will be used to speed up query execution
* After proof of concept resources will be terminated to avoid incurring costs. 

### Prerequisities 

* Follow this [tutorial](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html)
You should be able to create Administrator user smoohtly by following steps on above link using AWS web console.
* Create key pairs to access EC2 instances by following the this [tutorial](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#prepare-key-pair)
* SSH client should be available on your local host to enable SSH tunnel 

### 1. Create EMR cluster
Select Spark from Applications under Software configuration section. Select the EC2 key pair you created under Security and access section. Leave the rest with default values and create the cluster. Once the cluster reaches the RUNNING state, we can establish SSH tunnel to access the Zepplein Notebook as SQL web console

### 2. Enable SSH port 22
Review the Cluster details and find out security group for the master node. Add port 22 to inbound connection with from my ip option to secure access.

### 3. Enable SSH Tunnel
Please make sure that you downloaded EC2 key pair file beforehand. Locate the downloaded .pem file and run below command to create tunnel from `localhost:8157` to `ec2-13-59-236-49.us-east-2.compute.amazonaws.com:8890`
On master Zeppelin notebook is running on port 8890. EMR cluster security model prevents direct port access and below port forwarding with EC2 access key is the one of options. For other options [see](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-web-interfaces.html). The option we used here is Setting Up an SSH Tunnel to the Master Node Using Local Port Forwarding.
```
ssh -i asg-pairs.pem -N -L 8157:localhost:8890 hadoop@ec2-13-59-236-49.us-east-2.compute.amazonaws.com
```
### 4. Access Zeppelin Notebook Web UI
You should be able to access Zeppelin Notebook UI via URL http://localhost:8157. From that point SQL like scripts are provided to present use cases.

```sql
%sql CREATE EXTERNAL TABLE IF NOT EXISTS UserMovieRatings (
userId int,
movieId int,
rating int,
unixTimestamp bigint
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION 's3://us-east-1.elasticmapreduce.samples/sparksql/movielens/user-movie-ratings'
```

```sql
%sql CREATE EXTERNAL TABLE IF NOT EXISTS MovieDetails (
movieId int,
title string,
genres array<string>
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '#'
collection items terminated by '|'
STORED AS TEXTFILE
LOCATION 's3://us-east-1.elasticmapreduce.samples/sparksql/movielens/movie-details'
```

```sql
%sql SELECT title, count(*) numberOf5Ratings FROM usermovieratings r
JOIN moviedetails d ON (r.movieid = d.movieid)
WHERE rating = 5
GROUP BY title
ORDER BY numberOf5Ratings desc limit 5
```

```sql
%sql cache table usermovieratings
%sql cache table moviedetails
```
```sql
%sql SELECT title, count(*) numberOf5Ratings FROM usermovieratings r
JOIN moviedetails d ON (r.movieid = d.movieid)
WHERE rating = 5
GROUP BY title
ORDER BY numberOf5Ratings desc limit 5
```

```sql
%sql CREATE EXTERNAL TABLE IF NOT EXISTS UserDetails (
userId int,
age int,
gender CHAR(1),
occupation string,
zipCode String
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '|'
STORED AS TEXTFILE
LOCATION 's3://us-east-1.elasticmapreduce.samples/sparksql/movielens/user-details'
```

```sql
%sql SELECT * FROM (SELECT 'M' AS gender, title, count(*) numberOf5Ratings FROM usermovieratings r
JOIN moviedetails d ON (r.movieid = d.movieid)
JOIN userdetails u ON (r.userid = u.userid)
WHERE rating = 5
AND gender = 'M'
GROUP BY title, gender 
ORDER BY numberOf5Ratings desc limit 5) AS m
UNION
SELECT * FROM (SELECT 'F' AS gender, title, count(*) numberOf5Ratings FROM usermovieratings r
JOIN moviedetails d ON (r.movieid = d.movieid)
JOIN userdetails u ON (r.userid = u.userid)
WHERE rating = 5
AND gender = 'F'
GROUP BY title, gender 
ORDER BY numberOf5Ratings desc limit 5) AS f
ORDER BY gender desc, numberof5Ratings desc
```

