# KeywordAnalysis
Word analysis, by domain, on the Common Crawl data set for the purpose of finding industry trends

***
## Process
### Specific Domain Data Capturing
#### Common Crawl IBM data capturing
***Note that remote_copy project does not work now due to dataset path deprecated***
1. Start one node AWS EMR spark cluster
2. SSH to the instance: ec2-54-196-129-41.compute-1.amazonaws.com (change)   user: hadoop 
3. sudo yum -y install git
4. git clone https://github.com/trivio/common_crawl_index
5. export AWS_ACCESS_KEY=(your access key)
6. export AWS_SECRET_KEY=(your secret key)
7. 
```
sudo vi /usr/local/lib/python2.7/site-packages/boto/__init__.py
```
Insert host='s3.amazonaws.com' to below lines 
```
def connect_s3(aws_access_key_id=None, aws_secret_access_key=None, host='s3.amazonaws.com', **kwargs)
return S3Connection(aws_access_key_id, aws_secret_access_key, host='s3.amazonaws.com', **kwargs)
```
8. 
```
cd /common_crawl_index/
./remote_copy check "com.ibm.www"
``` 
```
files: 26045
webpages: 77768
Source compressed file size (MB): 2604500
Destination compressed file size (MB): 3197
```
[hadoop@ip-10-43-215-181 bin]$ ./remote_copy check "com.netapp.www"
```
files: 3363
webpages: 5381
Source compressed file size (MB): 336300
Destination compressed file size (MB): 68
```
9. 
```
./remote_copy copy "com.ibm.www" --bucket (your s3 bucket name) –key common_crawl/(your s3 bucket path) --parallel 4
```

