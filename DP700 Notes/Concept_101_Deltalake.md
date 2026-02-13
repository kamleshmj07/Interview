---


---

<h2 id="microsoft-fabric-delta-lake">Microsoft Fabric Delta Lake</h2>
<p>Tables in a Microsoft Fabric lakehouse are based on the Linux foundation <em>Delta Lake</em> table format, commonly used in Apache Spark.<br>
Delta Lake is an open-source storage layer for Spark that enables relational database capabilities for batch and streaming data. By using Delta Lake, you can implement a lakehouse architecture to support SQL-based data manipulation semantics in Spark with support for transactions and schema enforcement. The result is an analytical data store that offers many of the advantages of a relational database system with the flexibility of data file storage in a data lake.</p>
<p>Delta tables are schema abstractions over data files that are stored in Delta format. For each table, the lakehouse stores a folder containing <em>Parquet</em> data files and a <strong>_delta_Log</strong> folder in which transaction details are logged in JSON format.</p>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/work-delta-lake-tables-fabric/media/delta-files.png" alt="Screenshot of the files view of the parquet files in the salesorders table viewed through Lakehouse explorer."></p>
<p>The benefits of using Delta tables include:</p>
<ul>
<li>
<p><strong>Relational tables that support querying and data modification</strong>. With Apache Spark, you can store data in Delta tables that support  <em>CRUD</em>  (create, read, update, and delete) operations.</p>
</li>
<li>
<p><strong>Support for  <em>ACID</em>  transactions</strong>. Delta Lake brings <strong><em>ACID</em></strong> transactional support to Spark by implementing a transaction log and enforcing serializable isolation for concurrent operations.</p>
</li>
<li>
<p><strong>Data versioning and  <em>time travel</em></strong>. Because all transactions are logged in the transaction log, you can track multiple versions of each table row and even use the  <em>time travel</em>  feature to retrieve a previous version of a row in a query.</p>
</li>
<li>
<p><strong>Support for batch and streaming data</strong>. Spark includes native support for streaming data through the Spark Structured Streaming API. Delta Lake tables can be used as both  <em>sinks</em>  (destinations) and  <em>sources</em>  for streaming data.</p>
</li>
<li>
<p><strong>Standard formats and interoperability</strong>. The underlying data for Delta tables is stored in Parquet format, which is commonly used in data lake ingestion pipelines. Additionally, you can use the SQL analytics endpoint for the Microsoft Fabric lakehouse to query Delta tables in SQL.</p>
</li>
</ul>
<h3 id="creating-a-delta-table-from-a-dataframe">Creating a delta table from a dataframe</h3>
<pre><code># Load a file into a dataframe
df = spark.read.load('Files/mydata.csv', format='csv', header=True)

# Save the dataframe as a delta table
df.write.format("delta").saveAsTable("mytable")
</code></pre>
<h3 id="managed--vs--external--tables"><em>Managed</em>  vs  <em>external</em>  tables</h3>
<p>In the previous example, the dataframe was saved as a  <em>managed</em>  table; meaning that the table definition in the metastore and the underlying data files are both managed by the Spark runtime for the Fabric lakehouse. Deleting the table will also delete the underlying files from the  <strong>Tables</strong>  storage location for the lakehouse.</p>
<p>You can also create tables as  <em>external</em>  tables, in which the relational table definition in the metastore is mapped to an alternative file storage location. For example, the following code creates an external table for which the data is stored in the folder in the  <strong>Files</strong>  storage location for the lakehouse:</p>
<pre><code>df.write.format("delta").saveAsTable("myexternaltable", path="Files/myexternaltable")
</code></pre>
<p>In this example, the table definition is created in the metastore (so the table is listed in the  <strong>Tables</strong>  user interface for the lakehouse), but the Parquet data files and JSON log files for the table are stored in the  <strong>Files</strong>  storage location (and will be shown in the  <strong>Files</strong>  node in the  <strong>Lakehouse explorer</strong>  pane).</p>
<p>You can also specify a fully qualified path for a storage location, like this:</p>
<pre><code>df.write.format("delta").saveAsTable("myexternaltable", path="abfss://my_store_url..../myexternaltable")
</code></pre>
<hr>
<h3 id="creating-table-metadata">Creating table metadata</h3>
<p>While it’s common to create a table from existing data in a dataframe, there are often scenarios where you want to create a table definition in the metastore that will be populated with data in other ways. There are multiple ways you can accomplish this goal.</p>
<h4 id="use-the--deltatablebuilder--api">Use the  <em>DeltaTableBuilder</em>  API</h4>
<p>The  <strong>DeltaTableBuilder</strong>  API enables you to write Spark code to create a table based on your specifications. For example, the following code creates a table with a specified name and columns.</p>
<pre><code>from delta.tables import *

DeltaTable.create(spark) \
  .tableName("products") \
  .addColumn("Productid", "INT") \
  .addColumn("ProductName", "STRING") \
  .addColumn("Category", "STRING") \
  .addColumn("Price", "FLOAT") \
  .execute()
</code></pre>
<h4 id="use-spark-sql">Use Spark SQL</h4>
<p>Managed table</p>
<pre><code>%%sql

CREATE TABLE salesorders
(
    Orderid INT NOT NULL,
    OrderDate TIMESTAMP NOT NULL,
    CustomerName STRING,
    SalesTotal FLOAT NOT NULL
)
USING DELTA
</code></pre>
<p>External table</p>
<pre><code>%%sql

CREATE TABLE MyExternalTable
USING DELTA
LOCATION 'Files/mydata'
</code></pre>
<p>Save table</p>
<pre><code>delta_path = "Files/mydatatable"
df.write.format("delta").save(delta_path)
</code></pre>
<p>For overwriting</p>
<p><code>new_df.write.format("delta").mode("overwrite").save(delta_path)</code></p>
<p>For appending</p>
<p><code>new_rows_df.write.format("delta").mode("append").save(delta_path)</code></p>
<h3 id="optimize-delta-tables">Optimize delta tables</h3>

