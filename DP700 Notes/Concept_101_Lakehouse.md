---


---

<h2 id="microsoft-fabric-lakehouse">Microsoft Fabric lakehouse</h2>
<p>A <strong>lakehouse</strong> presents as a database and is built on top of a data lake using Delta format tables.<br>
Lakehouses combine the SQL-based analytical capabilities of a relational data warehouse and the flexibility and scalability of a data lake.<br>
Lakehouses store all data formats and can be used with various analytics tools and programming languages.<br>
As cloud-based solutions, lakehouses can scale automatically and provide high availability and disaster recovery.</p>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/get-started-lakehouses/media/lakehouse-components.png" alt="Diagram of a lakehouse, displaying the folder structure of a data lake and the relational capabilities of a data warehouse."></p>
<p><strong>Some benefits of a lakehouse include:</strong></p>
<ul>
<li>Lakehouses use Spark and SQL engines to process large-scale data and support machine learning or predictive modeling analytics.</li>
<li>Lakehouse data is organized in a  <em>schema-on-read format</em>, which means you define the schema as needed rather than having a predefined schema.</li>
<li>Lakehouses support ACID (Atomicity, Consistency, Isolation, Durability) transactions through Delta Lake formatted tables for data consistency and integrity.</li>
</ul>
<hr>
<h3 id="secure-a-lakehouse">Secure a lakehouse</h3>
<p>Lakehouse access is managed either through the workspace or item-level sharing.<br>
Workspaces roles should be used for collaborators because these roles grant access to all items within the workspace.<br>
Item-level sharing is best used for granting access for read-only needs, such as analytics or Power BI report development.</p>
<p>Fabric lakehouses also support data governance features including sensitivity labels, and can be extended by using Microsoft Purview with your Fabric tenant.</p>
<blockquote>
<p>Note: For more information, see the  <a href="https://learn.microsoft.com/en-us/fabric/security/security-overview">Security in Microsoft<br>
Fabric</a> documentation.</p>
</blockquote>
<h4 id="authenticate">Authenticate</h4>
<p>Microsoft Fabric is a SaaS platform, like many other Microsoft services such as Azure, Microsoft Office, OneDrive, and Dynamics. All these Microsoft SaaS services including Fabric, use  <a href="https://learn.microsoft.com/en-us/entra/verified-id/decentralized-identifier-overview">Microsoft Entra ID</a>  as their cloud-based identity provider. Microsoft Entra ID helps users connect to these services quickly and easily from any device and any network.</p>
<hr>
<h3 id="explore-lakehouse">Explore lakehouse</h3>
<ul>
<li>The  <strong>lakehouse</strong>  contains shortcuts, folders, files, and tables.</li>
<li>The  <strong>Semantic model (default)</strong>  provides an easy data source for Power BI report developers.</li>
<li>The  <strong>SQL analytics endpoint</strong>  allows read-only access to query data with SQL.</li>
</ul>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/get-started-lakehouses/media/lakehouse-items.png" alt="Screenshot of the three Lakehouse items as described."></p>
<p>Ingesting data into your lakehouse is the first step in your ETL process. Use any of the following methods to bring data into your lakehouse.</p>
<ul>
<li><strong>Upload</strong>: Upload local files.</li>
<li><strong>Dataflows Gen2</strong>: Import and transform data using Power Query.</li>
<li><strong>Notebooks</strong>: Use Apache Spark to ingest, transform, and load data.</li>
<li><strong>Data Factory pipelines</strong>: Use the Copy data activity.</li>
</ul>
<p><strong>Spark job definitions</strong> can also be used to submit batch/streaming jobs to Spark clusters. By uploading the binary files from the compilation output of different languages (for example, .jar from Java), you can apply different transformation logic to the data hosted on a lakehouse. Besides the binary file, you can further customize the behavior of the job by uploading more libraries and command line arguments.</p>
<hr>
<h3 id="access-data-using-shortcuts">Access data using shortcuts</h3>
<p>Another way to access and use data in Fabric is to use  <em>shortcuts</em>. Shortcuts enable you to integrate data into your lakehouse while keeping it stored in external storage.</p>
<p>Shortcuts are useful when you need to source data that’s in a different storage account or even a different cloud provider.</p>
<p>Shortcuts can be created in both lakehouses and KQL databases, and appear as a folder in the lake. This allows Spark, SQL, Real-Time intelligence and Analysis Services to all utilize shortcuts when querying data.</p>
<hr>
<h3 id="transform-and-load-data">Transform and load data</h3>
<p>Regardless of your ETL design, you can transform and load data simply using the same tools to ingest data. Transformed data can then be loaded as a file or a Delta table.</p>
<ul>
<li><em>Notebooks</em> are favored by data engineers familiar with different programming languages including PySpark, SQL, and Scala.</li>
<li><em>Dataflows Gen2</em> are excellent for developers familiar with Power BI or Excel since they use the Power Query interface.</li>
<li><em>Pipelines</em> provide a visual interface to perform and orchestrate ETL processes. Pipelines can be as simple or as complex as you need.</li>
</ul>
<p>After data is ingested, transformed, and loaded, it’s ready for others to use.</p>
<ul>
<li>Data scientists can use notebooks or Data wrangler to explore and train machine learning models for AI.</li>
<li>Report developers can use the semantic model to create Power BI reports.</li>
<li>Analysts can use the SQL analytics endpoint to query, filter, aggregate, and otherwise explore data in lakehouse tables.</li>
</ul>

