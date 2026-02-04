
## Ingest Data : Dataflows Gen2

 - *Dataflows Gen2*  are used to ingest and transform data from multiple sources, and then land the cleansed data to another destination.
 - They can be incorporated into data pipelines for more complex activity orchestration, and also used as a data source in Power BI.
 - *Dataflows Gen2* allow you to prepare the data to ensure consistency, and then stage the data in the preferred destination.
 - Dataflows can be horizontally partitioned as well. Once you create a global dataflow, data analysts can use dataflows to create specialized semantic models for specific needs.
 - In Microsoft Fabric, you can create a *Dataflow Gen2* in the Data Factory workload or Power BI workspace, or directly in the lakehouse.
 - You can also load your dataflow to Azure SQL database, Azure Data Explorer, or Azure Synapse Analytics.

Dataflows Gen2 provide a low-to-no-code solution to ingest, transform, and load data into your Fabric data stores.

### Benefits:
-   Extend data with consistent data, such as a standard date dimension table.
-   Allow self-service users access to a subset of data warehouse separately.
-   Optimize performance with dataflows, which enable extracting data once for reuse, reducing data refresh time for slower sources.
-   Simplify data source complexity by only exposing dataflows to larger analyst groups.
-   Ensure consistency and quality of data by enabling users to clean and transform data before loading it to a destination.
-   Simplify data integration by providing a low-code interface that ingests data from various sources.

### Limitations:
-   Dataflows aren't a replacement for a data warehouse.
-   Row-level security isn't supported.
-   Fabric capacity workspace is required.

### Dataflows Gen2 and Pipelines
Data pipelines are a common concept in data engineering and offer a wide variety of activities to orchestrate. Some common activities include:

-   Copy data
-   Incorporate Dataflow
-   Add Notebook
-   Get metadata
-   Execute a script or stored procedure


## Ingest Data : Understand pipelines
Pipelines in Microsoft Fabric encapsulate a sequence of _activities_ that perform data movement and processing tasks. 

You can use a pipeline to define data transfer and transformation activities, and orchestrate these activities through control flow activities that manage branching, looping, and other typical processing logic.

### Core pipeline concepts

#### Activities
Activities are the executable tasks in a pipeline. You can define a flow of activities by connecting them in a sequence. 

The outcome of a particular activity (success, failure, or completion) can be used to direct the flow to the next activity in the sequence.

There are two broad categories of activity in a pipeline.

-   Data transformation activities  - data transfer operations, like *Copy Data* , and more complex  *Data Flow*  activities that encapsulate dataflows (Gen2). Other data transformation activities include  *Notebook*  activities to run a Spark notebook,  *Stored procedure*  activities to run SQL code,  *Delete data*  activities to delete existing data, and others.
    
-   Control flow activities  - activities that you can use to implement loops, conditional branching, or manage variable and parameter values. The wide range of control flow activities enables you to implement complex pipeline logic to orchestrate data ingestion and transformation flow.

#### Parameters

Pipelines can be parameterized, enabling you to provide specific values to be used each time a pipeline is run. For example, you might want to use a pipeline to save ingested data in a folder, but have the flexibility to specify a folder name each time the pipeline is run.

Using parameters increases the reusability of your pipelines, enabling you to create flexible data ingestion and transformation processes.

#### Pipeline runs

Each time a pipeline is executed, a  _data pipeline run_  is initiated. Runs can be initiated on-demand in the Fabric user interface or scheduled to start at a specific frequency. Use the unique run ID to review run details to confirm they completed successfully and investigate the specific settings used for each execution.

#### Pipeline templates
You can define pipelines from any combination of activities you choose, enabling to create custom data ingestion and transformation processes to meet your specific needs. However, there are many common pipeline scenarios for which Microsoft Fabric includes predefined pipeline templates that you can use and customize as required.

To create a pipeline based on a template, select the  *Templates*  tile in a new pipeline.

### Run and monitor pipelines
When you have completed a pipeline, you can use the *Validate* option to check that the configuration is valid, and then either run it interactively or specify a schedule.
You can view the **run history** for a pipeline to see details of each run, either from the pipeline canvas or from the pipeline item listed in the page for the workspace.

## Ingest Data : Use Apache Spark
Apache Spark is an open source parallel processing framework for large-scale data processing and analytics.
Spark is popular in "big data" processing scenarios, and is available in multiple platform implementations; including *Azure HDInsight*, *Azure Synapse Analytics*, and *Microsoft Fabric*.

Apache Spark is a distributed data processing framework that enables large-scale data analytics by coordinating work across multiple processing nodes in a cluster, known in Microsoft Fabric as a  _Spark pool_.
Put more simply, Spark uses a "divide and conquer" approach to processing large volumes of data quickly by distributing the work across multiple computers. The process of distributing tasks and collating results is handled for you by Spark.

Spark can run code in Java, Scala (a Java-based scripting language), Spark R, Spark SQL, and PySpark (a Spark-specific variant of Python).

In practice, most data engineering and analytics workloads are accomplished using a combination of PySpark and Spark SQL.

