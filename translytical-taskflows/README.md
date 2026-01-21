<!--
---
page_type: sample
languages:
- python
products:
- fabric
- fabric-database-cosmos-db
- power-bi
name: |
   Translytical Task Flows
urlFragment: translytical-taskflows
description: Build end-to-end translytical workflows combining Cosmos DB in Fabric, User Data Functions, and Power BI for real-time data updates from interactive reports.
---
-->

# 🔄 Translytical Task Flows with Cosmos DB in Microsoft Fabric

**Build end-to-end translytical workflows combining operational and analytical workloads**

This tutorial walks you through building an end-to-end translytical task flow that utilizes Cosmos DB in Microsoft Fabric as the data store. Learn how operational and analytical workloads can work together seamlessly using User Data Functions and Power BI.

> **🎯 Note**: This sample demonstrates how to create interactive Power BI reports that can write data back to Cosmos DB in real-time using User Data Functions, enabling true translytical scenarios.
> **Note:** Translytical task flows are currently in public preview.

## 🎯 What You'll Learn

### **📊 Translytical Concepts**

- **Real-time data updates** from Power BI reports back to Cosmos DB
- **User Data Functions** for executing business logic from visualizations
- **DirectQuery connectivity** for live data access
- **Interactive data workflows** combining analytics and operations

### **🔗 Fabric Integration Patterns**

- **Cosmos DB in Fabric** as the unified data store
- **User Data Functions** for write-back operations
- **Power BI DirectQuery** for real-time reporting
- **Data function buttons** for triggering operations from reports

### **🛠️ Development Patterns**

- **Function parameter mapping** from report controls
- **Error handling** for Cosmos DB operations
- **Price history tracking** with append-only patterns
- **Report refresh automation** after data updates

## 📋 Prerequisites

### **Required Services**

