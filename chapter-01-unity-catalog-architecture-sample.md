# Chapter 01 --- Unity Catalog: Internal Architecture

> **Track:** Enterprise Data & AI Governance\
> **Level:** Solution Architect\
> **Scope:** Databricks Unity Catalog and Unity Catalog OSS\
> **Status:** Architecture foundation
>
> This is a sample extract for validating the GitHub Markdown format.
> Diagrams use Mermaid, which GitHub can render directly in Markdown
> files.

------------------------------------------------------------------------

## 1.1 What is Unity Catalog?

Unity Catalog (UC) is a governance and catalog layer for data and AI
assets. In Databricks, it provides a consistent way to organize assets,
manage permissions, discover data, and audit access across supported
workspaces and workloads.

A useful way to understand UC is to separate three responsibilities:

  -----------------------------------------------------------------------
  Responsibility          What it means           Examples
  ----------------------- ----------------------- -----------------------
  Catalog and metadata    Describes and organizes Metastores, catalogs,
                          governed assets         schemas, tables,
                                                  volumes

  Governance              Determines who can      Ownership, privileges,
                          discover or use assets  workspace bindings, row
                                                  filters, column masks

  Data processing and     Executes workloads and  Spark, SQL Warehouses,
  storage                 stores physical data    cloud object storage,
                                                  Delta files
  -----------------------------------------------------------------------

**Key principle:** Unity Catalog is not the physical storage engine.
Data can remain in cloud object storage while Unity Catalog manages the
metadata and governance relationships around registered assets.

## 1.2 High-level architecture

The following is a conceptual architecture. It describes the main
interactions without claiming to reproduce Databricks' proprietary
internal service topology.

``` mermaid
flowchart TB
    U[Users and Applications]
    ID[Identity Provider and Databricks Identity]
    WS[Databricks Workspace]
    UC[Unity Catalog Governance Service]
    META[Catalog Metadata and Permissions]
    COMP[Databricks Compute]
    STORE[(Cloud Object Storage)]
    DATA[Delta / Parquet / Other Supported Data]

    U --> ID
    U --> WS
    WS --> COMP
    COMP <-->|Metadata and authorization interactions| UC
    UC --> META
    COMP -->|Read or write governed data| STORE
    STORE --> DATA
```

### Main architectural layers

  -----------------------------------------------------------------------
  Layer                   Primary responsibility  Typical components
  ----------------------- ----------------------- -----------------------
  Account and identity    Organization, users,    Databricks account,
                          groups, service         identity provider
                          principals and          
                          workspace               
                          administration          

  Governance              Metadata, privileges,   Unity Catalog metastore
                          policies and asset      and securable objects
                          discovery               

  Compute                 Executes SQL, Spark,    SQL Warehouse,
                          streaming and ML        Databricks Runtime,
                          workloads               serverless compute

  Storage                 Persists data files and Amazon S3, Azure Data
                          transaction logs        Lake Storage, Google
                                                  Cloud Storage
  -----------------------------------------------------------------------

### Control plane and compute plane

The **control plane** provides Databricks-managed platform services,
including workspace management and governance-related services.

The **compute plane** is where workloads execute. Its deployment model
depends on the compute type:

-   **Classic compute:** runs in the customer's cloud environment,
    subject to the workspace deployment and networking configuration.
-   **Serverless compute:** runs in Databricks-managed serverless
    infrastructure.

Do not assume that serverless compute means customer data is
automatically moved into Databricks-managed storage. Compute location
and data-storage location are separate architectural decisions.

``` mermaid
flowchart LR
    subgraph CONTROL["Databricks control plane"]
        A[Workspace and account services]
        B[Identity integration]
        C[Unity Catalog services]
        D[Metadata and governance]
        A --> C
        B --> C
        C --> D
    end

    subgraph COMPUTE["Compute plane"]
        E[SQL Warehouse / Spark compute]
        F[Query execution]
        E --> F
    end

    subgraph CUSTOMER["Customer cloud storage"]
        G[(S3 / ADLS / GCS)]
        H[Data files and Delta transaction logs]
        G --> H
    end

    F <-->|Metadata and authorization interactions| C
    F -->|Governed data access| G
```

