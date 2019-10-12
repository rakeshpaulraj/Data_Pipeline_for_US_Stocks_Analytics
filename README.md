# Data Analytics of Historical Stock Prices
#### *_Data Engineering Capstone Project_*

### Project Summary
This project is to capture the Historical Stock prices of US stocks and Stock fundamentals and create a data pipeline to load the data into Dimension and Fact tables which will enable to do stock analytics. The goal is to create Stock analytics table which can give us various statistics about each stock using which investment and trading decisions can be made. 

### Scope of Project
* Capture Historical stock data
* Define a data model
* Create Dimension & Fact models based on star schema design
* Create Data Pipeline to load the Dimension and Facts
* Create Stock Analytics table to provide Analytical Insights on the Stock data
* Perform Analytics

### Setup AWS Credentials
  * AWS Access key and Secret Access key are set in a config file `dl.cfg` (In general *.cfg).
  * **Note** : This file will not be checked in git (or) the values cannot be shared. Keep a note to assign your own keys in this file before running this code
  
### Initiate Spark Session
```
from pyspark.sql import SparkSession
spark = SparkSession.builder.\
config("spark.jars.packages","org.apache.hadoop:hadoop-aws:2.7.0")\
.enableHiveSupport().getOrCreate()
```

### Datasets


##### Following datasets are used in this project:
* `**Stock Fundamentals data**`
    * *Description* :  This dataset contains the US stock fundamental paremers such as Revenues, EBITDA, Liabilities etc.
    * *Data Range*  : This data is available for a period ranging from 2008 till 2016
    * *Data Frequency* : Every Quarter
    * *Data format* : csv file (delmited by semicolon)
    * *Size* : 105 MB
    * *Number of records* : 1.95 Million rows
    * *Dataset Source* : https://simfin.com/data/access/download
 
 
* `**Historical Stock Prices data**`
    * *Description* :  This dataset contains the Historical Stock Prices of US Stocks.
    * *Data Range*  : This data is available for a period ranging from 1970 till 2018
    * *Data Frequency* : Daily
    * *Data format* : csv file (delmited by Comma)
    * *Size* : 2 GB
    * *Number of records* : 21 Million rows
    * *Dataset Source* : https://www.kaggle.com/ehallmar/daily-historical-stock-prices-1970-2018
    
    
* `**Stock list**`
    * *Description* :  This dataset contains the Ticker symbol, name and Industry descriptionof each US stock
    * *Data Range*  : This data is available for a period ranging from 1970 till 2018
    * *Data Frequency* : Static data.
    * *Data format* : JSON file
    * *Size* : 1027 KB
    * *Number of records* : 6460 rows
    * *Dataset Source* : https://www.kaggle.com/ehallmar/daily-historical-stock-prices-1970-2018


* `**SP500 Historical Returns**`
    * *Description* :  This dataset contains the Historical price of S&P500 Index
    * *Data Range*  : This data is available for a period ranging from 1970 till 2019
    * *Data Frequency* : Monthly.
    * *Data format* : JSON file
    * *Size* : 47 KB
    * *Number of records* : 599 rows
    * *Dataset Source* : https://finance.yahoo.com/quote/%5EGSPC/history/
   
   
* `**Historical GOLD price**`
    * *Description* :  This dataset contains the Gold price
    * *Data Range*  : This data is available for a period ranging from 1968 till 2019
    * *Data Frequency* : Monthly.
    * *Data format* : JSON file
    * *Size* : 108 KB
    * *Number of records* : 622 rows
    * *Dataset Source* : https://www.quandl.com/data/LBMA/GOLD-Gold-Price-London-Fixing


* `**US Unemployment Rate**`
    * *Description* :  This dataset contains the US Unemployment Rate
    * *Data Range*  : This data is available for a period ranging from 1948 till 2019
    * *Data Frequency* : Monthly.
    * *Data format* : JSON file
    * *Size* : 44 KB
    * *Number of records* : 861 rows
    * *Dataset Source* : https://www.quandl.com/data/FRED/UNRATE-Civilian-Unemployment-Rate
    
