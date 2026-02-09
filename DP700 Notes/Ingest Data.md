---


---

<h2 id="ingest-data--dataflows-gen2">Ingest Data : Dataflows Gen2</h2>
<ul>
<li><em>Dataflows Gen2</em>  are used to ingest and transform data from multiple sources, and then land the cleansed data to another destination.</li>
<li>They can be incorporated into data pipelines for more complex activity orchestration, and also used as a data source in Power BI.</li>
<li><em>Dataflows Gen2</em> allow you to prepare the data to ensure consistency, and then stage the data in the preferred destination.</li>
<li>Dataflows can be horizontally partitioned as well. Once you create a global dataflow, data analysts can use dataflows to create specialized semantic models for specific needs.</li>
<li>In Microsoft Fabric, you can create a <em>Dataflow Gen2</em> in the Data Factory workload or Power BI workspace, or directly in the lakehouse.</li>
<li>You can also load your dataflow to Azure SQL database, Azure Data Explorer, or Azure Synapse Analytics.</li>
</ul>
<p>Dataflows Gen2 provide a low-to-no-code solution to ingest, transform, and load data into your Fabric data stores.</p>
<h3 id="benefits">Benefits:</h3>
<ul>
<li>Extend data with consistent data, such as a standard date dimension table.</li>
<li>Allow self-service users access to a subset of data warehouse separately.</li>
<li>Optimize performance with dataflows, which enable extracting data once for reuse, reducing data refresh time for slower sources.</li>
<li>Simplify data source complexity by only exposing dataflows to larger analyst groups.</li>
<li>Ensure consistency and quality of data by enabling users to clean and transform data before loading it to a destination.</li>
<li>Simplify data integration by providing a low-code interface that ingests data from various sources.</li>
</ul>
<h3 id="limitations">Limitations:</h3>
<ul>
<li>Dataflows aren’t a replacement for a data warehouse.</li>
<li>Row-level security isn’t supported.</li>
<li>Fabric capacity workspace is required.</li>
</ul>
<h3 id="dataflows-gen2-and-pipelines">Dataflows Gen2 and Pipelines</h3>
<p>Data pipelines are a common concept in data engineering and offer a wide variety of activities to orchestrate. Some common activities include:</p>
<ul>
<li>Copy data</li>
<li>Incorporate Dataflow</li>
<li>Add Notebook</li>
<li>Get metadata</li>
<li>Execute a script or stored procedure</li>
</ul>
<h2 id="ingest-data--understand-pipelines">Ingest Data : Understand pipelines</h2>
<p>Pipelines in Microsoft Fabric encapsulate a sequence of <em>activities</em> that perform data movement and processing tasks.</p>
<p>You can use a pipeline to define data transfer and transformation activities, and orchestrate these activities through control flow activities that manage branching, looping, and other typical processing logic.</p>
<h3 id="core-pipeline-concepts">Core pipeline concepts</h3>
<h4 id="activities">Activities</h4>
<p>Activities are the executable tasks in a pipeline. You can define a flow of activities by connecting them in a sequence.</p>
<p>The outcome of a particular activity (success, failure, or completion) can be used to direct the flow to the next activity in the sequence.</p>
<p>There are two broad categories of activity in a pipeline.</p>
<ul>
<li>
<p>Data transformation activities  - data transfer operations, like <em>Copy Data</em> , and more complex  <em>Data Flow</em>  activities that encapsulate dataflows (Gen2). Other data transformation activities include  <em>Notebook</em>  activities to run a Spark notebook,  <em>Stored procedure</em>  activities to run SQL code,  <em>Delete data</em>  activities to delete existing data, and others.</p>
</li>
<li>
<p>Control flow activities  - activities that you can use to implement loops, conditional branching, or manage variable and parameter values. The wide range of control flow activities enables you to implement complex pipeline logic to orchestrate data ingestion and transformation flow.</p>
</li>
</ul>
<h4 id="parameters">Parameters</h4>
<p>Pipelines can be parameterized, enabling you to provide specific values to be used each time a pipeline is run. For example, you might want to use a pipeline to save ingested data in a folder, but have the flexibility to specify a folder name each time the pipeline is run.</p>
<p>Using parameters increases the reusability of your pipelines, enabling you to create flexible data ingestion and transformation processes.</p>
<h4 id="pipeline-runs">Pipeline runs</h4>
<p>Each time a pipeline is executed, a  <em>data pipeline run</em>  is initiated. Runs can be initiated on-demand in the Fabric user interface or scheduled to start at a specific frequency. Use the unique run ID to review run details to confirm they completed successfully and investigate the specific settings used for each execution.</p>
<h4 id="pipeline-templates">Pipeline templates</h4>
<p>You can define pipelines from any combination of activities you choose, enabling to create custom data ingestion and transformation processes to meet your specific needs. However, there are many common pipeline scenarios for which Microsoft Fabric includes predefined pipeline templates that you can use and customize as required.</p>
<p>To create a pipeline based on a template, select the  <em>Templates</em>  tile in a new pipeline.</p>
<h3 id="run-and-monitor-pipelines">Run and monitor pipelines</h3>
<p>When you have completed a pipeline, you can use the <em>Validate</em> option to check that the configuration is valid, and then either run it interactively or specify a schedule.<br>
You can view the <strong>run history</strong> for a pipeline to see details of each run, either from the pipeline canvas or from the pipeline item listed in the page for the workspace.</p>
<h2 id="ingest-data--use-apache-spark">Ingest Data : Use Apache Spark</h2>
<p>Apache Spark is an open source parallel processing framework for large-scale data processing and analytics.<br>
Spark is popular in “big data” processing scenarios, and is available in multiple platform implementations; including <em>Azure HDInsight</em>, <em>Azure Synapse Analytics</em>, and <em>Microsoft Fabric</em>.</p>
<p>Apache Spark is a distributed data processing framework that enables large-scale data analytics by coordinating work across multiple processing nodes in a cluster, known in Microsoft Fabric as a  <em>Spark pool</em>.<br>
Put more simply, Spark uses a “divide and conquer” approach to processing large volumes of data quickly by distributing the work across multiple computers. The process of distributing tasks and collating results is handled for you by Spark.</p>
<p>Spark can run code in Java, Scala (a Java-based scripting language), Spark R, Spark SQL, and PySpark (a Spark-specific variant of Python).</p>
<p>In practice, most data engineering and analytics workloads are accomplished using a combination of PySpark and Spark SQL.</p>
<h3 id="spark-pools">Spark pools</h3>
<p>A Spark pool consists of compute  <em>nodes</em>  that distribute data processing tasks.</p>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-pool.png" alt="spark-pool-1"></p>
<p>As shown in the diagram, a Spark pool contains two kinds of node:</p>
<ol>
<li>A  <em>head</em>  node in a Spark pool coordinates distributed processes through a  <em>driver</em>  program.</li>
<li>The pool includes multiple  <em>worker</em>  nodes on which  <em>executor</em>  processes perform the actual data processing tasks.</li>
</ol>
<p>The Spark pool uses this distributed compute architecture to access and process data in a compatible data store - such as a data lakehouse based in OneLake.</p>
<h3 id="spark-pools-in-microsoft-fabric">Spark pools in Microsoft Fabric</h3>
<p>Microsoft Fabric provides a  <em>starter pool</em>  in each workspace, enabling Spark jobs to be started and run quickly with minimal setup and configuration. You can configure the starter pool to optimize the nodes it contains in accordance with your specific workload needs or cost constraints.</p>
<p>Additionally, you can create custom Spark pools with specific node configurations that support your particular data processing needs.</p>
<p>Learn More :<br>
<a href="https://learn.microsoft.com/en-us/fabric/data-engineering/capacity-settings-overview">Capacity Administration</a></p>
<p>You can manage settings for the starter pool and create new Spark pools in the <strong>Admin portal</strong> section of the workspace settings, under <strong>Capacity settings</strong>, then <strong>Data Engineering/Science Settings.</strong></p>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-settings.png" alt="admin-portal-spark"></p>
<p>Specific configuration settings for Spark pools include:</p>
<ul>
<li><strong>Node Family</strong>: The type of virtual machines used for the Spark cluster nodes. In most cases,  <em>memory optimized</em>  nodes provide optimal performance.</li>
<li><strong>Autoscale</strong>: Whether or not to automatically provision nodes as needed, and if so, the initial and maximum number of nodes to be allocated to the pool.</li>
<li><strong>Dynamic allocation</strong>: Whether or not to dynamically allocate  <em>executor</em>  processes on the worker nodes based on data volumes.</li>
</ul>
<p>Learn More:<br>
<a href="https://learn.microsoft.com/en-us/fabric/data-engineering/configure-starter-pools">Configure starter pool</a><br>
<a href="https://learn.microsoft.com/en-us/fabric/data-engineering/create-custom-spark-pools">Create custom spark pool</a></p>
<h3 id="runtimes-and-environments">Runtimes and environments</h3>
<p>The Spark open source ecosystem includes multiple versions of the Spark  <em>runtime</em>, which determines the version of Apache Spark, Delta Lake, Python, and other core software components that are installed. Additionally, within a runtime you can install and use a wide selection of code libraries for common (and sometimes very specialized) tasks.</p>
<p>In some cases, organizations may need to define multiple <em>environments</em> to support a diverse range of data processing tasks. Each environment defines a specific runtime version as well as the libraries that must be installed to perform specific operations. Data engineers and scientists can then select which environment they want to use with a Spark pool for a particular task.</p>
<p>Learn More:<br>
<a href="https://learn.microsoft.com/en-us/fabric/data-engineering/runtime">Apache Spark Runtimes in Fabric</a></p>
<h3 id="environments-in-ms-fabric">Environments in MS Fabric</h3>
<p>You can create custom environments in a Fabric workspace, enabling you to use specific Spark runtimes, libraries, and configuration settings for different data processing operations.<br>
<img src="https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-environment.png" alt="spark-env-01"></p>
<p>When creating an environment, you can:</p>
<ul>
<li>Specify the Spark runtime it should use.</li>
<li>View the built-in libraries that are installed in every environment.</li>
<li>Install specific public libraries from the Python Package Index (PyPI).</li>
<li>Install custom libraries by uploading a package file.</li>
<li>Specify the Spark pool that the environment should use.</li>
<li>Specify Spark configuration properties to override default behavior.</li>
<li>Upload resource files that need to be available in the environment.</li>
</ul>
<p>Learn More:<br>
<a href="https://learn.microsoft.com/en-us/fabric/data-engineering/create-and-use-environment">Create, configure, and use an environment in Microsoft Fabric</a></p>
<h3 id="additional-spark-configuration-options">Additional Spark configuration options</h3>
<h4 id="native-execution-engine">Native execution engine</h4>
<p>The  <em>native execution engine</em>  in Microsoft Fabric is a vectorized processing engine that runs Spark operations directly on lakehouse infrastructure.</p>
<p>Using the native execution engine can significantly improve the performance of queries when working with large data sets in Parquet or Delta file formats.</p>
<p>To use the native execution engine, you can enable it at the environment level or within an individual notebook.</p>
<p>To enable the native execution engine at the environment level, set the following Spark properties in the environment configuration:</p>
<ul>
<li><strong>spark.native.enabled</strong>: true</li>
<li><strong>spark.shuffle.manager</strong>: org.apache.spark.shuffle.sort.ColumnarShuffleManager</li>
</ul>
<p>To enable the native execution engine for a specific script or notebook, you can set these configuration properties at the beginning of your code, like this:</p>
<pre><code>%%configure 
{ 
   "conf": {
       "spark.native.enabled": "true", 
       "spark.shuffle.manager": "org.apache.spark.shuffle.sort.ColumnarShuffleManager" 
   } 
}
</code></pre>
<p>Learn More:<br>
<a href="https://learn.microsoft.com/en-us/fabric/data-engineering/native-execution-engine-overview">Native execution engine for Fabric Spark</a></p>
<h4 id="high-concurrency-mode">High concurrency mode</h4>
<p>When you run Spark code in Microsoft Fabric, a Spark session is initiated. You can optimize the efficiency of Spark resource usage by using  <em>high concurrency mode</em>  to share Spark sessions across multiple concurrent users or processes.</p>
<p>A notebook uses a Spark session for its execution. When high concurrency mode is enabled, multiple users can, for example, run code in notebooks that use the same Spark session, while ensuring isolation of code to avoid variables in one notebook being affected by code in another notebook.</p>
<p>You can also enable high concurrency mode for Spark jobs, enabling similar efficiencies for concurrent non-interactive Spark script execution.</p>
<p>Learn More:<br>
<a href="https://learn.microsoft.com/en-us/fabric/data-engineering/high-concurrency-overview">High concurrency mode in Apache Spark for Fabric</a></p>
<h4 id="automatic-mlflow-logging">Automatic MLFlow logging</h4>
<p>MLFlow is an open source library that is used in data science workloads to manage machine learning training and model deployment. A key capability of MLFlow is the ability to log model training and management operations. By default, Microsoft Fabric uses MLFlow to implicitly log machine learning experiment activity without requiring the data scientist to include explicit code to do so. You can disable this functionality in the workspace settings.</p>
<h3 id="spark-in-notebook-using-pyspark">Spark in Notebook using Pyspark</h3>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/notebook.png" alt="spark-in-notebook"></p>
<h3 id="spark-job-definition">Spark job definition</h3>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-job.png" alt="spark-job-def"></p>
<h3 id="spark-dataframe-in-pyspark">Spark dataframe in Pyspark</h3>
<p>Natively, Spark uses a data structure called a <em>resilient distributed dataset</em> (RDD); but while you <em>can</em> write code that works directly with RDDs, the most commonly used data structure for working with structured data in Spark is the <em>dataframe</em>, which is provided as part of the <em>Spark SQL</em> library. Dataframes in Spark are similar to those in the ubiquitous <em>Pandas</em> Python library, but optimized to work in Spark’s distributed processing environment.</p>
<blockquote>
<p><strong>Note</strong>: In addition to the Dataframe API, Spark SQL provides a<br>
strongly-typed  <em>Dataset</em>  API that is supported in Java and Scala.<br>
We’ll focus on the Dataframe API in this module.</p>
</blockquote>
<h4 id="inferring-a-schema">Inferring a schema</h4>
<p>Pyspark</p>
<pre><code>%%pyspark
df = spark.read.load('Files/data/products.csv',
    format='csv',
    header=True
)
display(df.limit(10))
</code></pre>
<p>Scala</p>
<pre><code>%%spark
val df = spark.read.format("csv").option("header", "true").load("Files/data/products.csv")
display(df.limit(10))
</code></pre>
<h4 id="explicit-schema">Explicit schema</h4>
<p>Pyspark</p>
<pre><code>from pyspark.sql.types import *
from pyspark.sql.functions import *