### Remove html tags
I have run “dkpro-c4corpus” boilerplate removal code for three days with about 10 r4.4xlarge EMC instances (16vCPU, 122Gb Memory), because it works for me only on small data (20MB-100MB). And sometimes I got error “Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded”, even I have already set instance memory as huge, maybe some setting or code issue to look at later. So I have to split IBM.com data to 84 files, each file data processing (boilerplate removal) consumed 0.5~2 hours. Heavy manual work (split, recurring processing, join etc.) were done. Finally, I got plain text (s3://CommonCrawl/ibm_boiler) of IBM.com (data size decreased from 7GB with html tags to 1GB plain text). And ran spark word count for the IBM.com plain text and got the top60000 (attached) and word count results (s3://CommonCrawl/wordcount-output/wordcount-ibm_bolier).

1. Start 1 nodes AWS EMR spark cluster
Advanced option: r4.4xlarge, 1node, spark, Hadoop
Configuration:
```
[{"Classification": "spark", "Properties": {"maximizeResourceAllocation": "true", "spark.executor.memory": "10G", "yarn.nodemanager.pmem-check-enabled": "false", "yarn.nodemanager.vmem-check-enabled": "false"}}] 
```
VPC: your VPC and subnet

2. SSH to the instance: ec2-54-90-80-85.compute-1.amazonaws.com (change)   user: hadoop 

3. sudo vi /etc/spark/conf.dist/spark-defaults.conf
```
spark.driver.maxResultSize       12g
spark.driver.memory              12g
```
4. sudo yum install -y git
5. wget http://mirrors.hust.edu.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
6. tar zxvf apache-maven-3.3.9-bin.tar.gz
7. sudo vi .bashrc
```
export MAVEN_HOME=/home/hadoop/apache-maven-3.3.9
export M2_HOME=/home/hadoop/apache-maven-3.3.9
export M2=/home/hadoop/apache-maven-3.3.9
export PATH=/home/hadoop/apache-maven-3.3.9/bin:$PATH
```
8. source .bashrc
9. sudo yum install -y git
10. git clone https://github.com/dkpro/dkpro-c4corpus
11. aws s3 cp s3://CommonCrawl/ibm/26279.gz /var/tmp/
12. gunzip 26279.gz
13. split -n84 ibm (26279) (split file to 20-100MB)
14. Sync file between S3 to EC2
```
aws s3 sync . s3://CommonCrawl/data/ibm84
aws s3 sync s3://CommonCrawl/data/ibm84 /var/tmp/.
aws s3 sync s3://CommonCrawl/boilerplate/pending /var/tmp/
aws s3 cp /var/tmp/xba_boiler s3://CommonCrawl/boilerplate/ibm/
```
15. run dkpro-c4corpus-boilerplate
```
cd dkpro-c4corpus/dkpro-c4corpus-boilerplate/
mvn package
java -jar target/dkpro-c4corpus-boilerplate-1.0.1-SNAPSHOT.jar /var/tmp/26279 /var/tmp/ibm_boiler false
```

### Wordcount process
1. 
```
spark-shell
```
IBM Wordcount process:
```Scala
val file = sc.textFile("s3://CommonCrawl/ibm_boiler")
val counts = file.flatMap(line => line.toLowerCase().replace(".", " ").replace(",", " ").split(" ")).map(word => (word, 1L)).reduceByKey(_ + _)
val sorted_counts = counts.collect().sortBy(wc => -wc._2)    (1mins)
sc.parallelize(sorted_counts.take(60000)).saveAsTextFile("s3://CommonCrawl/boilerplate/ibm_boiler _top60000")
sc.parallelize(sorted_counts).saveAsTextFile("s3://CommonCrawl/boilerplate/wordcount-ibm_bolier")
```
Netapp Wordcount process:
```Scala
val file = sc.textFile("s3://CommonCrawl/boilerplate/netapp_boiler")
val counts = file.flatMap(line => line.toLowerCase().replace(".", " ").replace(",", " ").split(" ")).map(word => (word, 1L)).reduceByKey(_ + _)
val sorted_counts = counts.collect().sortBy(wc => -wc._2)    (3mins)
sc.parallelize(sorted_counts.take(20000)).saveAsTextFile("s3://CommonCrawl/top20000_netapp_boiler")
sc.parallelize(sorted_counts).saveAsTextFile("s3://CommonCrawl/wordcount-netapp_boiler")
```
Top 10 words

|Word |Count|
|-----|:---:|
|	|4327791|
|the	|2103578|
|0	|1159355|
|to	|1097568|
|and	|1057336|
|a	|856529|
|of	|811647|
|for	|737729|
|in	|646580|
|ibm	|623663|

### Dataframe, Dataset, Data source

Convert Text to Parquet, Spark 2.0 convert into parquet file in much more efficient than spark1.6.
```
[hadoop@ip-10-0-1-27 ~]$ aws s3 cp s3://CommonCrawl/netapp_boiler_top20000_np.csv /var/tmp
[hadoop@ip-10-0-1-27 ~]$ hdfs dfs -put /var/tmp/netapp_boiler_top20000_np.csv /user/hadoop/
```
Spark 1.4+
```
spark-shell --packages com.databricks:spark-csv_2.11:1.5.0
import org.apache.spark.sql.SQLContext
val sqlContext = new SQLContext(sc)
val df = sqlContext.read.format("com.databricks.spark.csv").option("header", "true").option("inferSchema", "true").load("/user/hadoop/netapp_boiler_top20000_np.csv")
val selectedData = df.select("words", "count")
selectedData.write.format("com.databricks.spark.csv").option("header", "true").save("netappparquet.csv")
```

### Remove Stop Words

The steps are
1. remove punctuation, by replace "[^A-Za-z0-9\s]+" with "", or not include numbers "[^A-Za-z\s]+"
2. trim all spaces
3. lower all words
4. remove stop words

```
aws s3 cp s3://CommonCrawl/netapp_boiler_top20000.txt /var/tmp
hdfs dfs -mkdir /user/hadoop/data/
hdfs dfs -put /var/tmp/netapp_boiler_top20000.txt /user/hadoop/data/
```
```
spark-shell
```
```Scala
import org.apache.spark.ml.feature.StopWordsRemover
import org.apache.spark.sql.functions.split

// val reg = raw"[^A-Za-z0-9\s]+" // remove punctuation with numbers

val reg = raw"[^A-Za-z\s]+" // remove punctuation not include numbers
val lines = sc.textFile("/user/hadoop/netapp_boiler_top20000_np.csv").map(_.replaceAll(reg, "").trim.toLowerCase).toDF("line")
val words = lines.select(split($"line", " ").alias("words"))
val remover = new StopWordsRemover().setInputCol("words").setOutputCol("filtered")
val noStopWords = remover.transform(words)
remover.transform(words).show(15)

```
![alt text][logo]

[logo]:https://github.com/CI-Research/KeywordAnalysis/blob/master/s08.JPG "Logo Title Text 2"
```

//val counts = noStopWords.select(explode($"filtered")).map(word =>(word, 1)).reduceByKey(_+_)

// from word -> num to num -> word
//val mostCommon = counts.map(p => (p._2, p._1)).sortByKey(false, 1)

//mostCommon.take(5)

//dataframe dump to csv
val stringify = udf((vs: Seq[String]) => s"""[${vs.mkString(",")}]""")
words.withColumn("words", stringify($"words")).write.csv("/netapp_filtered.csv")
hdfs dfs -get /netapp_filtered.csv .
```

```
4.24
***success!sudo yum install -y gitgit clone https://github.com/phatak-dev/spark-two-migrationhdfs dfs -put /spark-two-migration/test_data/sales.csv .spark-shellimport org.apache.spark.ml.feature.StopWordsRemoverimport org.apache.spark.sql.functions.splitimport org.apache.spark.sql.types._import org.apache.spark.sql.SQLContextimport org.apache.spark.sql.types.{StructType, StructField, StringType, IntegerType}val sqlContext = new SQLContext(sc)val loadedDF = sqlContext.read.format("csv").option("header", "true").load("sales.csv")loadedDF.columnsloadedDF.show(15)****success!aws s3 cp s3://CommonCrawl/netapp_boiler_top20000_np.csv /var/tmphdfs dfs -put /var/tmp/netapp_boiler_top20000_np.csv /user/hadoop/val wikiDF = sqlContext.read.format("csv").option("header", "true").load("netapp_boiler_top20000_np.csv")wikiDF.columnswikiDF.show(15)****val wiki2DF = spark.read.parquet("saves_parquet.csv/part-00000-08d951ae-c1eb-48b8-9091-9d6dcf4d3bbf.csv").cache()wikiDF.columnswikiDF.show(5)
wikiDF.write.format("csv").option("header", "true").save("netappparquet")val wiki2DF = spark.read.parquet("netapp_parquet1.csv").cache()wikiDF.columnswikiDF.show(5)
loadedDF.write.format("csv").option("header", "true").save("saves_parquet.csv/part-00000-08d951ae-c1eb-48b8-9091-9d6dcf4d3bbf.csv")val wikiDF = spark.read.parquet("saves_parquet.csv")
val wikiDF = sqlContext.read.format("csv").option("header", "true").load("netapp_boiler_top20000_np.csv")val wikiDF = sqlContext.read.format("csv").option("header", "true").load("sales.csv")val wikiLoweredDF = wikiDF.select($"*", lower($"words").as("lowerText"))wikiLoweredDF.show(2)
import org.apache.spark.ml.feature.{RegexTokenizer, StopWordsRemover, HashingTF, IDF, Normalizer}val tokenizer = new RegexTokenizer().setInputCol("lowerText").setOutputCol("words2").setPattern("""\W+""")val wikiWordsDF = tokenizer.transform(wikiLoweredDF)wikiWordsDF.printSchemawikiWordsDF.select("words").first
import org.apache.spark.sql.SparkSessionval spark = SparkSession.builder().appName("Spark SQL basic example").config("spark.some.config.option", "some-value").getOrCreate()import spark.implicits._val sqlContext = new SQLContext(sc)val df = sqlContext.read.format("csv").option("header", "true").load("netapp_boiler_top20000_np.csv")df.show()import spark.implicits._df.printSchema()df.select("count").show()df.select($"words", $"count").show()df.filter($"count" > 10000).show()df.groupBy("count").count().show()df.groupBy("words").count().show()val usersDF = spark.read.load("netappparquet.csv")usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
dbwiki***import org.apache.spark.ml.feature.StopWordsRemoverimport org.apache.spark.sql.functions.splitimport org.apache.spark.sql.types._import org.apache.spark.sql.SQLContextimport org.apache.spark.sql.types.{StructType, StructField, StringType, IntegerType}val sqlContext = new SQLContext(sc)val netappDF = sqlContext.read.format("csv").option("header", "true").load("netapp_boiler_top20000_np.csv")netappDF.columnsnetappDF.show(15)netappDF.printSchema()netappDF.select("count").show()netappDF.select($"words", $"count").show()netappDF.filter($"count" > 10000).show()netappDF.groupBy("count").count().show()netappDF.groupBy("words").count().show()
Lower case the text:val netappLoweredDF = netappDF.select($"*", lower($"words").as("lowerText"))netappLoweredDF.show(2)
Set up the ML Pipeline:import org.apache.spark.ml.feature.{RegexTokenizer, StopWordsRemover, HashingTF, IDF, Normalizer}
Step 1: Natural Language Processing: RegexTokenizer: Convert the lowerText col to a bag of wordsval tokenizer = new RegexTokenizer().setInputCol("lowerText").setOutputCol("netappwords").setPattern("""\W+""")val netappWordsDF = tokenizer.transform(netappLoweredDF.na.drop(Array("lowerText")))netappWordsDF.printSchemanetappWordsDF.select("netappwords").first
Step 2: Natural Language Processing: StopWordsRemover: Remove Stop wordsval remover = new StopWordsRemover().setInputCol("netappwords").setOutputCol("noStopWords")val noStopWordsListDF = remover.transform(netappWordsDF)noStopWordsListDF.printSchemanoStopWordsListDF.select("words", "count", "netappwords", "noStopWords").show(20)noStopWordsListDF.show(15)
Step 3: HashingTF// More features = more complexity and computational time and accuracy
val hashingTF = new HashingTF().setInputCol("noStopWords").setOutputCol("hashingTF").setNumFeatures(20000)val featurizedDataDF = hashingTF.transform(noStopWordsListDF)featurizedDataDF.printSchemafeaturizedDataDF.select("words", "count", "netappwords", "noStopWords").show(7)
Step 4: IDF// This will take 30 seconds or so to runval idf = new IDF().setInputCol("hashingTF").setOutputCol("idf")val idfModel = idf.fit(featurizedDataDF)
Step 5: Normalizer// A normalizer is a common operation for text classification.
// It simply gets all of the data on the same scale... for example, if one article is much longer and another, it'll normalize the scales for the different features.
// If we don't normalize, an article with more words would be weighted differently
val normalizer = new Normalizer().setInputCol("idf").setOutputCol("features")
Step 6: k-means & tie it all together...
import org.apache.spark.ml.Pipelineimport org.apache.spark.ml.clustering.KMeans
val kmeans = new KMeans().setFeaturesCol("features").setPredictionCol("prediction").setK(8).setSeed(0) 
// for reproducability
val pipeline = new Pipeline().setStages(Array(tokenizer, remover, hashingTF, idf, normalizer, kmeans))
// This can take more 1 hour to run!/*val model = pipeline.fit(netappLoweredDF.na.drop(Array("lowerText")))*/
****-over all success!aws s3 cp s3://CommonCrawl/ibm_boiler_top60000.csv /var/tmphdfs dfs -put /var/tmp/ibm_boiler_top60000.csv /user/hadoop/
val model2 = org.apache.spark.ml.PipelineModel.load("netapp_boiler_top20000_np.csv")
val model2 = org.apache.spark.ml.PipelineModel.load("saves_parquet.csv")
input path error
Let's take a look at a sample of the data to see if we can see a pattern between predicted clusters and titles.val rawPredictionsDF = model.transform(netappLoweredDF.na.drop(Array("lowerText")))rawPredictionsDF.columnsrawPredictionsDF.show(10)val predictionsDF = rawPredictionsDF.select($"words", $"prediction").cachepredictionsDF.show(15)
// This could take up to 5 minutes.predictionsDF.groupBy("prediction").count().orderBy($"count" desc).show(100)display(predictionsDF.filter("prediction = 3").select("words", "prediction").limit(30))display(predictionsDF.filter("prediction = 4").select("words", "prediction").limit(30))display(predictionsDF.filter("prediction = 2").select("words", "prediction").limit(30))predictionsDF.filter($"title" === "Apache Spark").show(10)display(predictionsDF.filter("prediction = 25").limit(25))

***
```