#### Dataset location

  * The datasets mentioned above are located in AWS S3 filesystem. Additionally 2 more directories are created in S3 to hold Staging data and Target data. 
Staging tables will be stored under staging_data_path in parquet format. Dimension and Fact tables are stored under target_data_path in parquet format.

`source_data_path = "s3a://<bucket_name>/source-data"`
  
`staging_data_path = "s3a://<bucket_name>/staging-data"`
  
`target_data_path = "s3a://<bucket_name>/target-data"`

**Note : In the above S3 paths, replace the <bucket_name> with your S3 Bucket name anmd create the sub folders such as source-date, staging-data, target-data inside the bucket. Make sure to place the source files under the source-data folder.**
  
  
#### Read and Explore Source data

Source Data | Record count
------------|-------------
Stocks | 6460
Stock Price | 20973889
Stock Fundamentals | 1943245
S&P500 Returns | 599
Gold Price | 622
Unemployment Rate | 861


### Data preparation - Staging data

#### Data Cleansing
  * Convert Ticker symbols to Upper name to avoid inconistencies while joining between datasets
  * Trim whitespaces around Ticker symbols, Indicator Names
  * Convert date values from String datatype to Date datatype
  * Add year and month columns based on the date value
  
  
#### Staging of Source data into Parquet files
  * Load all the Source data into S3 Staging area in S3 location `staging_data_path = "s3a://historical-stock-analytics/staging-data"`
  
  * The data is loaded using parquet format. Parquet, an open source file format for Hadoop. Parquet stores nested data structures in a flat columnar format. Compared to a traditional approach where data is stored in row-oriented approach, parquet is more efficient in terms of storage and performance.
  
  * Use Write mode as `Overwrite` to truncate and load the data on every run
  
  * Join the following 3 data sources and load into one staging table which is **staging_economic_indicators**. Since those are smaller files, having them in one table will be more efficient.
    * SP500 Returns     - `sp_500_returns.json`
    * Gold Price        - `gold_price.json`
    * Unemployment Rate - `unemployment_rate.json`
    
  ![alt text](staging_economic_indicators.png "Title")
    
  * **sp500_returns** and **unemployment_rate** data uses first day of the month to denote the month value whereas **Gold_price** data uses last day of the month. Since all these 3 datasets are going to be consolidated in one staging table, the dates should be consistent. So, the date in gold price dataset is converted from last day to first day of the month. For eg: 2000-01-01 denotes January, 2000-02-01 denotes February. Also extract data only for dates between '1970-01-01' and '2019-09-01'
 
#### Staging Record Counts

Source Data | Record count
------------|-------------
staging_stock | 6460
staging_stock_price | 20973889
staging_stock_fundamentals | 1943245
staging_economic_indicators | 599

  
#### Star Schema Model
To build a data pipeline and perform Stock Analytics, a star schema based dimensional model is chosen. The star schema consists of one or more fact tables referencing any number of dimension tables.
The star schema separates business process data into facts, which hold the measurable, quantitative data about a business, and dimensions which are descriptive attributes related to fact data.

##### Benefits of Star Schema:

The benefits of star schema denormalization are:

  * Simpler queries – star schema join logic is generally simpler than the join logic required to retrieve data from a highly normalized transactional schema.
  * Simplified business reporting logic – when compared to highly normalized schemas, the star schema simplifies common business reporting logic, such as period-over-period and as-of reporting.
  * Query performance gains – star schemas can provide performance enhancements for read-only reporting applications when compared to highly normalized schemas.
  * Fast aggregations – the simpler queries against a star schema can result in improved performance for aggregation operations.
  * Feeding cubes – star schemas are used by all OLAP systems to build proprietary OLAP cubes efficiently; in fact, most major OLAP systems provide a ROLAP mode of operation which can use a star schema directly as a source without building a proprietary cube structure.
  
  
#### Dimension tables
Dimension tables usually have a relatively small number of records compared to fact tables, but each record may have a very large number of attributes to describe the fact data.  Dimensional attributes help to describe the dimensional value. They are normally descriptive, textual values. 

