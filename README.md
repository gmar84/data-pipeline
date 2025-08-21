### Data Pipeline

---

### Description
This data pipeline was written using Python and various supporting libraries. It supports my other two dashboard projects: (https://github.com/gmar84/employee-performance-power-bi-dashboard)[employee-performance-power-bi-dashboard] and (https://github.com/gmar84/utilization-power-bi-dashboard)[utilization-power-bi-dashboard]

## The Problem
Our organization uses several reports to gauge performance. The problem is our EHR system does not have any built-in reporting tools, so traditionally this has meant downloading a CSV file, making manual edits to the data and turning that CSV file into a report. This has taken several hours to perform all the necessary steps from start to finish.

## The solution
The data pipeline extracts the data from the source EHR system, makes all the necessary steps to clean it, and then loads that data into a MySQL database hosted on AWS. 

Here is a brief overview:
1. Log into the EHR system, download the data, and load it into Python
2. Clean the data: Removing unncessary services, renaming columns, stripping out commas (,) and converting data types
3. Categorize data: Group services into numbered groups for easier reporting
4. Perform aggregate functions: Service Utilization is calculated as a ratio by performing the following calculation: (in_process_dollars + billed_dollars) / authorized_dollars
5. Load the finalized data set into the MySQL Database

---

### Code
This data pipeline is written in Python and uses the following libraries:
- pandas
- time
- datetime:date, timedelta
- selenium: webdriver, By, Service
- chromedriver_binary
- requests
- os
- sqlalchemy: create_engine, select
- urllib3
- socket


