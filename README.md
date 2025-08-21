## Data Pipeline

---

### Description
This data pipeline was written using Python and various supporting libraries. It supports my other two dashboard projects: [employee-performance-power-bi-dashboard](https://github.com/gmar84/employee-performance-power-bi-dashboard) and [utilization-power-bi-dashboard](https://github.com/gmar84/utilization-power-bi-dashboard)

### The Problem
Our organization uses several reports to gauge performance. The problem is our EHR system does not have any built-in reporting tools, so traditionally this has meant downloading a CSV file, making manual edits to the data and turning that CSV file into a report. This has taken several hours to perform all the necessary steps from start to finish.

### The solution
The data pipeline extracts the data from the source EHR system, makes all the necessary steps to clean it, and then loads that data into a MySQL database hosted on AWS. 

Here is a very simple and brief overview:
1. Log into the EHR system, download the data, and load it into Python (this is done programmatically using Selenium, more info below)
2. Clean the data: Removing unncessary services, renaming columns, stripping out characters such as commas (,) and converting data types
3. Categorize data: Group services into numbered groups for easier reporting
4. Perform aggregate functions: Service Utilization is calculated as a percentage by performing the following calculation: (in_process_dollars + billed_dollars) / authorized_dollars
5. Load the finalized data set into the MySQL Database

There is a lot more going on than this simple overview. Read on below to see a more in-depth review.

---

### Code
Python Libraries Used:
- pandas
- time
- datetime: date, timedelta
- selenium: webdriver, By, Service
- chromedriver_binary
- requests
- os
- sqlalchemy: create_engine, select
- urllib3
- socket

### Step 1 - Data Extraction

By using the Selenium Chromedriver library, the program logs into the EHR system, navigates the GUI, and downloads the CSV data file. Here is how that is done:
1. By entering Developer mode in the web browser, locating the id element for fields and buttons allows me to locate and interact the various navigation elements
    - Example:
    - The following code sets up the driver object:\
       `driver = webdriver.Chrome()`\
    - Then, we can use the driver to locate and interact with the button that is labeled 'Billing':
    - `driver.find_element(By.LINK_TEXT, "Billing").click()`

