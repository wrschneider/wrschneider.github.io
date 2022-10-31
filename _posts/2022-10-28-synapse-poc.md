I took a look at Azure Synapse and compared it against Azure Databricks.  

*I am sharing some personal observations from a brief POC, and this is not to be construed as a recommendation or endorsement for either product.*

At first glance, Azure Synapse and Databricks seem like analogous solutions in the context of big-data ecosystems.  There is mostly a one-to-one correspondence between their components:

* A "workspace" is the top-level context for both
* Workspaces have Spark runtimes - Spark pools in Synapse, Clusters in Databricks
* Workspaces have notebooks (based on Jupyter), which connect to Spark pools/clusters
* Workspaces have SQL endpoints - Serverless SQL pools in Synapse, SQL endpoints in Databricks
* Workspaces have a catalog that serves as the Hive metastore, with a lifecycle tied to the workspace rather than individual Spark pools/clusters.  This catalog is a "lake database" in Synapse.
* Both Databricks and Synapse support Delta Lake, although Synapse SQL Pool access to delta tables is read-only. (Synapse Spark pools support SQL DML operations.)
* Both support integrating with Git repositories for notebooks.

#### What I liked about Synapse

The main advantage I see of Synapse over Databricks is Synapse is more tightly integrated with the native Azure ecosystem, so infrastructure provisioning is simpler compared with Azure Databricks, with fewer moving parts.  

* Networking: In Synapse, there's no explicit vnet configuration or IP address ranges to manage, and the concept of connecting to ADLS2 over a private endpoint is just another box to check in the Azure portal.

  Similarly, egress control and exfiltration is built in, by limiting egress only to storage accounts via private endpoint.  In Databricks, by comparison, egress control must be supplied explicitly by vnet configuration.

* Storage authentication: Synapse gets access to ADLS2 via managed identity, rather than having to create a service principal, and then telling Databricks to use that service principal and secret for a particular ADLS2 location.  This distinction may fade now that [Unity catalog provides ability to use managed identity in Databricks as well.](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/azure-managed-identities)

* Orchestration: the capabilities of Azure Data Factory pipelines are natively available in Azure Synapse.  You can refer to Databricks jobs from Azure Data Factory or Synapse workflows too.

* Access control: Active Directory users and groups are automatically reflected in Synapse without a SCIM connector.

#### Challenges working with Synapse

* SQL pool connections are TCP connections over SQL Server port 1433. This makes it harder to connect clients through firewalls.  In this respect, working with SQL endpoints feels more like working with AWS Redshift than working with Athena or Databricks SQL endpoints over HTTPS port 443.

* Synapse has two different concepts of database catalog: [Lake Databases, and SQL Databases](https://learn.microsoft.com/en-us/azure/synapse-analytics/metadata/database).  
  The Lake Database is the Hive metastore, analogous to Databricks or AWS Glue catalog.  Spark pools can only access Lake databases; SQL pools can access both.  SQL databases can also define external tables backed by ADLS2, but theses are not visible from Spark pools.

  Synapse SQL pools feel similar to Redshift, in the sense that Redshift has both its own native catalog that can include both database-managed storage and external tables, and external tables inherited from a Glue catalog shared with Spark jobs.
  
  The two types of Synapse databases have different access control models.  
  * SQL Databases follow the SQL Server permissions model with logins, grants, roles, etc.  You can have
    either AD users or local SQL users.

  * Lake databases only work with Active Directory users.  Access control is performed at the storage layer and the docs specify "only Azure AD users can use tables in the lake databases".

  I tried connecting Power BI to Synapse, and found a few combinations that worked:
  * You can use an AAD user with both Lake databases and SQL databases. This works but may pose a [problem with automating deployments via REST APIs](https://community.powerbi.com/t5/Developer/Updating-OAuth-data-source-credentials-via-API-bearer-token/td-p/2028001).

  * You can use a SQL local username with SQL databases.  This works, and even allows you to use managed
    identity to access the underlying ADLS2 layer for external tables (with the right `CREATE CREDENTIAL` commands), but external ADLS2 tables in SQL databases will not be visible to Spark.