# Spark SQL Cheat sheet
The Spark SQL module consists of two main parts. The first one is the representation of the Structure APIs, called DataFrames and Datasets, that define the high-level APIs for working with structured data. The DataFrame concept was inspired by the Python pandas DataFrame; the main difference is that a DataFrame in Spark can handle a large volume of data that is spread across many machines. The second part of the Spark SQL module is the Catalyst optimizer, which does all the hard work behind the scenes to make your life easier and to speed up your data processing logic. One of the cool features the Spark SQL module offers is the ability to execute SQL queries to perform data processing.
The figure below shows how the Spark SQL component is built on top of the good old reliable Spark Core component.

![Spark SQL Components](SparkSQLComponents.png)
	
## DataFrames
A DataFrame is an immutable, distributed collection of data that is organized into rows, where each one consists a set of columns and each column has a name and an associated type. In other words, this distributed collection of data has a structure defined by a schema. If you are familiar with the table concept in a relational database management system (RDBMS), then you will realize that a DataFrame is essentially equivalent. Each row in the DataFrame is represented by a generic Row object.

### Creating DataFrames
DataFrames can be created by reading data from the many structured data sources as well as by reading data from tables in Hive and databases.
In addition, the Spark SQL module makes it easy to convert an RDD to a DataFrame by providing the schema information about the data in the RDD. The DataFrame APIs are available in Scala, Java, Python, and R.

#### Creating Dataframes from RDDs
Let’s start with creating a DataFrame from an RDD.
- Creating a DataFrame from an RDD of Numbers
```
import scala.util.Random
val rdd = spark.sparkContext.parallelize(1 to 10).map(x => (x, Random.nextInt(100)* x))
val kvDF = rdd.toDF("key","value")
```
Another way to create a DataFrame is by specifying an RDD with a schema that is created programmatically.
- Creating a DataFrame from an RDD with a Schema Created Programmatically
```
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._
val peopleRDD = spark.sparkContext.parallelize(Array(Row(1L, "John Doe",  30L),
                                                     Row(2L, "Mary Jane", 25L)))
val schema = StructType(Array(
        StructField("id", LongType, true),
        StructField("name", StringType, true),
        StructField("age", LongType, true)
))
val peopleDF = spark.createDataFrame(peopleRDD, schema)
```
#### Creating Dataframes from a Range of Numbers
Spark 2.0 introduced a new entry point for Spark applications. It is represented by a class called SparkSession, which has a convenient function called range that can easily create a DataFrame with a single column with the name id and the type LongType. This function has a few variations that can take additional parameters to specify the start and end of a range as well as the steps of the range.
- Using the SparkSession.range Function to Create a DataFrame
```
val df1 = spark.range(5).toDF("num").show

val df2 = spark.range(5,10).toDF("num").show

val df3 = spark.range(5,15,2).toDF("num").show // The last param represents the step size
```
Notice the range function can create only a single-column DataFrame.
One option to create a multicolumn DataFrame is to use Spark’s implicits that convert a collection of tuples inside a Scala Seq collection.
- Converting a Collection Tuple to a DataFrame Using Spark’s toDF Implicit
```
val movies = Seq(("Damon, Matt", "The Bourne Ultimatum", 2007L),
                 ("Damon, Matt", "Good Will Hunting", 1997L))
val moviesDF = movies.toDF("actor", "title", "year")
```