productSchema = StructType([
    StructField("ProductID", IntegerType()),
    StructField("ProductName", StringType()),
    StructField("Category", StringType()),
    StructField("ListPrice", FloatType())
    ])

df = spark.read.load('Files/data/product-data.csv',
    format='csv',
    schema=productSchema,
    header=False)
display(df.limit(10))
</code></pre>
<blockquote>
<p><strong>Tip:</strong> Specifying an explicit schema also improves performance!</p>
</blockquote>
<h4 id="filtering-and-grouping">Filtering and grouping</h4>
<p><em>Selecting columns:</em></p>
<pre><code>pricelist_df = df.select("ProductID", "ListPrice")
</code></pre>
<p>or</p>
<pre><code>pricelist_df = df["ProductID", "ListPrice"]
</code></pre>
<p><em>Chaining functions:</em></p>
<pre><code>bikes_df = df.select("ProductName", "Category", "ListPrice").where((df["Category"]=="Mountain Bikes") | (df["Category"]=="Road Bikes"))
display(bikes_df)
</code></pre>
<p><em>Group by:</em></p>
<pre><code>counts_df = df.select("ProductID", "Category").groupBy("Category").count()
display(counts_df)
</code></pre>
<p><em>Saving df to parquet:</em></p>
<pre><code>bikes_df.write.mode("overwrite").parquet('Files/product_data/bikes.parquet')
</code></pre>
<p><em>Partitioning the df to parquet:</em></p>
<pre><code>bikes_df.write.partitionBy("Category").mode("overwrite").parquet("Files/bike_data")
</code></pre>
<blockquote>
<p><strong>Note:</strong> Partitioning is an optimization technique that enables Spark to<br>
maximize performance across the worker nodes. More performance gains<br>
can be achieved when filtering data in queries by eliminating<br>
unnecessary disk IO.</p>
</blockquote>
<p><em>Load partitioned data:</em></p>
<pre><code>road_bikes_df = spark.read.parquet('Files/bike_data/Category=Road Bikes')
display(road_bikes_df.limit(5))
</code></pre>
<blockquote>
<p><strong>Note:</strong> The partitioning columns specified in the file path are omitted<br>
in the resulting dataframe. The results produced by the example query<br>
would not include a <strong>Category</strong> column - the category for all rows<br>
would be <em>Road Bikes</em>.</p>
</blockquote>
<h3 id="spark-sql">Spark SQL</h3>
<p>The Dataframe API is part of a Spark library named Spark SQL, which enables data analysts to use SQL expressions to query and manipulate data.</p>
<h4 id="creating-database-objects-in-the-spark-catalog">Creating database objects in the Spark catalog</h4>
<p>The Spark catalog is a metastore for relational data objects such as views and tables. The Spark runtime can use the catalog to seamlessly integrate code written in any Spark-supported language with SQL expressions that may be more natural to some data analysts or developers.</p>
<p>One of the simplest ways to make data in a dataframe available for querying in the Spark catalog is to create a temporary view, as shown in the following code example:</p>
<pre><code>df.createOrReplaceTempView("products_view")
</code></pre>
<p>A <em>view</em> is temporary, meaning that it’s automatically deleted at the end of the current session. You can also create <em>tables</em> that are persisted in the catalog to define a database that can be queried using Spark SQL.</p>
<p>Tables are metadata structures that store their underlying data in the storage location associated with the catalog. In Microsoft Fabric, data for <em>managed</em> tables is stored in the <strong>Tables</strong> storage location shown in your data lake, and any tables created using Spark are listed there.</p>
<p>You can create an empty table by using the  <code>spark.catalog.createTable</code>  method, or you can save a dataframe as a table by using its  <code>saveAsTable</code>  method. Deleting a managed table also deletes its underlying data.</p>
<p>For example, the following code saves a dataframe as a new table named  <strong>products</strong>:</p>
<pre><code>df.write.format("delta").saveAsTable("products")
</code></pre>
<blockquote>
<p><strong>Note:</strong> The Spark catalog supports tables based on files in various<br>
formats. The preferred format in Microsoft Fabric is  <strong>delta</strong>, which<br>
is the format for a relational data technology on Spark named  <em>Delta<br>
Lake</em>. Delta tables support features commonly found in relational<br>
database systems, including transactions, versioning, and support for<br>
streaming data.</p>
</blockquote>
<p>Additionally, you can create <em>external</em> tables by using the <code>spark.catalog.createExternalTable</code> method. External tables define metadata in the catalog but get their underlying data from an external storage location; typically a folder in the <strong>Files</strong> storage area of a lakehouse. Deleting an external table doesn’t delete the underlying data.</p>
<h4 id="using-spark-sql-api-to-query-data">Using Spark SQL API to query data</h4>
<pre><code>bikes_df = spark.sql("SELECT ProductID, ProductName, ListPrice \
                      FROM products \
                      WHERE Category IN ('Mountain Bikes', 'Road Bikes')")