In this `Stock Analytics`, following dimension tables are designed.

* **Date Dimension**
  * All the staging tables used in this project have date based on a time period. For eg: Stock price for every day, Stock fundamentals of every quarter, Economic Indicators of every month. So designing a date dimension table is very essential so that Analysts can derive some useful analytics based on any time frame.
  * Date Dimension has following columns: (For column description, please refer the Data Dictionary)
    * *Date*
    * *Year*
    * *Month*
    * *Day*
    * *Week*
    * *Quarter*
    * *Weekday*
    * *Weekday_name*
    * *Load_ts*
    
    
* **Stock Dimension**
  * Stock Dimension table is designed to store descriptive attributes of each stock such as Stock name, Industry, Sector, Exchange.
  * Stock Dimension has following columns: (For column description, please refer the Data Dictionary)
    * *Ticker*
    * *Name*
    * *Industry*
    * *Sector*
    * *Exchange*
    * *Load_ts*
  
  

#### Fact tables
Fact tables record measurements or metrics for a specific event. Fact tables generally consist of numeric values, and foreign keys to dimensional data where descriptive information is kept. Fact tables are designed to a low level of uniform detail (referred to as "granularity" or "grain"), meaning facts can record events at a very atomic level. This can result in the accumulation of a large number of records in a fact table over time. 

In this **Stock Analytics**, following Fact tables are designed.

* **Stock_Price**
  * Stock_Price Fact table has following columns: (For column description, please refer the Data Dictionary)
    * *Ticker*
    * *Open*
    * *Close*
    * *Adj_close*
    * *Low*
    * *High*
    * *Volume*
    * *Period_date*
    * *load_ts*
    
    
* **Stock_Fundamentals**
  * Stock_Fundamentals Fact table has following columns: (For column description, please refer the Data Dictionary)
    * *Ticker*
    * *Period_date*
    * *Period_year*
    * *Period_quarter*
    * *accounts_payable*
    * *net_profit*
    * *revenues*
    * *dividends*
    * *EBITDA*
    * *total_assets*
    * *total_liabilities*
    * *load_ts*
    

* **Economic_Indicators**
  * Economic_Indicators Fact table has following columns: (For column description, please refer the Data Dictionary)
    * *Period_date*
    * *Period_quarter*
    * *Period_quarter*
    * *sp500_returns*
    * *gold_price*
    * *unemployment_rate*
    * *load_ts*
    
    
* **sp500_Annual_Returns**
  * Stock_Annual_Summary Fact table has following columns: (For column description, please refer the Data Dictionary)
    * *Period_year*
    * *Start_Price*
    * *End_Price*
    * *Avg_price*
    * *rate_of_return*
    * *load_ts*
    
    
* **Stock_Annual_Returns**
  * Stock_Annual_Summary Fact table has following columns: (For column description, please refer the Data Dictionary)
    * *Ticker*
    * *Period_year*
    * *Start_Price*
    * *End_Price*
    * *Avg_price*
    * *Avg_volume*
    * *rate_of_return*
    * *sp500_rate_of_return*
    * *load_ts*
    
 
#### ER Diagram
##### Mentioned below is the Entity-Relationship (ER) diagram which is a visual representation of the Modeling.

  * This diagram is developed on the platform dbdiagram.io <https://dbdiagram.io/home>
  
![alt text](stocks.png "Stock_Analytics_Data_Model")


## Project Architecture

The overall project architecture for Stock Analytics is shown as below:
This process chart is developed using draw.io <https://www.draw.io/>

![alt text](project_architecture.png "Stock_Analytics_Project_Architecture")


## Stock Analytics

One of the main goals of this project is to make the data available for Analysts so that they can various Analytics and derive valuable insights.
There are various analytics that can be run on this Stock Price data.

The following Analysis are performed in this project using visualizations: 
* Comparison of the Total Stock Maket (S&P500 value) with Gold Prices during the same period.
* Most profitable Stocks (Top Gainers) for year 2017
* Range of values for Unemployment rate.
* Number of Companies per each Sector

