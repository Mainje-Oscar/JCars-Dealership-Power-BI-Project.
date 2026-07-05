## JCars Logistics Performance Analytics Dashboard
## A Power BI project on car sales in Kenya.
### Project Objective
The objective of this project was to import, clean, model, and visualize a flat, un-normalized dataset for JCars Logistics Company. By deploying a Star schema model, this interactive Power BI report reveals critical operational data points surrounding vehicle inventory profitability, supply chain logistics delays across Kenyan counties, and sales representative performance to inform high level decision making.


### 1. How I imported data into the Cloud hosted Database (PostgreSQL on Aiven)

This process assumes that the database is already set up on Aiven. It further assumes that because Aiven is hosted in the cloud, and cannot directly get access through a local path on your a laptop, a data streaming SQL client DBeaver in my case (or pgAdmin) has been installed on the laptop and a connection has been established between the service on Aiven and DBeaver (since I had already done this). 

**Importing Data**
- **Step 1:** Open DBeaver, in the left navigator bar, expand the Aiven connection, in the created schema wherein the CSV file is to be imported and follow the steps below.
Right-click your newly created table name in the left navigator bar, Select **Import Data → Input files → Browse**.
Choose CSV as the source and browse to locate the file on your laptop.
Click **Next**, map your CSV headers to your database columns.
Click **Start** to execute the pipeline upload into Aiven

- **Step 2**:Power BI Desktop utilizes the underlying Windows Certificate Store to validate encrypted database links. Because Aiven uses its own to secure cloud data, Windows will block the connection as "Untrusted" unless we add it to the laptop's root directory.

- **Part A:** Download the CA certificate from Aiven
Open your web browser, navigate to your Aiven Console, and click on your running PostgreSQL service.
Stay on the Overview page and scroll down to the Connection information section.
Locate the row labeled CA Certificate and click the Download button next to it.
This will save a file named `ca.pem` directly to the laptop's Downloads folder.

- **Part B:** Install it into the Windows Trusted Root Store.
To make this certificate trusted by Power BI for all connections, you must install it at the local machine.
In the search box, **search Manage User Certificates**.
In the left-hand console folder tree, expand Certificates.
**Right-click** on the Trusted Root Certification Authorities folder → hover over **All Tasks → click Import**
The Certificate Import Wizard will open.
Click **Next → Browse**  to find your file.

**Crucial Step**: In the file browser pop-up, change the file extension filter dropdown in the bottom right to All Files. Otherwise, your `ca.pem` file will stay hidden.
Navigate to your Downloads folder, select ca.pem, and click **Open**.
Click **Next**.
Ensure the option Place all certificates in the following store is selected and that it explicitly targets the Trusted Root Certification Authorities store.
Click **Next**.
Review the summary page and click **Finish**. 
A pop-up confirming "The import was successful" will appear.
Close the console.

- **Step 3: Establishing the Connection in Power BI Desktop**

Now that your laptop recognizes Aiven as a trusted authority, you can complete the link securely.
Launch Power BI Desktop. 
On the top Home ribbon, click **Get Data → select More** → choose **Database** → select **PostgreSQL database**. 
Click **Connect**.

Input your parameters copied from your Aiven Console:
- Server: Enter your Host name followed by a colon and the port number (e.g.,**`oscar-oscarmainje-b405.j.aivencloud.com:Port Number`**)
- Database:Type  **`defaultdb`**

Data Connectivity Mode

Select **Import**.
Click **OK**.

When the database authentication window pops up, click the Database tab on the left-side margin.
Input your database user details.
- User name: **`avnadmin`**
- Password: Paste your specific Aiven service password.
  Click **Connect**
  
A Navigator canvas window will appear displaying your database contents. 
Simply check the box next to your loaded table (jcars in my case) and click **Transform Data** to start transforming it within Power Query

### 2. Data model
To optimize compute performance and eliminate redundancies, the flat table was split into a normalized Star Schema:
- **`dim_customers`**: Contains categorical attributes of all buyers.
- **`dim_vehicles`**: Has inventory information (`Car Make`, `Car Model`, `Vehicle Type`).
- **`dim_location`**: Aggregates branches, cities, counties, and regional offices.
- **`dim_salesperson`**: Contains information relating to all salespersons.
- **`fact_sales`**: Contains transactional primary metrics.