#### Creating DataFrames from Data Sources
Out of the box, Spark SQL supports six built-in data sources, where each data source is
mapped to a data format. The data source layer in the Spark SQL module is designed to
be extensible, so custom data sources can be easily integrated into the DataFrame APIs.
There are hundreds of custom data sources written by the Spark community, and it is not
too difficult to implement them.
The two main classes in Spark SQL for reading and writing data are DataFrameReader and DataFrameWriter, respectively.
An instance of the DataFrameReader class is available as the read variable of the SparkSession class.
- Common Pattern for Interacting with DataFrameReader
```
spark.read.format(...).option("key", value").schema(...).load()
```
- Specifying the Data Source Format
```
spark.read.json("<path>")
spark.read.format("json")
spark.read.parquet("<path>")
spark.read.format("parquet")
spark.read.jdbc
spark.read.format("jdbc")
spark.read.orc("<path>")
spark.read.format("orc")
spark.read.csv("<path>")
spark.read.format("csv")
spark.read.text("<path>")
spark.read.format("text")
// custom data source – fully qualifed package name
spark.read.format("org.example.mysource")
```
- Reading the README.md File As a Text File from a Spark Shell
```
val textFile = spark.read.text("README.md")
```
- Reading CSV Files with Various Options
```
val movies = spark.read.option("header","true").csv("<path>/book/chapter4/data/movies.csv")
```
- Various Examples of Reading a JSON File
```
val movies5 = spark.read.json("<path>/book/chapter4/data/movies/movies.json")
```
```
Specify a schema to override the Spark's inferring schema.
producted_year is specified as integer type
import org.apache.spark.sql.types._
val movieSchema2 = StructType(Array(StructField("actor_name", StringType, true),
                             StructField("movie_title", StringType, true),
                             StructField("produced_year", IntegerType, true)))
val movies6 = spark.read.option("inferSchema","true").schema(movieSchema2)
                              .json("<path>/book/chapter4/data/movies/movies.json")
```
- Reading a Parquet File in Spark
```
Parquet is the default format, so don't need to specify the format when reading
val movies9 = spark.read.load("<path>/book/chapter4/data/movies/movies.parquet")           
```
If we want to be more explicit, we can specify the path to the parquet
function
```
val movies10 = spark.read.parquet("<path>/book/chapter4/data/movies/movies.parquet")
```
- Reading an ORC File in Spark
```
val movies11 = spark.read.orc("<path>/book/chapter4/data/movies/movies.orc")
```
- Create Dataframes from jdbc
```
// First, connect MySQL ot Spark

import java.sql.DriverManager
val connectionURL = "jdbc:mysql://localhost:3306/<table>?user=<username>&password=<password>"
val connection = DriverManager.getConnection(connectionURL)
connection.isClosed()
connection close()
```
```
//Reading Data from a Table in MySQL Server

val mysqlURL= "jdbc:mysql://localhost:3306/sakila"
val filmDF = spark.read.format("jdbc").option("driver", "com.mysql.jdbc.Driver")
                                      .option("url", mysqlURL)
                                      .option("dbtable", "film")
                                      .option("user", "<username>")
                                      .option("password","<password>")
                                      .load()
```




### Working with Structured Operations
Unlike the RDD operations, the structured operations are designed to be more relational, meaning these operations mirror the kind of expressions you can do with SQL, such as projection, filtering, transforming, joining, and so on. Similar to RDD operations, the structured operations are divided into two categories: transformation and action. The semantics of the structured transformations and actions are identical to the ones in RDDs. In other words, structured transformations are lazily evaluated, and structured actions are eagerly evaluated.

#### Working with Columns