- **Microsoft Fabric workspace** with appropriate permissions (or [start a free Fabric trial](https://learn.microsoft.com/fabric/get-started/fabric-trial))
- **Cosmos DB artifact** in Fabric with sample data loaded
- **[Power BI Desktop](https://powerbi.microsoft.com/desktop/)** with preview features enabled

### **Required Data**

- **SampleData container** - Product catalog with pricing and reviews (loaded from Cosmos DB sample data)

## 🏗️ Solution Architecture

A translytical task flow combines three key components:

| Component | Purpose | Technology |
|-----------|---------|------------|
| **Store Data** | Real-time access to operational data for analytics | Cosmos DB in Fabric |
| **Develop Data Logic** | Execute business logic when triggered from reports | User Data Functions |
| **Visualize & Interact** | Display data and provide interactive controls | Power BI with DirectQuery |

## 🚀 Getting Started

### Step 1: Create a Cosmos DB Database in Microsoft Fabric

This sample utilizes the sample data provided in the Cosmos DB database in Microsoft Fabric.

1. Create a new workspace in Microsoft Fabric. Follow the  steps outlined in [Create a workspace](https://learn.microsoft.com/en-us/fabric/fundamentals/create-workspaces).

1. Follow the steps outlined at [Create a Cosmos DB database in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/database/cosmos-db/quickstart-portal). The steps involve creating a new Cosmos DB database in your Fabric workspace and loading sample data into it.

    > **Note:** Ensure that you load the sample data into your Cosmos DB database as described in the tutorial, we'll be using that data later in the tutorial.

1. Take note of the database name and container name you create, as you'll need them later.

### Step 2: Build and Publish a User Data Function

Next, we'll create a user data function that updates the current price of a product in the Cosmos DB database.

1. In your Fabric workspace, select **+ New Item** from the top-right corner then select **User Data Function** in the Develop data section.

    ![New User Data Function](./images/new-user-data-function.png)

1. Provide a name for your User Data Function, e.g., `update_price_writeback`.

1. On the User Data Function editor page, select **New function**.

    ![New Function](./images/new-function.png)

1. Replace the default code in the editor with the following code snippet.

    ```python
    import fabric.functions as fn
    udf = fn.UserDataFunctions()

    import logging
    from typing import Any
    from datetime import datetime, timezone
    from fabric.functions.cosmosdb import get_cosmos_client
    from azure.cosmos import exceptions

    @udf.generic_connection(argName="cosmosDb", audienceType="CosmosDB")
    @udf.function()
    def update_price(cosmosDb: fn.FabricItem, categoryName: str, productId: str, newPrice: float) -> list[dict[str, Any]]:

        COSMOS_DB_URI = "YOUR_COSMOS_DB_URI_HERE"
        DB_NAME = "YOUR_DATABASE_NAME_HERE"
        CONTAINER_NAME = "SampleData"

        try:
            cosmosClient = get_cosmos_client(cosmosDb, COSMOS_DB_URI)
            database = cosmosClient.get_database_client(DB_NAME)
            container = database.get_container_client(CONTAINER_NAME)

            # Get the current product document
            product = container.read_item(item=productId, partition_key=categoryName)

            # Update the product's price
            product["currentPrice"] = newPrice
            
            now = datetime.now().replace(microsecond=0)
            current_time_iso = now.isoformat()

            # Append to the price history
            product["priceHistory"].append({
                "date": current_time_iso,
                "price": newPrice
            })

            # Replace the document in the container
            container.replace_item(item=productId, body=product)

            return product

        except exceptions.CosmosResourceNotFoundError as e:
            logging.error(f"Item not found in update_product: {e}")
            raise
        except exceptions.CosmosHttpResponseError as e:
            logging.error(f"Cosmos error in update_product: {e}")
            raise  
        except exceptions as e:
            logging.error(f"Unexpected error in update_product: {e}")
            raise
    ```

1. Replace `YOUR_COSMOS_DB_URI_HERE` with the URI of your Cosmos DB database. You can find this in the **Settings** > **Connection** section of your Cosmos DB database in Fabric.

1. Replace `YOUR_DATABASE_NAME_HERE` with the name of the Cosmos DB database that you created earlier.

1. On the top menu bar, select **Publish** to publish your user data function.

    ![Publish User Data Function](./images/publish-user-data-function.png)

1. Once the function is published, hover over the function name in the left pane and select the **Test** icon.

    ![Test User Data Function](./images/test-user-data-function.png)

1. In the test pane, provide sample input values for `categoryName`, `productId`, and `newPrice`. For example:

    - `categoryName`: Computers, Desktops
    - `productId`: 05bca58f-257c-4129-8373-1b0951cb8104
    - `newPrice`: 5000

1. Select the **Test** button to execute the function. Review the output and logs to ensure the function executed successfully then close the test pane.

### Step 3: Connect & Transform Data in Power BI

Finally, we'll create a Power BI report that allows users to update product prices and see the changes reflected in real-time.

1. From the Power BI Desktop, create a new report.

1. Connect to your Cosmos DB database in Fabric by selecting **Get Data** > **More** then search for **Cosmos DB** and select **Azure Cosmos DB v2** connector and select **Connect**.

    ![Get Data Cosmos DB](./images/get-data-cosmos-db.png)

1. In the connection dialog, provide the Cosmos DB Endpoint and for **Data Connectivity mode**, select **DirectQuery**. Select **OK**.

    ![Cosmos DB Connection](./images/cosmos-db-connection.png)

1. In the Navigator pane, select the Cosmos DB database and the two containers: `SampleData` and `SampleData_PriceHistory`. Select **Transform Data** to open the Power Query Editor.

    ![Navigator Pane](./images/navigator-pane.png)

1. In the Power Query Editor, rename **SampleData_priceHistory[]** to **PriceHistory** for easier reference.

1. Select the **SampleData** table and remove the following columns: `_attachments`, `_etag`, `_rid`, `_self`, `_ts`, `customerNAme`,`rating`, `reviewDate`, `reviewText`, `stars`, `SampleData_priceHistory[](categoryName)` and `SampleData_priceHistory[](id)`. This can be done by selecting the columns, right-clicking, and choosing **Remove Columns**.

1. Select the **docType** column, in the **SampleData** table, then filter to only include rows where `docType` is equal to `product`.

1. Select the **PriceHistory** table and remove the following columns: `SampleData(categoryName)`, `SampleData(id)`.

1. Rename the columns `SampleData_priceHistory[]_date` and `SampleData_priceHistory[]_price` to `Date` and `Price` respectively.

1. Close and apply the changes to load the data into Power BI.

    ![Power Query Editor](./images/power-query-editor.png)

### Step 4: Create Interactive Report in Power BI

In this section, build visuals with the data that you loaded into your Power BI report and connect the user data function to enable price updates.

1. Switch to the **Model** view in Power BI Desktop, and select **Manage Relationships** from the top menu.

    ![Manage Relationships](./images/manage-relationships.png)

1. In the Manage Relationships dialog, select **New Relationship** and in the New Relationship dialog, create a relationship between the `id` field in the `SampleData` table and the `id` field in the `PriceHistory` table. Select **Save** to create the relationship.

    ![New Relationship](./images/new-relationship.png)

1. In the **Visualizations** pane, select the **Slicer** visual to add it to the report canvas, and from the **Data** pane, drag the `categoryName` field from the `PriceHistory` table.

    ![Category Slicer](./images/category-slicer.png)

1. Select the **Format Visual** tab in the Visualizations pane, in the **Slicer Settings** section, change the **Style** to **Dropdown**.

    ![Slicer Format](./images/slicer-format.png)

1. Using the same steps, add another slicer visual for the `name` field from the `SampleData` table. This slicer will allow users to select a specific product.

1. Add a **Card** visual to the report canvas and select the `currentPrice` field from the `SampleData` table.

    ![Current Price Card](./images/current-price-card.png)

1. In the **Format Visual** tab, select the **General** tab. Toggle on **Title**, and set the title text to `Current Price` by expanding the **Title** section.

1. In the **Callouts** section of the **Format Visual** tab, toggle off **Label**.

1. Add a **Line Chart** visual to the report canvas. From the **Data** pane, drag the `Date` field from the `PriceHistory` table to the **X axis** well, and the `Price` field to the **Y axis** well.

    ![Price History Line Chart](./images/price-history-line-chart.png)

1. Add a **Text Slicer** visual to the report canvas.

    ![Text Slicer](./images/text-slicer.png)

1. Select the text slicer and use the Format visual > General > Title options to give the text slicer the following title: `Enter New Price`.

1. On the taskbar, select the Insert menu and add a Blank button to the report. Drag the button under the text slicer.

1. Select the button and expand the Action options in the Format button pane. Turn the Action radio button to On.

1. Provide the following values for your button:

    - **Type**: Data function
    - **Data function**: Select the `update_price_writeback` function that you created earlier.

1. Map the function parameters to the appropriate fields and controls in your report:

    - **categoryName**: Select the `categoryName` field from the `PriceHistory` table.
    - **productId**: Select the `id` field from the `SampleData` table.
    - **newPrice**: Select the text slicer that you added earlier.

1. Select the button and expand the Style options in the Format button pane. Turn the Text radio button to On and label your button Enter.

1. In the Style options, switch the **Apply settings to** option to **Loading**. The loading state is a unique state available to data function buttons that you can use to define style options that appear while the data function is running.

1. Expand the **Text** menu and replace the button text value with `Submitting`.

### Step 5: Publish and Test the Report

1. Save your Power BI report and select **Publish** from the Home tab to publish the report to your Fabric workspace.

1. In Power BI Desktop, open the published report from your Fabric workspace.

1. Use the category and product slicers to select a product and view its current price and price history.

1. In the text slicer, enter a new price for the selected product.

1. Click the Submit button to invoke the user data function and update the product price in Cosmos DB.

1. After the function completes, select the **Refresh** button and notice that the current price and price history visuals update to reflect the new price.

## 📚 Additional Resources

- [Cosmos DB in Fabric Documentation](https://docs.microsoft.com/fabric/database/cosmos-db/overview)
- [User Data Functions Documentation](https://learn.microsoft.com/fabric/data-engineering/user-data-functions-overview)
- [Power BI DirectQuery](https://learn.microsoft.com/power-bi/connect-data/desktop-directquery-about)
- [Create a Fabric Workspace](https://learn.microsoft.com/fabric/fundamentals/create-workspaces)