### Calculated Measures (DAX)

The calculations of this dashboard were done using the following formulas:

```dax
// Sales volume measures

Number of Cars Sold = SUM('fact_sales'[Units Sold])
Number of Orders = DISTINCTCOUNT(fact_sales[Order ID])

// Financial perfomance measures

Sales Revenue = SUMX('fact_sales', (('fact_sales'[Unit Selling Price] * 'fact_sales'[Units Sold]) + 'fact_sales'[Delivery Fee]))
Recorded Sales Revenue = SUM('fact_sales'[Revenue Recorded])
Gross Profit = SUMX('fact_sales',(('fact_sales'[Unit Selling Price] * 'fact_sales'[Units Sold]) + 'fact_sales'[Delivery Fee]) - (('fact_sales'[Unit Cost] * 'fact_sales'[Units Sold]) + 'fact_sales'[Logistics Cost]))
Gross Profit Margin = DIVIDE([Gross Profit], [Sales Revenue], 0)
Recorded Gross Profit Margin = DIVIDE([Gross Profit], [Recorded Sales Revenue], 0)
Average Discount = AVERAGE(fact_sales[Discount])
Avg Revenue Per Customer = DIVIDE([Sales Revenue], DISTINCTCOUNT(fact_sales[Customer ID]), 0)

// Supply Chain measures

Average Delivery Days = AVERAGEX(fact_sales, DATEDIFF(fact_sales[Order Date], fact_sales[Delivery Date], DAY))
Logistics Cost % of Revenue = DIVIDE(SUM(fact_sales[Logistics Cost]), [Sales Revenue], 0)

// Operational measures

Total Returns Count = CALCULATE(COUNT(fact_sales[Order ID]), fact_sales[Returned] = "Y" || fact_sales[Delivery Status] = "Canceled")
Average Rating = AVERAGE(fact_sales[Customer Rating])
```

---

### Visual features of the dashboard

- **Executive Summary Page**: Contains the KPI matrix blocks alongside a geographical sales density map and line/column chart breaking down monthly performance trends. This can be filtered using a location slicer. 
- **Inventory Perfomance Page**: Has bar and column charts analysing key product perfomance metrics like profit, profit margin, number of units sold etc.
- **Additional features Page**: Contains additional tables showing customers that should be investigated as there orders were flagged as those with pending payment and cars delivered. This page also has a visual analysing average delivery days by region.

### Key Dashboard Insights

- Specific satellite branches located far away from the main branch have Logistics Cost % of Revenue rising to up to 2% of Sales Revenue. This points to substantial capital leakage in regional transit.
- The car models for year 2016 generating the highest total volume in Units Sold but underperforming on gross profit margins due to fairly high discounting strategies applied by regional sales representatives.
- A notable portion of overall Sales Revenue is tied to orders marked as Delivered under Delivery Status but have a Pending status in Payment Status. A table displaying such customers is on the Additional Visuals page.
- Traditional social media Lead Sources such as WhatsApp and Phone Calls are delivering smaller average deal sizes, whereas specialized channels such as Walk-ins yield the highest Average Customer Value.
- There's a notable increase in Average Delivery Days across distant counties, indicating clear freight consolidation or customs clearing delays.
- There was a huge gap of around 500M between the total Sales Revenue calculated from the Sales and Cost records and the Revenue Recorded. This may have been caused by under-declaration of Sales Revenue.

 ### Actionable Recommendations

- Restructure the supply chain network by grouping transit deliveries into regions for any branch where logistics overhead exceeds 1% of recorded sales revenue.
- Enforce an immediate credit freeze and transition to Cash on Delivery or verified debt clearing guidelines for regions showing high volumes of unresolved payment balances.
- A system should be put in place to automatically calculate and record Sales Revenue from the sales and cost records.
- Set the allowable discount to be between 9% and 12% as this proves to be the discount range that has generated the highest Sales Revenue.