### Spark pools
A Spark pool consists of compute  _nodes_  that distribute data processing tasks.

![spark-pool-1](https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-pool.png)

As shown in the diagram, a Spark pool contains two kinds of node:

1.  A  _head_  node in a Spark pool coordinates distributed processes through a  _driver_  program.
2.  The pool includes multiple  _worker_  nodes on which  _executor_  processes perform the actual data processing tasks.

The Spark pool uses this distributed compute architecture to access and process data in a compatible data store - such as a data lakehouse based in OneLake.

### Spark pools in Microsoft Fabric

Microsoft Fabric provides a  _starter pool_  in each workspace, enabling Spark jobs to be started and run quickly with minimal setup and configuration. You can configure the starter pool to optimize the nodes it contains in accordance with your specific workload needs or cost constraints.

Additionally, you can create custom Spark pools with specific node configurations that support your particular data processing needs.

Learn More : 
[Capacity Administration](https://learn.microsoft.com/en-us/fabric/data-engineering/capacity-settings-overview)

You can manage settings for the starter pool and create new Spark pools in the **Admin portal** section of the workspace settings, under **Capacity settings**, then **Data Engineering/Science Settings.**

![admin-portal-spark](https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-settings.png)

Specific configuration settings for Spark pools include:

-   **Node Family**: The type of virtual machines used for the Spark cluster nodes. In most cases,  _memory optimized_  nodes provide optimal performance.
-   **Autoscale**: Whether or not to automatically provision nodes as needed, and if so, the initial and maximum number of nodes to be allocated to the pool.
-   **Dynamic allocation**: Whether or not to dynamically allocate  _executor_  processes on the worker nodes based on data volumes.

Learn More:
[Configure starter pool](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-starter-pools)
[Create custom spark pool](https://learn.microsoft.com/en-us/fabric/data-engineering/create-custom-spark-pools)


### Runtimes and environments
The Spark open source ecosystem includes multiple versions of the Spark  _runtime_, which determines the version of Apache Spark, Delta Lake, Python, and other core software components that are installed. Additionally, within a runtime you can install and use a wide selection of code libraries for common (and sometimes very specialized) tasks.

In some cases, organizations may need to define multiple _environments_ to support a diverse range of data processing tasks. Each environment defines a specific runtime version as well as the libraries that must be installed to perform specific operations. Data engineers and scientists can then select which environment they want to use with a Spark pool for a particular task.

Learn More:
[Apache Spark Runtimes in Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/runtime)

### Environments in MS Fabric
You can create custom environments in a Fabric workspace, enabling you to use specific Spark runtimes, libraries, and configuration settings for different data processing operations.
![spark-env-01](https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-environment.png)

When creating an environment, you can:

-   Specify the Spark runtime it should use.
-   View the built-in libraries that are installed in every environment.
-   Install specific public libraries from the Python Package Index (PyPI).
-   Install custom libraries by uploading a package file.
-   Specify the Spark pool that the environment should use.
-   Specify Spark configuration properties to override default behavior.
-   Upload resource files that need to be available in the environment.

Learn More:
[Create, configure, and use an environment in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/create-and-use-environment)

### Additional Spark configuration options
#### Native execution engine

The  _native execution engine_  in Microsoft Fabric is a vectorized processing engine that runs Spark operations directly on lakehouse infrastructure. 

Using the native execution engine can significantly improve the performance of queries when working with large data sets in Parquet or Delta file formats.

To use the native execution engine, you can enable it at the environment level or within an individual notebook.

To enable the native execution engine at the environment level, set the following Spark properties in the environment configuration:

-   **spark.native.enabled**: true
-   **spark.shuffle.manager**: org.apache.spark.shuffle.sort.ColumnarShuffleManager

To enable the native execution engine for a specific script or notebook, you can set these configuration properties at the beginning of your code, like this:

    %%configure 
    { 
       "conf": {
           "spark.native.enabled": "true", 
           "spark.shuffle.manager": "org.apache.spark.shuffle.sort.ColumnarShuffleManager" 
       } 
    }

Learn More:
[Native execution engine for Fabric Spark](https://learn.microsoft.com/en-us/fabric/data-engineering/native-execution-engine-overview)

#### High concurrency mode

When you run Spark code in Microsoft Fabric, a Spark session is initiated. You can optimize the efficiency of Spark resource usage by using  _high concurrency mode_  to share Spark sessions across multiple concurrent users or processes. 

A notebook uses a Spark session for its execution. When high concurrency mode is enabled, multiple users can, for example, run code in notebooks that use the same Spark session, while ensuring isolation of code to avoid variables in one notebook being affected by code in another notebook.

You can also enable high concurrency mode for Spark jobs, enabling similar efficiencies for concurrent non-interactive Spark script execution.

Learn More:
[High concurrency mode in Apache Spark for Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/high-concurrency-overview)

#### Automatic MLFlow logging

MLFlow is an open source library that is used in data science workloads to manage machine learning training and model deployment. A key capability of MLFlow is the ability to log model training and management operations. By default, Microsoft Fabric uses MLFlow to implicitly log machine learning experiment activity without requiring the data scientist to include explicit code to do so. You can disable this functionality in the workspace settings.

### Spark in Notebook using Pyspark
![spark-in-notebook](https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/notebook.png)

### Spark job definition
![spark-job-def](https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/spark-job.png)

### Spark dataframe in Pyspark
Natively, Spark uses a data structure called a _resilient distributed dataset_ (RDD); but while you _can_ write code that works directly with RDDs, the most commonly used data structure for working with structured data in Spark is the _dataframe_, which is provided as part of the _Spark SQL_ library. Dataframes in Spark are similar to those in the ubiquitous _Pandas_ Python library, but optimized to work in Spark's distributed processing environment.

> **Note**: In addition to the Dataframe API, Spark SQL provides a
> strongly-typed  _Dataset_  API that is supported in Java and Scala.
> We'll focus on the Dataframe API in this module.

#### Inferring a schema
Pyspark

    %%pyspark
    df = spark.read.load('Files/data/products.csv',
        format='csv',
        header=True
    )
    display(df.limit(10))

Scala

    %%spark
    val df = spark.read.format("csv").option("header", "true").load("Files/data/products.csv")
    display(df.limit(10))

#### Explicit schema
Pyspark

    from pyspark.sql.types import *
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


> **Tip:** Specifying an explicit schema also improves performance!


#### Filtering and grouping
*Selecting columns:*

    pricelist_df = df.select("ProductID", "ListPrice")
or

    pricelist_df = df["ProductID", "ListPrice"]

*Chaining functions:*

    bikes_df = df.select("ProductName", "Category", "ListPrice").where((df["Category"]=="Mountain Bikes") | (df["Category"]=="Road Bikes"))
    display(bikes_df)

*Group by:*

    counts_df = df.select("ProductID", "Category").groupBy("Category").count()
    display(counts_df)

*Saving df to parquet:*

    bikes_df.write.mode("overwrite").parquet('Files/product_data/bikes.parquet')

*Partitioning the df to parquet:*

    bikes_df.write.partitionBy("Category").mode("overwrite").parquet("Files/bike_data")

> **Note:** Partitioning is an optimization technique that enables Spark to
> maximize performance across the worker nodes. More performance gains
> can be achieved when filtering data in queries by eliminating
> unnecessary disk IO.

*Load partitioned data:*

    road_bikes_df = spark.read.parquet('Files/bike_data/Category=Road Bikes')
    display(road_bikes_df.limit(5))

> **Note:** The partitioning columns specified in the file path are omitted
> in the resulting dataframe. The results produced by the example query
> would not include a **Category** column - the category for all rows
> would be _Road Bikes_.

### Spark SQL
The Dataframe API is part of a Spark library named Spark SQL, which enables data analysts to use SQL expressions to query and manipulate data.

#### Creating database objects in the Spark catalog
The Spark catalog is a metastore for relational data objects such as views and tables. The Spark runtime can use the catalog to seamlessly integrate code written in any Spark-supported language with SQL expressions that may be more natural to some data analysts or developers.

One of the simplest ways to make data in a dataframe available for querying in the Spark catalog is to create a temporary view, as shown in the following code example:

    df.createOrReplaceTempView("products_view")

A _view_ is temporary, meaning that it's automatically deleted at the end of the current session. You can also create _tables_ that are persisted in the catalog to define a database that can be queried using Spark SQL.

Tables are metadata structures that store their underlying data in the storage location associated with the catalog. In Microsoft Fabric, data for _managed_ tables is stored in the **Tables** storage location shown in your data lake, and any tables created using Spark are listed there.

You can create an empty table by using the  `spark.catalog.createTable`  method, or you can save a dataframe as a table by using its  `saveAsTable`  method. Deleting a managed table also deletes its underlying data.

For example, the following code saves a dataframe as a new table named  **products**:

    df.write.format("delta").saveAsTable("products")

> **Note:** The Spark catalog supports tables based on files in various
> formats. The preferred format in Microsoft Fabric is  **delta**, which
> is the format for a relational data technology on Spark named  _Delta
> Lake_. Delta tables support features commonly found in relational
> database systems, including transactions, versioning, and support for
> streaming data.

Additionally, you can create _external_ tables by using the `spark.catalog.createExternalTable` method. External tables define metadata in the catalog but get their underlying data from an external storage location; typically a folder in the **Files** storage area of a lakehouse. Deleting an external table doesn't delete the underlying data.

#### Using Spark SQL API to query data

    bikes_df = spark.sql("SELECT ProductID, ProductName, ListPrice \
                          FROM products \
                          WHERE Category IN ('Mountain Bikes', 'Road Bikes')")
    display(bikes_df)

#### Using SQL code


    %%sql
    
    SELECT Category, COUNT(ProductID) AS ProductCount
    FROM products
    GROUP BY Category
    ORDER BY Category

### Visualize data in a Spark notebook

#### Using built-in notebook charts
![notebook-visualization](https://learn.microsoft.com/en-in/training/wwl/use-apache-spark-work-files-lakehouse/media/notebook-chart.png)

#### Using graphics packages in code

    from matplotlib import pyplot as plt
    
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