- Different Ways of Referring to a Column
```
import org.apache.spark.sql.functions._
val kvDF = Seq((1,2),(2,3)).toDF("key","value")
// to display column names in a DataFrame, we can call the columns function
kvDF.columns
Array[String] = Array(key, value)
kvDF.select("key")
kvDF.select(col("key"))
kvDF.select(column("key"))
kvDF.select($"key")
kvDF.select('key)
// using the col function of DataFrame
kvDF.select(kvDF.col("key"))
kvDF.select('key, 'key > 1).show
```
#### Working with Structured Transformations
- Creating the movies Dataframe from a Parquet file
```
val movies = spark.read.parquet("<path>/chapter4/data/movies/movies.parquet")
```
#### select(columns)
This transformation is commonly used to perform projection, meaning selecting all
or a subset of columns from a DataFrame. During the selection, each column can be transformed via a column expression. There are two variations of this transformation. One takes the column as a string, and the other takes columns as the Column class. This transformation does not permit you to mix the column type when using one of these two variations.
- Two Variations of the select Transformation
```
movies.select("movie_title","produced_year").show(5)

// using a column expression to transform year to decade
movies.select('movie_title,('produced_year - ('produced_year % 10)).as("produced_decade")).show(5)
```
#### selectExpr(expressions)
This transformation is a variant of the select transformation. The one big difference is that it accepts one or more SQL expressions, rather than columns. However, both are essentially performing the same projection task. SQL expressions are powerful and flexible constructs to allow you to express column transformation logic in a natural way, just like the way you think. You can express SQL expressions in a string format, and Spark will parse them into a logical tree so they will be evaluated in the right order.
- Adding the decade Column to the movies DataFrame Using a SQL Expression
```
movies.selectExpr("*","(produced_year - (produced_year % 10)) as decade").
show(5)
```
- Using a SQL Expression and Built-in Functions
```
movies.selectExpr("count(distinct(movie_title)) as movies","count(distinct(actor_name)) as actors").show
```
#### filler(condition), where(condition)
This transformation is a fairly straightforward one to understand. It is used to filter out the rows that don’t meet the given condition, in other words, when the condition evaluates to false. A different way of looking at the behavior of the filter transformation is that it returns only the rows that meet the specified condition. Both the filter and where transformations have the same behavior, so pick the one you are most comfortable with. The latter one is just a bit more relational than the former.
- Filter Rows with Logical Comparison Functions in the Column Class
```
movies.filter('produced_year < 2000)
movies.where('produced_year > 2000)
movies.filter('produced_year >= 2000)
movies.where('produced_year >= 2000)
// equality comparison requires 3 equal signs
movies.filter('produced_year === 2000).show(5)
// inequality comparison uses an interesting looking operator =!=
movies.select("movie_title","produced_year").filter('produced_year =!=2000).show(5)
// to combine one or more comparison expressions, we will use either the OR and AND expression operator
movies.filter('produced_year >= 2000 && length('movie_title) < 5).show(5)
// the other way of accomplishing the same result is by calling the filter function two times
movies.filter('produced_year >= 2000).filter(length('movie_title) < 5).show(5)
```
#### distinct, dropDuplicates
These two transformations have identical behavior. However, dropDuplicates allows you to control which columns should be used in deduplication logic. If none is specified, the deduplication logic will use all the columns in the DataFrame.
- Using distinct and dropDuplicates to Achieve the Same Goal
```
movies.select("movie_title").distinct.selectExpr("count(movie_title) as movies").show
movies.dropDuplicates("movie_title").selectExpr("count(movie_title) as movies").show
```
#### sort(columns), orderBy(columns)
Both of these transformations have the same semantics. The orderBy transformation is more relational than the other one. By default, the sorting is in ascending order, and it is fairly easy to change it to descending. When specifying more than one column, it is possible to have a different order for each of the columns.
- Sorting the DataFrame in Ascending and Descending Order
```
val movieTitles = movies.dropDuplicates("movie_title")
                        .selectExpr("movie_title", "length(movie_title) as
			 title_length", , "produced_year")
movieTitles.sort('title_length).show(5)
// sorting in descending order
movieTitles.orderBy('title_length.desc).show(5)
// sorting by two columns in different orders
movieTitles.orderBy('title_length.desc, 'produced_year).show(5)
```
#### limit(n)
This transformation returns a new DataFrame by taking the first n rows. This transformation is commonly used after the sorting is done to figure out the top n or bottom n rows based on the sorting order.
- Using the limit Transformation to Figure Out the Top Ten Actors with the Longest Names
```
// first create a DataFrame with their name and associated length
val actorNameDF = movies.select("actor_name").distinct.selectExpr("*", "length(actor_name) as length")
// order names by length and retrieve the top 10
actorNameDF.orderBy('length.desc).limit(10).show
```
#### union(otherDataFrame)
We learned earlier that DataFrames are immutable. So if there is a need to add more rows to an existing DataFrame, then the union transformation is useful for that purpose as well as for combining rows from two DataFrames. This transformation requires both DataFrames to have the same schema, meaning both column names and their order must exactly match.
- Adding a Missing Actor to the movies DataFrame
```
// we want to add a missing actor to movie with title as "12"
val shortNameMovieDF = movies.where('movie_title === "12")

// create a DataFrame with one row
import org.apache.spark.sql.Row
val forgottenActor = Seq(Row("Brychta, Edita", "12", 2007L))
val forgottenActorRDD = spark.sparkContext.parallelize(forgottenActor)
val forgottenActorDF = spark.createDataFrame(forgottenActorRDD,shortNameMovieDF.schema)

// now adding the missing actor
val completeShortNameMovieDF = shortNameMovieDF.union(forgottenActorDF)
```
#### withColumn(colName, column)
This transformation is used to add a new column to a DataFrame. It requires two input parameters: a column name and a value in the form of a column expression. You can accomplish pretty much the same goal by using the selectExpr transformation. However, if the given column name matches one of the existing ones, then that column is replaced with the given column expression.
- Adding a Column As Well As Replacing a Column Using the withColumn Transformation
```
// adding a new column based on a certain column expression
movies.withColumn("decade", ('produced_year - 'produced_year % 10)).show(5)

// now replace the produced_year with new values
movies.withColumn("produced_year", ('produced_year - 'produced_year % 10)).show(5)
```
#### withColumnRenamed(existingColName, newColName)
This transformation is strictly about renaming an existing column name in a DataFrame. Notice that if the provided existingColName doesn’t exist in the schema, Spark doesn’t throw an error, and it will silently do nothing.
- Using the withColumnRenamed Transformation to Rename Some of the Column Names
```
movies.withColumnRenamed("actor_name", "actor")
      .withColumnRenamed("movie_title", "title")
      .withColumnRenamed("produced_year", "year").show(5)
```
#### drop(columnName1, columnName2)
This transformation simply drops the specified columns from the DataFrame. You can specify one or more column names to drop, but only the ones that exist in the schema will be dropped and the ones that don’t will be silently ignored.
```
movies.drop("actor_name", "me").printSchema
```
#### sample(fraction), sample(fraction, seed), sample(fraction, seed, withReplacement)
This transformation returns a randomly selected set of rows from the DataFrame. The number of the returned rows will be approximately equal to the specified fraction, which represents a percentage, and the value has to be between 0 and 1.
- Different Ways of Using the sample Transformation
```
// sample with no replacement and a fraction
movies.sample(false, 0.0003).show(3)

// sample with replacement, a fraction and a seed
movies.sample(true, 0.0003, 123456).show(3)
```
#### randomSplit(weights)
This transformation is commonly used during the process of preparing the data to train machine learning models. Unlike the previous transformations, this one returns one or more DataFrames. The number of DataFrames it returns is based on the number of weights you specify. If the provided set of weights don’t add up to 1, then they will be
normalized accordingly to add up to 1.
- Using randomSplit to split the movies DataFrames into Three Parts
```
// the weights need to be an Array
val smallerMovieDFs = movies.randomSplit(Array(0.6, 0.3, 0.1))
```
#### Working with Missing or Bad Data
- Dropping Rows with Missing Data
```
// dropping rows that have missing data in any column
// both of the lines below will achieve the same purpose
badMoviesDF.na.drop().show
badMoviesDF.na.drop("any").show

// drop rows that have missing data in every single column
badMoviesDF.na.drop("all").show

// drops rows when column actor_name has missing data
badMoviesDF.na.drop(Array("actor_name")).show
```
#### describe(columnNames)
- Use the describe Transformation to Show Statistics for the produced_year Column
```
movies.describe("produced_year").show
```
#### Working with Structured Actions
![Structured Actions](StructuredActions.png)

