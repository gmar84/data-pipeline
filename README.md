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

### Step 1 - Dates and Billing Cycles Setup

1. start_date and end_date are calling custom defined functions written to determine which dates to use, based on our organization's billing cycles.
   - Example: I wrote the following code to set up date and time functions:

    ```python
    current_time = datetime.now()
    def GetCurYear():
        return str(current_time.year)
    
    def GetCurDay():
        return str(current_time.day)
    
    def GetCurMonthNum():
        return str(current_time.strftime("%m"))
    
    def GetCurDate():
        return date.today().strftime("%m%d%Y")
    
    def GetYesterday():
        return (date.today() - timedelta(days=1)).strftime("%m%d%Y")
        
    def GetPrevMonthNum():
        return (date.today().replace(day=1) - timedelta(days=1)).strftime("%m")
        
    def GetPrevMonthLastDay():
        return (date.today().replace(day=1) - timedelta(days=1)).strftime("%d")
    
    def GetPrevMonthYear():
        return (date.today().replace(day=1) - timedelta(days=1)).strftime("%Y")
    
    def Get30DaysAgo():
        return (date.today() - timedelta(days=30)).strftime("%m%d%Y")
    
    def GetOneYearAgo():
        return (date.today() - timedelta(days=365)).strftime("%m%d%Y")
    
    def GetPrevMonthStartDate():
        return GetPrevMonthNum() + '01' + GetPrevMonthYear()
    
    def GetPrevMonthEndDate():
        return GetPrevMonthNum() + GetPrevMonthLastDay() + GetPrevMonthYear()

   def GetNotesBillingDates():
       BILL1_START = GetCurMonthNum() + '01' + GetCurYear()
       BILL1_END = GetCurMonthNum() + '15' + GetCurYear()
       BILL2_START = GetPrevMonthNum() + '16' + GetPrevMonthYear()
       BILL2_END = GetPrevMonthNum() + GetPrevMonthLastDay() + GetPrevMonthYear()
   
       if int(GetCurDay()) <= 15:
           start_date = BILL2_START
           end_date = BILL2_END
       if int(GetCurDay()) > 15:
           start_date = BILL1_START
           end_date = BILL1_END
       return start_date, end_date
    ```
2. Then, when running the Utilization script, I can simply call those functions, and the program will automatically determine which billing cycle we are currently in, based off today's date, and other supporting date/time functions.
   ```python
    GetUtilization(GetPrevMonthStartDate(), GetPrevMonthEndDate())
   ```

### Step 2 - Data Extraction

#### By using the Selenium Chromedriver library, the program logs into the EHR system, navigates the GUI, and downloads the CSV data file. Here is how that is done:

1. By entering Developer mode in the web browser, locating the id element for fields and buttons allows me to locate and interact the various navigation elements
    - Example: The following code sets up the driver object
      ```python
      driver = webdriver.Chrome()
      ```
    - Then, we can use the driver to locate and interact with the button that is labeled 'Billing':
      ```python
      driver.find_element(By.LINK_TEXT, "Billing").click()
      ```
    - We can also tell it to enter data into fields, such as dates, or other necessary input to filter our data set.
    - Example: The following code finds the start and end date fields, then inputs the necessary dates
      ```python
      driver.find_element(By.ID, 'datefield-1865-inputEl').send_keys(start_date)
      driver.find_element(By.ID, 'datefield-1867-inputEl').send_keys(end_date)
      ```
    - By using the webdriver library, I am able to navigate the system, open the report screen, enter in required filters, and download the data file

2. Due to our EHR system's database, it is querying thousands of records, so it takes time to process the file. So, I wrote a function to detect when the file was ready to be imported.
      ```python
      reportExport = 'C:\\Users\\' + GetUsername() + '\\Downloads\\reportExport.csv'
      
      def CheckReportExportExists():
         while not os.path.exists(reportExport):
            print("File not finished downloading yet, retrying in 30 seconds...")
            time.sleep(30)
         else:
            print("Download Complete.")
      ```
      - Then, simply call the function, read in the csv data file, and delete it from downloads.
      ```python
      CheckReportExportExists()
      df = pd.read_csv(reportExport)
      os.remove(reportExport)
      ```
  
### Step 3 - Data Cleaning

#### Several cleaning and formatting steps are needed to get the dataset into the appropriate format for loading into the database. This includes removing characters such as commans (,) and dollar signs ($), and converting date formats (from MM-DD-YYYY to YYYY-MM-DD), since MySQL requires that format.

```python

def ConvertDate(df, col): 
    # Convert the 'date' column to datetime format
    df[col] = pd.to_datetime(df[col], format='%m/%d/%Y')
    
    # Convert the 'date' column to the desired format
    df[col] = df[col].dt.strftime('%Y-%m-%d')

    #Strip out the timestamp by using string slicing to get only the first 10 characters
    df[col] = df[col].str.slice(0, 10)
    return df

def ConvertDateTime(df, col): 
    # Convert the 'date' column to datetime format
    df[col] = pd.to_datetime(df[col], format='%m/%d/%Y %H:%M %p')
    
    # Convert the 'date' column to the desired format
    df[col] = df[col].dt.strftime('%Y-%m-%d %H:%M %p')

    #Strip out the timestamp by using string slicing to get only the first 10 characters
    df[col] = df[col].str.slice(0, 10)  
    return df

#Cleaning Steps
#Replaces space characters with underscores 
#Checks each column for dollar sign ($) then removes ($,) from only those columns
#Converts date from format MM/DD/YYYY to YYYY-MM-DD
def CleanDataSet(df):
    col_names = df.columns.str.replace(' ', '_').tolist()
    col_names = [x.lower() for x in col_names]
    df.columns = col_names
    for col in df.columns:
        if df[col].dtype == 'object':
            df[col] = df[col].str.replace(r'[\$,]', '', regex=True)
        if 'date' in col:
            ConvertDate(df, col)
    return df

#Fix Column Names: replace spaces with underscores, and converts upper case to lower case
def FixColNames(df):
    col_names = df.columns.str.replace(' ', '_').tolist()
    col_names = [x.lower() for x in col_names]
    df.columns = col_names
```

### Step 4 - Data Formatting

### Test