#### SP500 Index Vs Gold Price
  * Line chart mentioned below compares S&P500 Index value with Gold Price for the time period 1970 to 2019
  
  * An important observation here is to note how the Gold Price increases when the Stock Market returns are low.
  
  * Its very common in Investing world that Investors will redirect the funds from Stocks to Gold as a hedging tecnique and for the reason, we can see the Gold prices when the market is experiencing a pullback or recession.

  * In the below chart, you can see that during the 2007-2010 recession period, S&P500 Index value has dropped. At the same time, Gold Price has gone up.
  
  * Similarly, After 2013 once the recession fears are settled down, Stock Market was set to rebound. So Investors turned their money again from Gold into Stock Market due to which Gold Price went down for next few years.
  
  * Lastly, observe the Spike in the Gold Price since last year. This is due to the fact that the Stock Market is fully saturated and its highlu expected a recession to follow in coming years. So Investors again preparing themselves to route the money to Gold.
  

![alt text](sp500_vs_gold.png "S&P500 Vs Gold Price")


To find out the Most profitable stocks for year 2017, following steps are done:
  * Use the stock_annual_returns fact table
  * Filter data for period_year = 2017
  * Since a lot of Small caps or Penny stocks (Espcially Pharma companies) can have very high volatility, an assumption is made to consider stocks whose Start price is over $20. So filter data whose Start Price > 20
  * Filter data whose rate of return is greater than SP500 rate of returns.
  * Finally fetch only Top 20 Stocks
  
Using the above data, a bar chart is done to show the Top 20 profitable stocks of year 2017.
  * For year 2017, SP500 Rate of return is 17.32%
  * For the Top 20 profitable stocks, the rate of return ranges from 246% to 133%
  * Some of the big names such as SHOP, ANET are also exists in the Top 20
  * Shopify (SHOP) is a well known growth stock. Mentioned below is an article about Shopify's (SHOP) stellar growth on 2017.

    https://www.fool.com/investing/2018/01/22/why-2017-was-a-year-to-remember-for-shopify-inc.aspx
    

![alt text](profitable_stocks_2017.png "Most Profitable Stocks on year 2017")


#### Unemployment Rate

* Below Box plot and Density Plot shows that most of the time, Unemployment Rate stays between 5% and 7%
* The Area chart shows that the Unemployment remained highest from year 2008 till 2011 due to the Economic recession at that time.
* The Area chart shows that the Unemployment is at one of its lowest levels now since the US economy is adding more jobs at recent times.
  
  Mentioned below is an article published recently mentioning that the US unemployment falls to 50 year low.
  
  https://www.whitehouse.gov/articles/u-s-unemployment-rate-falls-50-year-low/

![alt text](unemp_rate.png "Unemployment Rate")

### Tools and Environment

Mentioned below is the list of Tools and Environment used for this project and the rationale behind making these choices.