## 1.3 Core Unity Catalog hierarchy

In Databricks, the common three-level namespace is:

``` text
catalog.schema.object
```

For example:

``` sql
SELECT *
FROM retail_prod.sales.orders;
```

The names resolve as follows:

  -----------------------------------------------------------------------
  Namespace component     Example                 Meaning
  ----------------------- ----------------------- -----------------------
  Catalog                 `retail_prod`           Top-level logical
                                                  container

  Schema                  `sales`                 Logical container
                                                  within a catalog

  Object                  `orders`                A table, view or
                                                  another supported
                                                  schema-level object
  -----------------------------------------------------------------------

A simplified hierarchy:

``` mermaid
flowchart TB
    M[Unity Catalog Metastore]
    M --> C1[retail_prod]
    M --> C2[finance_prod]
    C1 --> S1[sales]
    C1 --> S2[customer]
    C2 --> S3[finance]
    S1 --> T1[orders table]
    S1 --> T2[revenue view]
    S2 --> V1[customer_files volume]
    S3 --> T3[transactions table]
```

A metastore can be associated with multiple workspaces in the same
region. Workspace association does not automatically grant every user
access to every catalog. Workspace bindings and object privileges are
separate controls.

### Example: Create a catalog and schema

``` sql
CREATE CATALOG IF NOT EXISTS retail_prod;

CREATE SCHEMA IF NOT EXISTS retail_prod.sales;
```

### Example: Create a managed table

``` sql
CREATE TABLE IF NOT EXISTS retail_prod.sales.orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_amount DECIMAL(12, 2),
    order_date DATE
);
```

### Example: Inspect the hierarchy

``` sql
SHOW CATALOGS;

SHOW SCHEMAS IN retail_prod;

SHOW TABLES IN retail_prod.sales;

DESCRIBE EXTENDED retail_prod.sales.orders;
```

> **Implementation note:** Run these examples only in a workspace where
> you have the required privileges. Catalog creation and table creation
> are governed operations.

## 1.4 Metadata is not the same as physical data

Consider a Delta table registered in Unity Catalog.

Unity Catalog manages catalog metadata and governance information, such
as the object's name, schema, owner, privileges and storage
relationship. The underlying Delta table's transaction log and data
files reside at the table's storage location.

A simplified Delta table layout might look like this:

``` text
orders/
├── _delta_log/
│   ├── 00000000000000000000.json
│   └── 00000000000000000001.json
├── part-00000-....parquet
└── part-00001-....parquet
```

Keep these responsibilities distinct:

  Technology             Responsibility
  ---------------------- --------------------------------------------------------
  Unity Catalog          Catalog, metadata and governance
  Delta Lake             Table transaction log and transactional table behavior
  Cloud object storage   Physical persistence of files
  Databricks compute     Query and workload execution

This separation is fundamental to Lakehouse architecture.

## 1.5 Managed tables and external tables

Both managed and external tables can be governed by Unity Catalog. The
key difference is the relationship between the table and its storage
lifecycle.

  -----------------------------------------------------------------------
  Dimension               Managed table           External table
  ----------------------- ----------------------- -----------------------
  Metadata governance     Unity Catalog           Unity Catalog

  Storage location        Within a UC-managed     Explicit external
                          storage hierarchy       location

  Storage lifecycle       Managed by the platform Managed outside the
                                                  table registration

  Drop behavior           UC manages associated   Dropping the table
                          data cleanup according  removes its
                          to the platform's       registration;
                          lifecycle rules         underlying files are
                                                  retained

  Common use              Data managed as a       Existing data or
                          platform data product   storage shared with
                                                  other systems
  -----------------------------------------------------------------------

Example of an external table:

``` sql
CREATE TABLE retail_prod.sales.external_orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_amount DECIMAL(12, 2)
)
USING DELTA
LOCATION 's3://example-retail-data/orders/';
```

