# Using DynamoDB Streams in Apache Spark
Kinesis Adapter implementation for DynamoDB streams for Spark Streaming

# Steps
* Clone this repo
* Build the package jar using mvn
* Include it in your project dependencies manually


The implementation is a modification of [kinesis-asl](https://github.com/apache/spark/tree/master/external/kinesis-asl") artifact from spark-2.4.0 package


# Usage Sample

```
import java.util

import com.amazonaws.services.dynamodbv2.document.internal.InternalUtils
import com.amazonaws.services.dynamodbv2.streamsadapter.model.RecordAdapter
import com.amazonaws.services.kinesis.model.Record
import org.apache.spark.sql.{Encoders, Row, SparkSession}
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming.kinesis.dynamostream.KinesisInitialPositions.Latest
import org.apache.spark.streaming.kinesis.dynamostream.KinesisInputDStream
import org.apache.spark.streaming.{Milliseconds, Seconds, StreamingContext}

object DynamoStreamApp extends App {

  //session setup
  val sparkSession = SparkSession.builder()
    .master("local[*]")
    .appName("test-app")
    .getOrCreate()
  val sc = sparkSession.sparkContext
  val ssc = new StreamingContext(sc, Seconds(10))
  val sqlContext = sparkSession.sqlContext

  //creates an array of strings from raw byte array
  def rawRecordHandler: Record => Array[String] = (record: Record) => new String(record.getData.array()).split(",")

  //converts records to map of attribute value pair
  def recordHandler: Record => util.Map[String, Nothing] = (record: Record) => {
    val sRecord = record.asInstanceOf[RecordAdapter].getInternalObject
    InternalUtils.toSimpleMapValue(sRecord.getDynamodb.getNewImage)
  }

  //case class that can represent your schema
  case class MyClass(id:String,amount:Int,dummyValue:String)

  val stream_raw = KinesisInputDStream.builder
    .streamingContext(ssc)
    .streamName("sample-tablename-1")
    .regionName("us-east-1")
    .initialPosition(new Latest())
    .checkpointAppName("sample-app")
    .checkpointInterval(Milliseconds(100))
    .storageLevel(StorageLevel.MEMORY_AND_DISK_2)
    .buildWithMessageHandler(rawRecordHandler)

  val stream = KinesisInputDStream.builder
    .streamingContext(ssc)
    .streamName("sample-tablename-2")
    .regionName("us-east-1")
    .initialPosition(new Latest())
    .checkpointAppName("sample-app")
    .checkpointInterval(Milliseconds(100))
    .storageLevel(StorageLevel.MEMORY_AND_DISK_2)
    .buildWithMessageHandler(recordHandler)

  //some processing on rdd
  stream_raw.print() //should print the record in the form of {"attribute"->"value"} map

  case class SchemeClass(id:String,value;String)
  //creating dataframe, can be stored as temp view
  val sampleSchema = Encoders.product[SchemeClass].schema
  stream.foreachRDD(rdd => {
    val rowRdd = rdd.map(r => Row.fromSeq(r))
    val df = sqlContext.createDataFrame(rowRdd, cabSchema)
    df.printSchema() // should print a schema similar to your case class
  })
  ssc.start()
  ssc.awaitTermination()
}
```