Tool & Environment | Name | Rationale behind using this tool/platform
------------------ | ------------ | ---------------
Processing framework | Apache Spark | This project is developed using Apache Spark to make use of distributed computing and In-memory processing. 
Data Storage layer | Amazon S3 | Amazon S3 provides easy-to-use management features so you can organize your data and configure finely-tuned access controls to meet your specific business, organizational, and compliance requirements. Amazon S3 is designed for 99.999999999% (11 9's) of durability, and stores data for millions of applications for companies all around the world
Cluster Computing Engine/Service | Amazon EMR | Amazon Elastic MapReduce (EMR) is an Amazon Web Services (AWS) tool for big data processing and analysis. Amazon EMR offers the expandable low-configuration service as an easier alternative to running in-house cluster computing
Development Tools | Python, Ipython, Jupyter notebook | Python has vey rich packages to work with Spark and Data processing needs. In this project, Pyspark package is used for Spark SQL framework. Jupyter Notebook helps to do step by step execution of the program during development stages and provides a neat interface to run the code as well to record the observations in presentable format.
ER Diagram        | dbdiagram.io | A simple and great interface to generate ER diagram
Process flow charts | draw.io | A simple and great interface to draw various process flow diagrams. Has many inbuilt templates to choose from.
Code Repository | Github | GitHub is a Git repository hosting service, but it adds many of its own features. While Git is a command line tool, GitHub provides a Web-based graphical interface. It also provides access control and several collaboration features, such as a wikis and basic task management tools for every project

![alt text](datalake.png "Data Lake")

  
  
### Data dictionary 


#### Date
Column Name | Description
----------- | -------------------------------------------------------------------
date | date value of each day ranging from 1970 till 2019
year | correponding year of each date
month | correponding month of each date
day | correponding numerical day value of each date
quarter | correponding quarter of each date
week | correponding week number of each date
week_day | correponding weekday numeric value of each date. Has values 0 to 6 where 0 represents Monday, 1 -  Tuesday, 2 - Wednesday, 3 - Thursday, 4 - Friday, 5 - Saturday, 6 - Sunday
week_day_name | Corresponding Weekday name of each date. Has values Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday
load_ts | Processing time indicating the time at which the data is loaded in this table.



#### Stock
Column Name | Description
----------- | -------------------------------------------------------------------
ticker|A ticker symbol is an arrangement of characters — usually letters — representing particular securities listed on an exchange or otherwise traded publicly which the investors and traders use to transact orders. 
name | 
industry | Industry is a narrow category used to classify and group stocks which performs similar type of business. Each market sector will have a group of Industries underneath and each industry will have a group of stocks
sector | The term market sector is used in economics and finance to describe a part of the economy. Analysts divide the stock market itself into market sectors so that shares of companies that are in direct competition are listed alongside each other
exchange | A stock exchange, securities exchange or bourse, is a facility where stock brokers and traders can buy and sell securities, such as shares of stock and bonds and other financial instruments.
load_ts | Processing time indicating the time at which the data is loaded in this table.



##### Stock_Price
Column Name | Description
----------- | -------------------------------------------------------------------
ticker|A ticker symbol is an arrangement of characters — usually letters — representing particular securities listed on an exchange or otherwise traded publicly which the investors and traders use to transact orders. 
open | The opening price is the price at which a security first trades when an exchange opens for the day. 
close | Closing price is the price of the final trade before the close of the trading session. 
adj_close | The adjusted closing price shows the stock's value after posting a dividend. 
low | Low price is the lowest price of the security for the given time period. 
high | High price is the highest price of the security for the given time period. 
volume | Trading volume, is the amount (total number) of a security (or a given set of securities, or an entire market) that was traded during a given period of time.
period_date | Event Date or the period on which this event has recorded.
period_year | Corresponding Year value of the Event Date or the period on which this event has recorded.
period_month | Corresponding MOnth value of the Event Date or the period on which this event has recorded.
load_ts | Processing time indicating the time at which the data is loaded in this table.


##### Stock_Fundamentals
Column Name | Description
----------- | -------------------------------------------------------------------
ticker|A ticker symbol is an arrangement of characters—usually letters—representing particular securities listed on an exchange or otherwise traded publicly which the investors and traders use to transact orders. 
period_date | Event Date or the period on which this event has recorded.
period_year | Corresponding Year value of the Event Date or the period on which this event has recorded.
period_quarter | Corresponding Quarter value of the Event Date or the period on which this event has recorded.
accounts_payable| Accounts Payable
net_profit| Net Profit for the given time period
revenues| Total revenues for the given time period
dividends| Dividends paid for the given time period
EBITDA| Earnings Before Tax, Depreciation & Assets
total_assets| Total Assets value for the given time period
total_liabilities| Total Liabilities for the given time period
load_ts | Processing time indicating the time at which the data is loaded in this table.


##### Economic Indicators
Column Name | Description
----------- | -------------------------------------------------------------------
period_date | Event Date or the period on which this event has recorded.
period_year | Corresponding Year value of the Event Date or the period on which this event has recorded.
period_quarter | Corresponding Quarter value of the Event Date or the period on which this event has recorded.
sp500_returns | S&P500 Index value for the given time period
gold_price | Godl price from London Bullion Market Association (LBMA) for the given time period
unemployment_rate | The unemployment rate represents the number of unemployed as a percentage of the labor force. Labor force data are restricted to people 16 years of age and older, who currently reside in 1 of the 50 states or the District of Columbia, who do not reside in institutions (e.g., penal and mental facilities, homes for the aged), and who are not on active duty in the Armed Forces. 
load_ts | Processing time indicating the time at which the data is loaded in this table.


#####  SP500_annual_summary
Column Name | Description
----------- | -------------------------------------------------------------------
period_year | Corresponding Year value of the Event Date or the period on which this event has recorded.
start_price | Starting price of the S&P 500 Index for the given time period
end_price | Ending price of the S&P 500 Index for the given time period
avg_price | Average price of the S&P 500 Index for the given time period
rate_of_return | Rate of Return for S&P500 Index for the given time period. A rate of return (RoR) is the net gain or loss on an investment over a specified time period, expressed as a percentage of the investment's initial cost. 
load_ts | Processing time indicating the time at which the data is loaded in this table.


#####  Stock_annual_summary
Column Name | Description
----------- | -------------------------------------------------------------------
ticker|A ticker symbol is an arrangement of characters—usually letters—representing particular securities listed on an exchange or otherwise traded publicly which the investors and traders use to transact orders. 
period_year | Corresponding Year value of the Event Date or the period on which this event has recorded.
start_price | Starting price of the Stock for the given time period
end_price | Ending price of the Stock for the given time period
avg_price | Average price of the Stock for the given time period
avg_volume | Average volume traded for the Stock for the given time period
rate_of_return | A rate of return (RoR) is the net gain or loss on an investment over a specified time period, expressed as a percentage of the investment's initial cost. 
sp500_rate_of_return | Rate of Return for S&P500 Index for the given time period. A rate of return (RoR) is the net gain or loss on an investment over a specified time period, expressed as a percentage of the investment's initial cost. 
load_ts | Processing time indicating the time at which the data is loaded in this table.




### Scenarios & Solutions for future considerations
 
 
#### 1. Scenario - Data increased by 100x:

For instance, consider the Spark is running on Amazon EMR using EC2 clusters. If the Clusters are of type m5.xlarge, then it provides 4 vCPUs, 16 GB of RAM, Network bandwidth of 10 Gbps. 

In the scenario of our data increasing by 100x, we can handle the load by following options:

1. Increase the number of instances by 100x
2. Upgrade to More powerful instance type
  *  For eg: In Amazopn EMR, Choosing an Instance Type of c5n.18xlarge provides provides 72 vCPUs, 192 GB of RAM and 100Gbps of Network speed.
3. Launch the Instances in Auto Scaling Mode (If your cloud provider has that option). Both Google (GCP) and Amazon (AWS) has Auto scaling mode.
  
  
#### 2. Scenario - data populates a dashboard that must be updated on a daily basis by 7am every day:

This can be achieved by automating the Data pipeline and schedule the run at the required time and frequency using Apache Airflow.

Airflow is a platform to programmatically author, schedule and monitor workflows.

Use airflow to author workflows as directed acyclic graphs (DAGs) of tasks. The airflow scheduler executes your tasks on an array of workers while following the specified dependencies. Rich command line utilities make performing complex surgeries on DAGs a snap. The rich user interface makes it easy to visualize pipelines running in production, monitor progress, and troubleshoot issues when needed.

https://airflow.apache.org/


#### 3. Scenario - database needed to be accessed by 100+ people:
In this project, data is loaded into Data Lake which is stored as Parquet files. The data can be retrieved from Parquet files programatically or by using tools such as Presto.

But if the database needs to be accessed by 100+ people with corresponding access prileges assigned to the end users, then one of the options is to load the final Dimensions and Fact tables into Amazon Redshift. In Redshift, we can configure the Clusters based on workload and the number of users consuming the data. Other reporting tools such as Tableau can connect to Redshift and pull the data for Dashboards.

When we expose the data to multiple users, it is important to manage the access privileges regarding which data users can access and which they should not. These privileges can be setg by creating different IAM roles in AWS and assign such roles to the end users.