## Datasets
Consider the Dataset as a younger brother of the DataFrame; however, it is more about
type safety and is object-oriented. A Dataset is a strongly typed, immutable collection
of data. Similar to a DataFrame, the data in a Dataset is mapped to a defined schema.
The Dataset APIs are good for production jobs that need to run on a regular basis and are written and maintained by data engineers. For most interactive and explorative analysis use cases, using the DataFrame APIs would be sufficient.

### Creating Datasets
There are a few ways to create a Dataset, but the first thing you need to do is to define a domain-specific object to represent each row. The first way is to transform a DataFrame to a Dataset using the as(Symbol) function of the DataFrame class. The second way is to use the SparkSession.createDataset() function to create a Dataset from a local collection objects. The third way is to use the toDS implicit conversion utility.
- Different Ways of Creating Datasets
```
// define Movie case class
case class Movie(actor_name:String, movie_title:String, produced_year:Long)

// convert DataFrame to strongly typed Dataset
val moviesDS = movies.as[Movie]

// create a Dataset using SparkSession.createDataset() and the toDS implicit function
val localMovies = Seq(Movie("John Doe", "Awesome Movie", 2018L),
                                 Movie("Mary Jane", "Awesome Movie", 2018L))
val localMoviesDS1 = spark.createDataset(localMovies)
val localMoviesDS2 = localMovies.toDS()
localMoviesDS1.show
```
### Working with Datasets
Now that you have a Dataset, you can manipulate it using the transformations and actions described earlier. Previously you referred to the columns in the DataFrame using one of the options described earlier. With a Dataset, each row is represented by a strongly typed object; therefore, you can just refer to the columns using the member variable names, which will give you type safety as well as compile-time validation.
If there is a misspelling in the name, the compiler will flag them immediately during the
development phase.
- Manipulating a Dataset in a Type-Safe Manner
```
// filter movies that were produced in 2010 using
moviesDS.filter(movie => movie.produced_year == 2010).show(5)

// displaying the title of the first movie in the moviesDS
moviesDS.first.movie_title

// perform projection using map transformation
val titleYearDS = moviesDS.map(m => ( m.movie_title, m.produced_year))

// demonstrating a type-safe transformation that fails at compile time, performing subtraction on a column with string type
// a problem is not detected for DataFrame until runtime
movies.select('movie_title - 'movie_title)

// a problem is detected at compile time
moviesDS.map(m => m.movie_title - m.movie_title)
error: value - is not a member of String

// take action returns rows as Movie objects to the driver
moviesDS.take(5)
Array[Movie] = Array(Movie(McClure, Marc (I),Coach Carter,2005), Movie(McClure, Marc (I),Superman II,1980), Movie(McClure, Marc (I),Apollo 13,1995))
```
### Running SQL in Spark
Spark provides a few different ways to run SQL.
- Spark SQL CLI (./bin/spark-sql)
- JDBC/ODBC server
- Programmatically in Spark applications  

