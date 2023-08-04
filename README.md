# DataCleaning
Demonstration of data cleaning using SQL on BigQuery. 

This readme will provide detail about the process and explain the coding choices I have made and my reasons for these choices. 

# Inspecting a preparing the data


This dataset contains 56,477 entries for Nashville property sale data over a span of 5 years. First, I open the data set in excel and have a look at how things are formatted. 

![inspecting data](https://github.com/AshAnalyzes/DataCleaning/assets/136401402/659bab7c-0d01-424b-8402-ea2a3a3b4259)

The first thing I check for is the date field, as Bigquery requires dates to be formatted as YYYY-MM-DD. This dataset has dates showing as “April 9, 2014’, and I suspect that when I import this into bigquery it will default to YYYY-DD-MM. To fix this, I reformat the date using excel and set the location to Australia. 

 