The example location is illustrative. In a real deployment, the cloud
path must be covered by an appropriately configured Unity Catalog
external location and storage credential.

**Architectural takeaway:** External does not mean ungoverned. Unity
Catalog can govern the registered table even though the underlying
storage lifecycle is managed separately.

## 1.6 What happens when a user queries a governed table?

Suppose an analyst submits:

``` sql
SELECT *
FROM retail_prod.sales.orders;
```

A simplified request lifecycle is:

``` mermaid
sequenceDiagram
    actor Analyst
    participant SQL as SQL Warehouse
    participant UC as Unity Catalog
    participant Cloud as Cloud Storage

    Analyst->>SQL: Submit SELECT query
    SQL->>UC: Resolve table metadata
    UC-->>SQL: Return applicable metadata
    SQL->>UC: Evaluate required privileges and policies
    UC-->>SQL: Authorization outcome
    alt Access permitted
        SQL->>Cloud: Read governed table data
        Cloud-->>SQL: Return data files
        SQL-->>Analyst: Return query results
    else Access denied
        SQL-->>Analyst: Return authorization error
    end
```

At a conceptual level, the process includes:

1.  **Authentication:** Establish the identity of the user or workload.
2.  **Name resolution:** Resolve the catalog, schema and object.
3.  **Authorization:** Evaluate the required privileges and applicable
    policies.
4.  **Storage access:** Use the platform's supported storage-access
    mechanism.
5.  **Execution:** Read the required data and execute the query.
6.  **Result delivery:** Return results subject to applicable governance
    controls.

The exact internal sequence and storage-credential handling can vary by
compute type, access mode and storage configuration. This diagram is an
explanatory model, not a guarantee that every query triggers a new
credential-vending request.

### Example: Basic table privileges

A user generally needs the relevant usage privileges on the parent
catalog and schema, as well as the required privilege on the table.

``` sql
GRANT USE CATALOG ON CATALOG retail_prod TO `analysts`;

GRANT USE SCHEMA ON SCHEMA retail_prod.sales TO `analysts`;

GRANT SELECT ON TABLE retail_prod.sales.orders TO `analysts`;
```

The principal `analysts` must exist, and the executing administrator
must have authority to grant these privileges.

## 1.7 Architecture checkpoints

Before moving to the next chapter, be able to explain these
distinctions:

  -----------------------------------------------------------------------
  Question                            Correct architectural understanding
  ----------------------------------- -----------------------------------
  Does UC store every table's data?   No. It governs metadata and access
                                      to assets whose data can reside in
                                      external cloud storage.

  Is a workspace a metastore?         No. A workspace is an environment
                                      associated with a metastore.

  Does sharing a metastore grant      No. Workspace bindings and object
  access to all catalogs?             privileges still apply.

  Is Delta Lake the same as Unity     No. Delta Lake provides
  Catalog?                            transactional table behavior; UC
                                      provides catalog and governance
                                      capabilities.

  Does dropping an external table     Normally no. It removes the table
  delete its files?                   registration.

  Does successful login mean a user   No. Authentication and
  can query every table?              authorization are separate.
  -----------------------------------------------------------------------

## 1.8 Official references

Use official documentation as the source of truth for implementation
details and release status.

-   [What is Unity
    Catalog?](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
-   [Databricks high-level
    architecture](https://docs.databricks.com/aws/en/getting-started/high-level-architecture)
-   [Unity Catalog architecture design
    guide](https://docs.databricks.com/aws/en/lakehouse-architecture/deployment-guide/unity-catalog)
-   [Unity Catalog OSS
    repository](https://github.com/unitycatalog/unitycatalog)
-   [Unity Catalog OSS documentation](https://docs.unitycatalog.io/)

> **Version and cloud note:** Databricks capabilities, API behavior and
> feature availability can vary by cloud, region, compute type and
> release status. Validate implementation details against the
> documentation for your target environment.

------------------------------------------------------------------------

**End of sample extract --- Chapter 01**
