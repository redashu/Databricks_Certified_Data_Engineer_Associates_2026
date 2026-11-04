Unity Catalog – Enterprise Governance & Architecture
Slide 1 – Title
Unity Catalog: Enterprise Governance, Security & Data Architecture

Subtitle: Addressing enterprise requirements across governance, storage, sharing, monitoring, multi-region architecture and disaster recovery.

Topics:
- Governance and security
- Data sharing and storage
- CDC and replication
- Monitoring and operations
- Catalog and application architecture
- Regional DR and resiliency
Slide 2 – Customer Requirements: Executive Overview
The customer has 14 requirements across five architectural domains.
Domain	Customer expectation
Governance & Security	Centralized RLS/CLS, ABAC, tags, storage access
Data Sharing & Storage	Cross-region sharing, archival, CDC
Monitoring & Operations	Query monitoring, maintenance windows, dashboards
Catalog Architecture	Cross-catalog access and regional metastores
DR & Resiliency	Regional failover and metadata replication

Architecture principle

Unity Catalog is the centralized governance layer, but it does not automatically solve every operational, replication, networking, or disaster recovery requirement.

We need to combine native Databricks capabilities with cloud services and automation.
Slide 3 – Centralized RLS/CLS and ABAC at Metastore Level
Customer requirement: Can security policies be defined once and enforced across all catalogs?
What Databricks provides
- Attribute-Based Access Control (ABAC).
- Governed tags for classifying data objects.
- Centralized policies using tags and identity attributes.
- Row filters to restrict records.
- Column masks to protect sensitive columns.
- Policies can be attached at catalog, schema, table and metastore scope.
Current product status – November 2026
ABAC core – GA
 Metastore-level ABAC – Beta

Capability	Status
ABAC core	GA
Catalog/schema/table policies	GA, subject to feature-specific limitations
Metastore-level policies	Beta
ABAC DENY policies	Beta
ABAC policies on views	Beta

Metastore-level policies can cover matching objects across all catalogs, including future matching catalogs. 
Databricks on AWS+2

Example
A retailer has three catalogs:
- sales_prod
- finance_prod
- hr_prod
Requirement: Mask every column tagged PII for regular employees.

Define a centralized ABAC column-mask policy that matches the PII tag.

Exception: Approved HR administrators can be excluded through the policy's EXCEPT principal condition.
Customer recommendation
Use ABAC for centralized, tag-driven protection. If the customer requires GA-only capabilities, do not promise metastore-level ABAC yet.
Slide 4 – Policy Inheritance and Override Model
Customer requirement: How do permissions and policies flow through the Unity Catalog hierarchy?
Unity Catalog hierarchy
Account
   |
   +-- Metastore
          |
          +-- Catalog
                 |
                 +-- Schema
                        |
                        +-- Table
                               |
                               +-- Column


Privilege inheritance
- Catalog-level grants can provide access to descendant schemas and objects.
- Schema-level grants can provide access to child objects.
- Object-level grants can provide additional permissions.
- Workspace-catalog bindings provide another access restriction.
ABAC policy behavior
- Higher-scope policies apply to matching descendant objects.
- A metastore policy can cover matching tables across catalogs.
- Table owners cannot simply remove or bypass an applicable higher-scope ABAC policy.
- Multiple applicable row filters and masks are evaluated according to Databricks policy-composition and conflict rules.
Example
Central policy:

All columns tagged Confidential must be masked.

Local team requirement:

Finance analysts need access to selected confidential financial fields.

Solution: Design a governed exception for an approved identity group rather than allowing the local table owner to remove the central policy.

Key message: Central security guardrails should not depend on individual table owners following manual instructions. 
Databricks on Google Cloud+2

Slide 5 – Governed Tags and Enterprise Classification
Customer requirement: Centrally define tags and use them for classification, security and cost allocation.
What are governed tags?
Governed tags are centrally defined key-value attributes used to classify Unity Catalog objects.

Examples:
Tag	Value	Purpose
classification	Confidential	Data sensitivity
contains_pii	Yes	Privacy
business_domain	Finance	Ownership
region	India	Data residency
data_quality	Certified	Data trust