DataFrames and Datasets are essentially like tables in a database. Before you can issue SQL queries to manipulate them, you need to register them as temporary views.
- Registering the movies DataFrame as a Temporary View and Inspecting the Spark Metadata Catalog
```
// display tables in the catalog, expecting an empty list
spark.catalog.listTables.show

// now register movies DataFrame as a temporary view
movies.createOrReplaceTempView("movies")

// should see the movies view in the catalog
spark.catalog.listTables.show

// show the list of columns of movies view in catalog
spark.catalog.listColumns("movies").show

// register movies as global temporary view called movies_g
movies.createOrReplaceGlobalTempView("movies_g")
```
- Programmatically Executing SQL Statements in Spark
```
val infoDF = spark.sql("select current_date() as today , 1 + 100 as value")
infoDF.show

// select from a view
spark.sql("select * from movies where actor_name like '%Jolie%' and
produced_year > 2009").show

// mixing SQL statement and DataFrame transformation
spark.sql("select actor_name, count(*) as count from movies group by actor_name")
         .where('count > 30)
         .orderBy('count.desc)
         .show

// using a subquery to figure out the number movies were produced in each year.
// leverage """ to format multi-line SQL statement
spark.sql("""select produced_year, count(*) as count
                   from (select distinct movie_title, produced_year from
movies)
                   group by produced_year""")
         .orderBy('count.desc).show(5)      

// select from a global view requires prefixing the view name with key word 'global_temp'
spark.sql("select count(*) from global_temp.movies_g").show
```
### Writing Data Out to Storage Systems
In Spark SQL, the DataFrameWriter class is responsible for the logic and complexity of writing out the data in a DataFrame to an external storage system. An instance of the DataFrameWriter class is available to you as the write variable in the DataFrame class.
- Common Interacting Pattern with DataFrameWriter
```
movies.write.format(...).mode(...).option(...).partitionBy(...).bucketBy(...)
.sortBy(...).save(path)
```
- Using DataFrameWriter to Write Data to File-Based Sources
```
// write data out as CVS format, but using a '#' as delimiter
movies.write.format("csv").option("sep", "#").save("/tmp/output/csv")

// write data out using overwrite save mode
movies.write.format("csv").mode("overwrite").option("sep", "#").save("/tmp/output/csv")
```
The number of files written out to the output directory corresponds to the number of partitions a DataFrame has.
- Displaying the Number of Partitions a DataFrame Has
```
movies.rdd.getNumPartitions
```
In some cases, the content of a DataFrame is not large, and there is a need to write to a single file. A small trick to achieve this goal is to reduce the number of partitions in your DataFrame to one and then write it out.
- Reducing the Number of Partitions in a DataFrame to 1
```
val singlePartitionDF = movies.coalesce(1)
```
- Writing the movies DataFrame Using the Parquet Format and Partition by the produced_year Column
```
movies.write.partitionBy("produced_year").save("/tmp/output/movies")
// the /tmp/output/movies directory will contain the following subdirectories 
produced_year=1961 to produced_year=2012
```
### DataFrame Persistence
- Persisting a DataFrame with a Human-Readable Name
```
val numDF = spark.range(1000).toDF("id")
// register as a view
numDF.createOrReplaceTempView("num_df")
// use Spark catalog to cache the numDF using name "num_df"
spark.catalog.cacheTable("num_df")
// force the persistence to happen by taking the count action
numDF.count
```
## Spark SQL (Advanced)

