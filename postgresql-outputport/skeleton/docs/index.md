{% set domainNameNormalized = values.domain | replace(r/domain:| |-/, "") %}
{% set dataProductNameNormalized = values.dataproduct.split(".")[1] | replace(r/ |-/g, "") %}
{% set dataProductMajorVersion = values.identifier.split(".")[2] %}
{% set componentNameNormalized = values.name.split(" ") | join("") | lower %}

{% set dataProductNameWithTrims = values.dataproduct.split(".")[1] | replace(r/ /g, "-") %}
{% set domainNameWithTrims = values.domain.split(":")[1] | replace(r/ /g, "-") %}
{% set componentNameWithTrims = values.name.split(" ") | join("-") | lower %}

# PostgreSQL Output Port

This component represents a **PostgreSQL Output Port** within the Data Mesh.
It enables secure access to a PostgreSQL database and automatically generates metadata about the exposed data structure.

The Techadapter will connect to the database and **automatically discover tables, columns and metadata** in order to generate the Data Contract.

---

# Component Basic Information

| Field name            | Example value                  |
| :-------------------- | :----------------------------- |
| **Name**              | ${{ values.name }}             |
| **Description**       | ${{ values.description }}      |
| **Domain**            | ${{ values.domain }}           |
| **Data Product**      | ${{ values.dataproduct }}      |
| **Identifier**        | ${{ values.identifier }}       |
| **Development Group** | ${{ values.developmentGroup }} |
| **Depends On**        | ${{ values.dependsOn }}        |

---

# PostgreSQL Connection Configuration

The following parameters are required to connect to the PostgreSQL database.

| Parameter             | Description                                    |
| --------------------- | ---------------------------------------------- |
| **Database**          | Target PostgreSQL database                     |
| **Schema**            | Database schema to analyze (default: `public`) |
| **Connection String** | JDBC connection string used by the Techada     |