Governance capabilities
- Tag definitions are managed at account level.
- Allowed tag values can be controlled.
- Permissions distinguish tag creation, management and assignment.
- Tags can be applied to catalogs, schemas, tables, columns and other supported securables.
- ABAC policies can evaluate tags dynamically.
Important inheritance detail:
- Objects can inherit tags from parent catalogs and schemas.
- Inherited tags can be overridden at supported levels.
- Column tags do not automatically inherit from the parent table.
Cost allocation distinction
Governed tags on data assets are not the same as compute cost-allocation tags.

Use:
- Governed tags for data classification and ABAC.
- Compute/resource custom tags and system.billing.usage for cost attribution.

Databricks on Google Cloud+1
Slide 6 – ADLS Gen2: External Locations and Storage Credentials
Customer requirement: Centralized governance of Azure storage access.
Understand the three components
Component	Responsibility
Storage Credential	Defines authentication to cloud storage
External Location	Defines a governed storage path and associated credential
External Table	Registers external data as a Unity Catalog table

Recommended architecture
Unity Catalog
      |
      v
Storage Credential
(Azure Managed Identity)
      |
      v
External Location
(ADLS Gen2 Path)
      |
      +------ External Tables
      |
      +------ External Volumes


Recommended security controls
1. Prefer Azure managed identities through an Azure Databricks Access Connector.
2. Avoid long-lived storage keys where possible.
3. Scope external locations to specific containers or paths.
4. Restrict who can create storage credentials and external locations.
5. Use workspace bindings where required.
6. Audit storage access through Databricks audit logs and Azure diagnostics.
Lifecycle and credential rotation
- Managed identity reduces manual secret rotation.
- Azure role assignments and identity lifecycle still require management.
- Dropping an external table generally removes its registration, not its underlying data files.
- Storage lifecycle and data deletion should be managed separately.
Recommendation: Treat cloud IAM and Unity Catalog permissions as complementary security layers.
Slide 7 – Databricks Clean Rooms vs Delta Sharing
Customer requirement: Secure collaboration with external customers and partners.
What are Clean Rooms?
Databricks Clean Rooms provide a controlled environment where multiple organizations can collaborate on analysis without directly exposing each other's underlying raw data.
Example: Retailer and advertising partner
Retailer contributes:
- Transaction data
- Customer purchase patterns
Advertising partner contributes:
- Campaign data
- Advertising engagement
Both want to analyze campaign effectiveness without directly exchanging raw customer datasets.
Clean Rooms vs Delta Sharing
Feature	Delta Sharing	Clean Rooms
Share governed data	Yes	Yes, as collaboration inputs
Recipient independently queries shared data	Yes	Not the primary purpose
Multi-party analysis	Possible	Designed for it
Controlled analysis code	Not its central feature	Core capability
Raw-data exposure control	Sharing permissions	Isolated collaboration model

Security and monitoring
- Collaborators contribute selected data assets.
- Analysis assets are subject to approval controls.
- Raw data is not directly browsable by other collaborators.
- Clean Room events and audit activity can be monitored.
- Region selection is important because collaboration may involve cross-region data movement.
Recommendation: Use Delta Sharing for controlled data distribution; use Clean Rooms for constrained joint analysis. 
Databricks on AWS

Slide 8 – Cross-Region Delta Sharing
Customer requirement: Understand compute location, data location, network usage and cost.
Example
Provider: Mumbai region

Recipient: Singapore region
Provider Region                 Recipient Region

ADLS Storage  ----------------> Recipient Compute
                                   |
                                   v
                              SQL / Spark Query


Where does compute execute?
In the normal open Delta Sharing model, the recipient uses its own compute to query the shared data.

Depending on the sharing protocol and configuration, data can be read remotely from provider storage or materialized locally.
Cost considerations
Cost component	Explanation
Provider egress	Data transferred out of provider region
Network	Cross-region data transfer
Recipient compute	SQL warehouse or Spark compute
Storage	Local materialized copies
Requests	Cloud storage read/list operations

Monitoring
Monitor:
- Bytes transferred.
- Query execution time.
- Storage read operations.
- Network egress.
- Recipient compute utilization.
- Data refresh frequency.
Recommendation: For frequent, high-volume workloads, evaluate recipient-side materialization instead of repeatedly reading remote data.
Slide 9 – Historical Data Archival and Low-Cost Storage
Customer requirement: Move historical data to cheaper storage without changing the logical table definition.
Azure storage access tiers
Tier	Suitable use
Hot	Frequently accessed data
Cool	Infrequently accessed online data
Cold	Rarely accessed online data
Archive	Long-term offline archival