display(bikes_df)
</code></pre>
<h4 id="using-sql-code">Using SQL code</h4>
<pre><code>%%sql

SELECT Category, COUNT(ProductID) AS ProductCount
FROM products
GROUP BY Category
ORDER BY Category
</code></pre>
<h3 id="visualize-data-in-a-spark-notebook">Visualize data in a Spark notebook</h3>
<h4 id="using-built-in-notebook-charts">Using built-in notebook charts</h4>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/notebook-chart.png" alt="notebook-visualization"></p>
<h4 id="using-graphics-packages-in-code">Using graphics packages in code</h4>
<pre><code>from matplotlib import pyplot as plt

# Get the data as a Pandas dataframe
data = spark.sql("SELECT Category, COUNT(ProductID) AS ProductCount \
                  FROM products \
                  GROUP BY Category \
                  ORDER BY Category").toPandas()

# Clear the plot area
plt.clf()

# Create a Figure
fig = plt.figure(figsize=(12,8))

# Create a bar plot of product counts by category
plt.bar(x=data['Category'], height=data['ProductCount'], color='orange')

# Customize the chart
plt.title('Product Counts by Category')
plt.xlabel('Category')
plt.ylabel('Products')
plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
plt.xticks(rotation=70)

# Show the plot area
plt.show()
</code></pre>
<h2 id="ingest-data--real-time-data-using-eventhouse">Ingest Data : Real-time data using Eventhouse</h2>
<p>An Eventhouse in Microsoft Fabric provides a data store for large volumes of data. It’s a container that houses one or more KQL databases, each optimized for storing and analyzing real-time data that arrives continuously from various sources.</p>
<p>You can load data into a KQL database in an Eventhouse using an Eventstream or you can directly ingest data into a KQL database. Once you have ingested data, you can then:</p>
<ul>
<li>Query the data using Kusto Query Language (KQL) or T-SQL in a KQL queryset.</li>
<li>Use Real-Time Dashboards to visualize the data.</li>
<li>Use Fabric Activator to automate actions based on the data.</li>
</ul>
<h3 id="how-do-kql-databases-work-with-real-time-data">How do KQL databases work with real-time data?</h3>
<ul>
<li>KQL databases automatically partition data by ingestion time, making<br>
recent data quickly accessible while storing historical data for<br>
trend analysis.
<ul>
<li><em>Partitioning</em> means the database organizes data into separate storage locations based on when it arrived, so when you query for<br>
recent data, the database knows exactly where to search rather than<br>
scanning all the data.</li>
<li>Think of it like a digital conveyor belt - events flow in continuously, get organized automatically by when they arrive, and are immediately available for analysis while the stream keeps flowing.</li>
</ul>
</li>
</ul>
<h3 id="eventhouse">Eventhouse</h3>
<p>An Eventhouse contains one or more KQL databases.</p>
<ul>
<li>You can create tables, stored procedures, materialized views, functions, data streams, and shortcuts to manage your data.</li>
<li>You can ingest data directly into your KQL database from various sources:
<ul>
<li>Local files, Azure storage, Amazon S3</li>
<li>Azure Event Hubs, Fabric Eventstream, Real-Time hub</li>
<li>OneLake, Data Factory copy, Dataflows</li>
<li>Connectors to sources such as Apache Kafka, Confluent Cloud Kafka, Apache Flink, MQTT (Message Queuing Telemetry Transport), Amazon Kinesis, Google Cloud Pub/Sub</li>
</ul>
</li>
</ul>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/query-data-kql-database-microsoft-fabric/media/get-data.png#lightbox" alt="kql-01"></p>
<ul>
<li>Database shortcuts
<ul>
<li>You can create  <strong>database shortcuts</strong>  to existing KQL databases in other eventhouses or Azure Data Explorer databases.</li>
<li>These shortcuts let you query data from external KQL databases as if the data were stored locally in your eventhouse, without actually copying the data.</li>
</ul>
</li>
<li>OneLake availability
<ul>
<li>You can enable  <strong>OneLake availability</strong>  for individual KQL databases or tables, making your data accessible throughout the Fabric ecosystem for cross-workload integration with Power BI, Warehouse, Lakehouse, and other Fabric services.</li>
</ul>
</li>
</ul>
<h3 id="query-data-in-a-kql-database">Query data in a KQL database</h3>
<p>KQL uses a pipeline approach where data flows from one operation to the next using the pipe (<code>|</code>) character. Think of it like a funnel - you start with an entire data table, and each operator filters, rearranges, or summarizes the data before passing it to the next step. The order of operators matters because each step works on the results from the previous step.</p>
<blockquote>
<p>Important : KQL is case-sensitive.</p>
</blockquote>
<h4 id="kql-query-optimization">KQL query optimization</h4>
<p>The key principle is: the less data your query needs to process, the faster it runs.</p>
<ul>
<li>Filter data early and effectively
<ul>
<li><em>Time-based filtering</em></li>
<li><em>Order your filters by how much data they eliminate</em></li>
</ul>
</li>
<li>Reduce columns early</li>
<li>Optimize aggregations and joins
<ul>
<li><em>For aggregations, limit results when exploring data</em></li>
<li><em>For joins, put the smaller table first</em></li>
</ul>
</li>
</ul>
<h4 id="materialized-views">Materialized views</h4>
<p>Materialized views are precomputed aggregations that solve a common performance challenge in KQL databases.</p>
<p>Materialized views store precomputed aggregation results and automatically update them as new data arrives.</p>
<p>The materialized view only processes the new data to update the aggregations.</p>
<p>A materialized view consists of two parts that work together to provide always-current results:</p>
<ul>
<li><strong>A materialized part</strong>: Precomputed aggregation results from data that has already been processed</li>
<li><strong>A delta</strong>: New data that has arrived since the last background update</li>
</ul>
<p>When you query a materialized view, the system automatically combines both parts at query time to give you fresh, up-to-date results. Regardless of when the background materialization process last ran.</p>
<p>Meanwhile, a background process periodically moves data from the delta part into the materialized part, keeping the precomputed results current.</p>
<p><em>Create materialized views</em></p>
<pre><code>.create materialized-view TripsByVendor on table TaxiTrips
{
    TaxiTrips
    | summarize trips = count(), avg_fare = avg(fare_amount), total_revenue = sum(fare_amount)
    by vendor_id, pickup_date = format_datetime(pickup_datetime, "yyyy-MM-dd")
}
</code></pre>
<p><em>Query materialized views</em></p>
<pre><code>TripsByVendor
| where pickup_date &gt;= ago(7d)
| project pickup_date, vendor_id, trips, avg_fare, total_revenue
| sort by pickup_date desc, total_revenue desc
</code></pre>
<h4 id="stored-functions">Stored functions</h4>
<p>Stored functions are useful in eventhouses where you have streaming data and multiple people writing queries.<br>
Instead of writing the same filtering or transformation logic repeatedly, you can define it once as a function and reuse it across different queries.<br>
Functions also help ensure that calculations are performed consistently when different team members need to apply the same logic to the data.</p>
<p><em>Create a function</em></p>
<pre><code>.create-or-alter function trips_by_min_passenger_count(num_passengers:long)
{
    TaxiTrips
    | where passenger_count &gt;= num_passengers 
    | project trip_id, pickup_datetime
}
</code></pre>
<p><em>Calling the function</em></p>
<pre><code>trips_by_min_passenger_count(3)
| take 10
</code></pre>