### Aggregation Functions
In Spark, all aggregations are done via functions. The aggregation functions are designed to perform aggregation on a set of rows, whether that set of rows consists of all the rows or a subgroup of rows in a DataFrame.
#### count(col)
Counting is a commonly used aggregation to find out the number of items in a group.
- Computing the Count for Different Columns in the flight_summary DataFrame
```
flight_summary.select(count("origin_airport"), count("dest_airport").as("dest_count")).show
```
#### countDistinct(col)
This function does what it sounds like. It counts only the unique items per group.
- Counting Unique Items in a Group
```
flight_summary.select(countDistinct("origin_airport"), countDistinct("dest_
airport"), count("*")).show
```
#### approx_count_distinct (col, max_estimated_error=0.05)
Counting the exact number of unique items in each group in a large dataset is an expensive and time-consuming operation. In some use cases, it is sufficient to have an approximate unique count. One of those use cases is in the online advertising business where there are hundreds of millions of ad impressions per hour and there is a need to generate a report to show the number of unique visitors per certain type of member segment. Approximating a count of distinct items is a well-known problem in the computer science field, and it is also known as the cardinality estimation problem.
- Counting Unique Items in a Group
```
// let's do the counting on the "count" colum of flight_summary DataFrame.  
// the default estimation error is 0.05 (5%)
flight_summary.select(count("count"),countDistinct("count"), approx_count_
distinct("count", 0.05)).show
```
#### min(col), max(col)
- Getting the Minimum and Maximum Values of the count Column
```
flight_summary.select(min("count"), max("count")).show
```
#### sum(col)
This function computes the sum of the values in a numeric column.
- Using the sum Function to Sum Up the count Values
```
flight_summary.select(sum("count")).show
```
#### sumDistinct(col)
This function does what it sounds like. It sums up only the distinct values of a numeric column.
- Using the sumDistinct Function to Sum Up the Distinct count Values
```
flight_summary.select(sumDistinct("count")).show
```
#### avg(col)
This function calculates the average value of a numeric column. This convenient function simply takes the total and divides it by the number of items.
- Computing the Average Value of the count Column Using Two Different Ways
```
flight_summary.select(avg("count"), (sum("count") / count("count"))).show
```
#### skewness(col), kurtosis(col)
In the field of statistics, the distribution of the values in a dataset tells a lot of stories behind the dataset. Skewness is a measure of the symmetry of the value distribution in a dataset.
Kurtosis is a measure of the shape of the distribution curve, whether the curve is normal, flat, or pointy. Positive kurtosis indicates the curve is slender and pointy, and negative kurtosis indicates the curve is fat and flat.
- Computing the Skewness and Kurtosis of the column Count
```
flight_summary.select(skewness("count"), kurtosis("count")).show
```
#### variance(col), stddev(col)
- Computing the Variance and Standard Deviation Using the variance and sttdev Functions  
In statistics, variance and standard deviation are used to measure the dispersion, or the spread, of the data.
```
// use the two variations of variance and standard deviation
flight_summary.select(variance("count"), var_pop("count"), stddev("count"), stddev_pop("count")).show
```
### Aggregation with Grouping

- Grouping by origin_airport and Performing a count Aggregation
```
flight_summary.groupBy("origin_airport").count().show(5, false)
```
- Grouping by origin_state and origin_city and Performing a Count Aggregation
```
flight_summary.groupBy('origin_state, 'origin_city).count
                .where('origin_state === "CA").orderBy('count.desc).show(5)
```
### Multiple Aggregations per Group
- Multiple Aggregations After Grouping by origin_airport
```
import org.apache.spark.sql.functions._
flight_summary.groupBy("origin_airport")
                        .agg(count("count").as("count"),
                             min("count"), max("count"),
                             sum("count")).show(5)
```
- Specifying Multiple Aggregations Using a Key-Value Map
```
flight_summary.groupBy("origin_airport")
                        .agg(
                                 "count" -> "count"
                                 "count" -> "min"
                                 "count" -> "max"
                                 "count" -> "sum"
                       .show(5)
```











