Important technical limitation
Delta tables rely on transaction logs and referenced data files.

Moving active Delta files to offline Archive storage can make queries fail because the files cannot be accessed normally.

Archive requires rehydration before the data can be read.
Recommended strategy
- Use Hot for active data.
- Use Cool or Cold for older data that must remain online.
- Use Archive for rarely accessed data with an explicit restore process.
- Consider a separate historical archive table or storage path.
- Validate Delta retention and transaction-log behavior before applying lifecycle policies.
Customer message: Transparent on-demand access to offline Archive storage should not be promised. 
Microsoft Learn+1

Slide 10 – Database Log-Based Replication and CDC
Customer requirement: Capture source database changes without repeatedly scanning full tables.
CDC architecture
Source Database
      |
      | Transaction Logs / Change Tracking
      v
CDC Connector
      |
      v
Databricks Lakeflow Connect
      |
      v
Bronze Delta Tables
      |
      v
Silver / Gold


Recommended mechanism
Evaluate:
- Lakeflow Connect for supported databases.
- Native database CDC.
- Database change tracking.
- Supported third-party CDC connectors.
The exact choice depends on the database engine, version, source capabilities and connector support.
Production database impact
Area	Potential impact
CPU	Change capture and processing
I/O	Reading transaction logs or change tables
Log growth	Depends on consumer lag and retention
Network	Continuous change transfer
Storage	CDC metadata and retained changes

Monitoring
- Replication lag.
- Last processed log position.
- Source log retention.
- Connector health.
- Checkpoint recovery.
- Failed records and retries.
Recommendation: CDC is generally more efficient than repeated full extracts, but it is not zero-impact. Perform a source-side DBA assessment.
Slide 11 – Query-Level Monitoring and Alerting
Customer requirement: Centrally monitor long-running, failed and resource-intensive queries.
Primary Databricks resource
system.query.history

This table provides query execution information across supported SQL warehouses and serverless workloads.

Useful attributes include:
- Workspace ID.
- Executing user.
- Query execution status.
- Start and end time.
- Execution duration.
- Compute type.
- Query details and available performance metrics.
Suggested alert rules
Condition	Example threshold
Long-running query	More than 30 minutes
Failed queries	More than 10 failures in 15 minutes
Queue time	More than 5 minutes
Data scanned	More than expected workload baseline
Spill	Abnormal spill compared with historical baseline
Cost	Unusual increase in query or warehouse consumption

Central monitoring approach
1. Query system tables.
2. Create curated monitoring views.
3. Build AI/BI dashboards.
4. Configure SQL alerts.
5. Notify platform teams through approved notification integrations.
Limitation: Validate telemetry availability for each workload type. Not every metric is exposed identically for SQL warehouses, serverless compute, classic compute and pipelines. 
Microsoft Learn+1

Slide 12 – System Tables and Enterprise Monitoring Dashboard
Customer requirement: Out-of-the-box monitoring and delegated access.
Important system tables
System table	Purpose
system.query.history	Query execution history
system.billing.usage	Databricks usage and cost
system.billing.list_prices	Historical pricing
system.access.audit	Security and administrative events
system.access.table_lineage	Table lineage
system.access.column_lineage	Column lineage
system.compute.warehouse_events	SQL warehouse events
system.compute.warehouses	Warehouse configuration history
system.lakeflow	Job and pipeline observability
system.replication.states	Managed DR replication status

Recommended dashboard structure
Platform Operations
- Job failures
- Query performance
- Warehouse utilization
- Pipeline health
Security Operations
- Access activity
- Permission changes
- Audit events
- Data lineage
FinOps
- DBU consumption
- Workspace cost
- Job cost
- Cost by resource tags
Access control
- Restrict raw system-table access.
- Build role-specific views.
- Grant access to platform, security and finance teams according to their responsibilities.
- Protect sensitive query text and audit information.
System tables are designed for historical observability, and their availability and retention vary by table. 
Databricks on AWS+2

Slide 13 – Continuous Jobs and Planned Maintenance Windows
Customer requirement: Stop continuous workloads during a recurring maintenance window and automatically resume.
Two separate requirements
A. Platform maintenance window

