# DataCleaning
Demonstration of data cleaning using SQL on BigQuery. 

This README will provide detail about the data cleaning process and explain the coding choices I have made and my reasons for these choices. For just the standalone code, please view the 'SQL Code' file associated with this repository. 

# Inspecting a preparing the data


This dataset contains 56,477 entries for Nashville property sale data over a span of 5 years. First, I open the data set in excel and have a look at how things are formatted. 

![inspecting data](https://github.com/AshAnalyzes/DataCleaning/assets/136401402/659bab7c-0d01-424b-8402-ea2a3a3b4259)

The first thing I check for is the date field, as Bigquery requires dates to be formatted as YYYY-MM-DD. This dataset has dates showing as “April 9, 2014’, and I suspect that when I import this into bigquery it will default to YYYY-DD-MM. To fix this, I reformat the date using excel and set the location to Australia. 

Dates before cleaning:

![dates pre clean](https://github.com/AshAnalyzes/DataCleaning/assets/136401402/888616b2-753b-48bd-9754-f0b21bd6e43a) 

Dates after cleaning: 

![dates post clean](https://github.com/AshAnalyzes/DataCleaning/assets/136401402/269a2c3f-55a4-469e-9ef9-75511759fcae) 
 
During my first attempt to load the data into BigQuery, I encountered the following error message: 

Error while reading data, error message: Error detected while parsing row starting at position: 336543. Error: Missing close double quote (") character.

After a lot of troubleshooting, I eventually found a solution on StackOverflow which suggested I change the write preference to “append to table” and enable the “allow quoted newlines” function under the Advanced options in BigQuery. In doing this, I was able to upload the dataset and begin working. 

I want to try to understand what impact the above setting change may have made to the data, so I run:

SELECT *
 FROM `spatial-motif-388503.Data_cleaning.Datacleaning` 

And notice that the results are not shown in the same order as the CSV file. This makes me wonder if perhaps some of the rows have been deleted, so I check the total number of records match the original file using: 

SELECT COUNT(UniqueID_)
 FROM `spatial-motif-388503.Data_cleaning.Datacleaning` 

This confirms we are still working with 56477 records, although I do notice that the original field name of “UniqueID” has now been changed to “UniqueID_”. 

I run some spot checks on the data by querying specific records by ID, 

SELECT *
 FROM `spatial-motif-388503.Data_cleaning.Datacleaning` 
 WHERE UniqueID_ = 2045

Which supports my belief that the data within the rows has not been altered by the “allow quoted lines” setting change. 
