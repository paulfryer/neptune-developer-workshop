# Airports and Flights

Getting Started with Amazon Neptune Hands-on Lab

## Create and Configure Cloud9 Instance

1. Create a new Cloud9 Development Environment
  * See documentation at https://docs.aws.amazon.com/cloud9/latest/user-guide/create-environment.html
  * Choose EC2 for the Environment Type
  * Choose t2.small for the Instance Type
  * Under “Networking Settings” ensure you choose the same VPC that you will be launching Neptune in. Neptune can only be accessed via VPC so you need to launch the Cloud9 environment in the same VPC.
  * Once the instance is created, click “Open IDE”
  * Open a new terminal via Window > New Terminal (Alt + T)

2. Install java
```
sudo yum install java-1.8.0-devel
sudo /usr/sbin/alternatives –-config java
```
  * When promopted, type the number for Java 8 (2)

4. Install Gremlin client
```
wget http://apache.claz.org/tinkerpop/3.3.3/apache-tinkerpop-gremlin-console-3.3.3-bin.zip 
unzip apache-tinkerpop-gremlin-console-3.3.3-bin.zip 
cd apache-tinkerpop-gremlin-console-3.3.3/bin
```
5. Open the file: apache-tinkerpop-gremlin-console-3.3.3/bin/gremlin.sh and update the $JAVA_HOME value to point to your Java8 location, at line 69. Save your change.
```
JAVA="/usr/lib/jvm/jre-1.8.0-openjdk.x86_64/bin/java -server"
```
6. Run the gremlin.sh script.
```
./gremlin.sh
```
You should see a gremlin!
         \,,,/
         (o o)
-----oOOo-(3)-oOOo-----
Exit with CTRL-Z or :exit

## Create Neptune Instance

7.	Create an Amazon Neptune db.r4.large instance
  *	See documentation at	https://docs.aws.amazon.com/neptune/latest/userguide/get-started-CreateInstance-Console.html
  * Choose “No” high availability
  * Choose a DB instance identifier
  * Leave all Advanced Configuration options on their default settings and click to Launch DB instance
  * Once the instance is created, find your Neptune endpoint name
8. Update Security Group to allow inner VPC traffic.
  * Add a custom TCP Rule that allows port 8182 from the CIDR block of the VPC your Neptune instance is running in. This will allow any traffic from within your VPC (Cloud9) to talk to Neptune.
  * Example:
    *	Custom TCP Rule
    * Protocol = TCP
    * Port Range = 8182
    * Source = 172.31.0.0/16 (or whatever your VPC CIDR is)

9.	Configure Gremlin to access Neptune
  * Open your Cloud9 instance.
  * Go to the Gremlin conf directory
```
cd ~/apache-tinkerpop-gremlin-console-3.3.3/conf
```
  * Create a file named neptune-remote.yaml that contains:
```
hosts: [your-neptune-endpoint]
port: 8182
serializer: { className: org.apache.tinkerpop.gremlin.driver.ser.GryoMessageSerializerV3d0, config: { serializeResultToString: true }}
```
  * Of course, you should replace your-neptune-endpoint with your Neptune endpoint. Do leave the [square brackets] in the file!
10. Run Gremlin and connect to your endpoint
```
cd apache-tinkerpop-gremlin-console-3.3.3/bin
./gremlin.sh
```
11. Connect to remote Neptune istance.
```
gremlin> :remote connect tinkerpop.server conf/neptune-remote.yaml
gremlin> :remote console
```
12.	Load Data into Neptune
  * Configure IAM to allow access
  * Follow documentation at https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load-tutorial-IAM.html
  * Create an IAM role, add the role to your Neptune cluster, and create an S3 VPC endpoint
  * Prepare and run your load command, follow documentation at https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load-data.html 
```
curl -X POST \
    -H 'Content-Type: application/json' \
    http://your-neptune-endpoint:8182/loader -d '
    { 
      "source" : "s3://bucket-name/object-key-name", 
      "format" : "csv",  
      "iamRoleArn" : "arn:aws:iam::account-id:role/role-name", 
      "region" : "region", 
      "failOnError" : "FALSE"
    }'
```
Example:
```
curl -X POST -H 'Content-Type: application/json' \
    http://darin-neptune-cluster.cluster-cxpjiluhh0c9.us-west-2.neptune.amazonaws.com:8182/loader -d '
    { 
      "source" : "s3://db-week-west/air-routes-latest-nodes.csv", 
      "format" : "csv",  
      "iamRoleArn" : "arn:aws:iam::011592212233:role/NeptuneLoadFromS3", 
      "region" : "us-west-2", 
      "failOnError" : "FALSE"
}'

curl -X POST -H 'Content-Type: application/json' \
    http://darin-neptune-cluster.cluster-cxpjiluhh0c9.us-west-2.neptune.amazonaws.com:8182/loader -d '
    { 
      "source" : "s3://db-week-west/air-routes-latest-edges.csv", 
      "format" : "csv",  
      "iamRoleArn" : "arn:aws:iam::011592212233:role/NeptuneLoadFromS3", 
      "region" : "us-west-2", 
      "failOnError" : "FALSE"
}'
```

Run this command twice, once for the source file air-routes-latest-nodes.csv and again for the source file air-routes-latest-edges.csv

If you are using a Neptune instance in us-east-1 (N. Virginia), use the s3://db-week/ bucket. If you are using a Neptune instance in us-west-2 (Oregon), use the s3://db-week-west/ bucket.

13.	Run Gremlin Transversals
  * Run Gremlin client
```
$ cd apache-tinkerpop-gremlin-console-3.3.3/bin
$ ./gremlin.sh
```
14. Connect to remote Neptune instance.
```
gremlin> :remote connect tinkerpop.server conf/neptune-remote.yaml
gremlin> :remote console
```
15. Try some interesting traversals:
Routes between Cleveland and Shanghai with two stops
```
gremlin> g.V().has('code','CLE').out().out().out().has('code','SHA').path().by('code')
```
What information do I have on San Francisco airport?
```
g.V().has('airport','code','SFO').values()
```
How many airports are there in the graph?
```
g.V().hasLabel('airport').count()
```
How many routes are there in the graph?
```
g.V().hasLabel('airport').outE('route').count()
```
Where can I fly nonstop from Cleveland?
```
g.V().has('airport','code','CVG').out().values('code').fold()
```
Flights from London Heathrow (LHR) to airports in Canada
```
g.V().has('code','LHR').out('route').has('country','CA').values('code')
```
Find lots more at http://kelvinlawrence.net/book/Gremlin-Graph-Guide.html