Azure Databricks added a native maintenance window for continuous Lakeflow pipelines in September 2026.
- Select weekday.
- Select start hour.
- Select time zone.
- Databricks performs platform updates requiring a restart within a one-hour window.
B. Business-defined downtime

Example:
- Run continuously Monday to Saturday.
- Stop Sunday from 02:00 to 04:00.
- Automatically resume after 04:00.
This requires orchestration.
Recommended architecture
Maintenance Schedule
        |
        v
Stop Streaming Workload
        |
        v
Preserve Checkpoint
        |
        v
Maintenance Window
        |
        v
Restart Workload
        |
        v
Recover and Catch Up


Important considerations
- Preserve checkpoints.
- Stop workloads gracefully.
- Avoid losing unprocessed source changes.
- Validate restart and catch-up behavior.
- Monitor stream health after restart.
Recommendation: Use Databricks Jobs, Jobs API or an enterprise scheduler for business-defined stop/start requirements. Do not confuse this with the native platform-update maintenance window. 
Databricks on AWS

Slide 14 – Cross-Catalog Access and Multi-Region Metastore Architecture
Part A: Cross-catalog access
Customer requirement: Applications should access data across multiple catalogs without repeatedly writing fully qualified object names.

Example:
sales_prod.crm.customers
finance_prod.reporting.revenue
hr_prod.employee.details


An application identity can access multiple catalogs when it has the necessary privileges and workspace bindings permit access.

Use session defaults:
USE CATALOG sales_prod;
USE SCHEMA crm;

SELECT * FROM customers;


Synonyms and abstraction
Do not assume SQL Server-style synonyms are available as a native Unity Catalog feature.

Recommended alternatives:
- Governed views.
- Application-facing schemas.
- SQL functions.
- Dedicated application catalogs.
Example:
CREATE VIEW app_catalog.customer_view AS
SELECT customer_id, customer_name
FROM sales_prod.crm.customers;


Part B: Multi-region metastores
Databricks Account
       |
       +--------------------+
       |                    |
   India Region          Europe Region
       |                    |
   Metastore IN          Metastore EU
       |                    |
   Catalogs               Catalogs
   Workspaces             Workspaces
   ADLS Storage           ADLS Storage


What should be centralized?
Governance component	Recommended approach
Identities and groups	Central account-level identity management
Governed tag definitions	Account-level standard
ABAC policies	Deploy and validate regionally
Catalogs and schemas	Regional
Grants	Synchronize through IaC
Storage credentials	Regional
External locations	Regional
Workspace bindings	Regional

Recommendation: Centralize governance standards but maintain regional data access and storage controls. 
Databricks on Google Cloud+1

Slide 15 – Regional DR, Failover and Final Recommendations
Customer requirement: Recover Unity Catalog metadata, permissions, policies, tables and applications after regional failure.
Databricks Managed DR
Databricks provides Managed Disaster Recovery for supported deployment configurations.

It requires a primary and secondary regional environment and has specific replication prerequisites and limitations.
Important replication considerations
Component	DR consideration
Managed Delta tables	Data and metadata replication supported
External tables	Metadata and external data must be considered separately
Views and functions	Supported assets can be replicated
Grants and permissions	Replication scope must be validated
External storage configuration	Plan secondary-region configuration
Secrets	Explicit recovery/recreation required
Delta Shares	Not replicated
Managed volumes	Not replicated
Model serving endpoints	Not replicated
Vector Search indexes	Not replicated
Streaming workloads	Validate supported pipeline and checkpoint scope

Critical ABAC DR limitation
Security risk to address
Metastore-level ABAC policies are not replicated by Unity Catalog Managed DR.

This means the secondary environment could have replicated data objects without the metastore-level policies protecting them.

The DR runbook must explicitly reapply and validate these policies before allowing users or applications to access the secondary environment. 
Databricks on AWS+1

DR design checklist
1. Define RPO and RTO for each workload.
2. Configure the secondary regional metastore.
3. Configure secondary storage and cloud identities.
4. Define replication scope.
5. Recreate unsupported resources.
6. Reapply security policies where required.
7. Validate application connectivity.
8. Perform regular failover and failback exercises.
Final customer recommendation
Classify each requirement into four categories:
Classification	Meaning
Native GA	Available as a generally available Databricks capability
Native + Configuration	Supported but needs architecture and setup
Beta / Preview	Available with product maturity limitations
Custom Integration	Requires orchestration, cloud services or custom engineering