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

I also double check that the date format has been applied correctly, by cross referencing the above data row with the original, and confirm that it has. 

# Dealing with NULL values 

Although the property address is missing in many entries, there are no missing parcel IDs. The parcel ID is unique for each property, and every entry for the same address has the same parcel ID. So, we can use this detail to perform a self join within the table on this value in the case where the parcelID appears more than once, with an address in at least one of those entries. 

Usually you could use:

UPDATE a
SET PropertyAddress = ISNULL(a.PropertyAddress, b.PropertyAddress)
 FROM `spatial-motif-388503.Data_cleaning.cleaning` a
 JOIN `spatial-motif-388503.Data_cleaning.cleaning` b
 ON a.ParcelID = b.ParcelID
 AND a.UniqueID_ <> b.UniqueID_
 WHERE a.PropertyAddress IS NULL

However, I’m using a free version of Bigquery and so do not have the option to use DML queries. To get around this, I’m going to have to create an additional column, “new_address” 

Select *,
      IFNULL(a.PropertyAddress, b.PropertyAddress) as new_prop,
FROM `spatial-motif-388503.Data_cleaning.cleaning` a
JOIN `spatial-motif-388503.Data_cleaning.cleaning` b
ON a.ParcelID = b.ParcelID
AND a.UniqueID_ <> b.UniqueID_
WHERE a.PropertyAddress IS NULL
 
This will create a dataset with an updated field for address (new_address) when the address field is null. However, it also creates duplicate columns for all other fields. To clean this up, I run a second query to pull out only the columns from the original file: 

 SELECT UniqueID_,
       ParcelID,
       LandUse,
       PropertyAddress,
       SaleDate,
       SalePrice,
       LegalReference,
       SoldAsVacant,
       OwnerName,
       OwnerAddress,
       Acreage,
       TaxDistrict,
       LandValue,
       BuildingValue,
       TotalValue,
       YearBuilt,
       Bedrooms,
       FullBath,
       HalfBath,
       new_address
FROM `spatial-motif-388503.Data_cleaning.new_address` 
And save this as a new table (new_addresss2). 

To get the rest of the data, (the data from the original table that did not have null values for the property address), I first create a second table with the same column (new_address): 

SELECT *,
    PropertyAddress AS new_address
FROM `spatial-motif-388503.Data_cleaning.cleaning` 
WHERE PropertyAddress IS NOT NULL

And combine the two new tables using Union All. 

CREATE TABLE `spatial-motif-388503.Data_cleaning.new_address2` AS  
SELECT * FROM `spatial-motif-388503.Data_cleaning.new_address`
UNION ALL 
SELECT * FROM `spatial-motif-388503.Data_cleaning.new_addressnonull`

To spot check if this was correctly executed, I have a look at some of the entries where the PropertyAddress column is Null:

SELECT *,
FROM `spatial-motif-388503.Data_cleaning.new_address_combined` 
WHERE PropertyAddress IS NULL

and check that the parce ID it’s associated with does actually have the same address as the new_address column: 

SELECT *,
FROM `spatial-motif-388503.Data_cleaning.new_address_combined` 
WHERE ParcelID = '026 06 0A 038.00'

They do. Finally, I remove the now redundant address column, and store the results in a new table:

SELECT *
EXCEPT (PropertyAddress)
FROM `spatial-motif-388503.Data_cleaning.new_address_combined`.

# Converting Property Address and Owner Address strings into more usable formats. 

Now that I have all the addresses in 1 column, I want to rework the data within that column to be more usable. At the moment, we have the street number and name followed by a comma, followed by the Suburb. I am going to split these into two columns, 1 containing the street name and number and then the second containing the suburb.  

I use SPLIT to achieve this:

SELECT *,
      SPLIT(new_address, ',')[offset(0)] as Property_Street_address,
      SPLIT(new_address, ',')[offset(1)] as Property_Suburb
FROM `spatial-motif-388503.Data_cleaning.updated_address` 

To separate the owner address into Street Address, Suburb and State, I use:

SELECT *,
      SPLIT(OwnerAddress, ',')[offset(0)] as Owner_Street_address,
      SPLIT(OwnerAddress, ',')[offset(1)] as Owner_Suburb,
      SPLIT(OwnerAddress, ',')[offset(2)] as Owner_State 
FROM `spatial-motif-388503.Data_cleaning.Split1` 

# Removing Duplicates 

In order to locate and remove the duplicates (and because I don’t have the ability to alter or delete content within the tables using BigQuery), I create a CTE which identifies rows which have the same exact values for the ParcelID, Property_Street_address, SalePrice, SaleDate and Legal Reference columns. Records with unique values will have a row_num result of 1. The first instance of a duplicate record will also have a row_num result of 1 and subsequent matching records will have a row_num result of 2 or more depending on the number of duplicate entries for this record. 

 Using the CTE, I create the row_num column, and then run a query to show only unique entries. 

WITH RowNumCTE AS 
(
SELECT *,
        ROW_NUMBER() OVER (
        PARTITION BY ParcelID,
                     Property_Street_address,
                     SalePrice,
                     SaleDate,
                     LegalReference) row_num
FROM `spatial-motif-388503.Data_cleaning.Split2` 
)
SELECT *,
FROM RowNumCTE
WHERE row_num = 1

# Correcting typos in land use column

I noticed during the data cleaning this far that some of the entries have typos under LandUse. E.g “VACANT RESIENTIAL LAND” where it should be “VACANT RESIDENTIAL LAND”. 

After checking that there are no NULL values for this column, I run a query to find out all the possible categories of land use as they are currently coded: 

SELECT *
FROM `spatial-motif-388503.Data_cleaning.unique_records` 
WHERE LandUse IS NULL

SELECT DISTINCT (LandUse)
FROM `spatial-motif-388503.Data_cleaning.unique_records` 
ORDER BY LandUse 

I export these results and this as a guide for the CASE command that I use to rewrite these entries. If I was using a proper version of SQL I would be better able to alter the table for specific instances of each typo, but as this is not available to me, the CASE statement can still achieve the same result (albeit in a much more cumbersome fashion). During this exercise I noticed that the “GREENBELT/RES” Column was behaving a bit differently than the others, and returning a null value when I tried to use the same WHEN statement on it, so I used the “ELSE” statement to reclassify that one (after checking that the CASE statement worked properly for the other columns). 


SELECT *,
    CASE WHEN LandUse = 'APARTMENT: LOW RISE (BUILT SINCE 1960)' THEN 'APARTMENT: LOW RISE (BUILT SINCE 1960)'
        WHEN LandUse = 'CHURCH' THEN 'CHURCH'
        WHEN LandUse = 'CLUB/UNION HALL/LODGE' THEN 'CLUB/UNION HALL/LODGE'
        WHEN LandUse = 'CONDO' THEN 'CONDO'
        WHEN LandUse = 'CONDOMINIUM OFC  OR OTHER COM CONDO' THEN 'CONDO'
        WHEN LandUse = 'CONVENIENCE MARKET WITHOUT GAS' THEN 'CONVENIENCE MARKET WITHOUT GAS'
        WHEN LandUse = 'DAY CARE CENTER' THEN 'DAY CARE CENTER'
        WHEN LandUse = 'DORMITORY/BOARDING HOUSE' THEN 'DORMITORY/BOARDING HOUSE'
        WHEN LandUse = 'DUPLEX' THEN 'DUPLEX'
        WHEN LandUse = 'FOREST' THEN 'FOREST'
        WHEN LandUse = 'GREENBELT' THEN 'GREENBELT'
        WHEN LandUse = 'GREENBELT/RESGRRENBELT/RES' THEN 'GREENBELT'
        WHEN LandUse = 'LIGHT MANUFACTURING' THEN 'LIGHT MANUFACTURING'
        WHEN LandUse = 'METRO OTHER THAN OFC, SCHOOL,HOSP, OR PARK' THEN 'METRO OTHER THAN OFC, SCHOOL,HOSP, OR PARK'
        WHEN LandUse = 'MOBILE HOME' THEN 'MOBILE HOME'
        WHEN LandUse = 'MORTUARY/CEMETERY' THEN 'MORTUARY/CEMETERY'
        WHEN LandUse = 'NIGHTCLUB/LOUNGE' THEN 'NIGHTCLUB/LOUNGE'
        WHEN LandUse = 'NON-PROFIT CHARITABLE SERVICE' THEN 'NON-PROFIT CHARITABLE SERVICE'
        WHEN LandUse = 'OFFICE BLDG (ONE OR TWO STORIES)' THEN 'OFFICE BLDG (ONE OR TWO STORIES)'
        WHEN LandUse = 'ONE STORY GENERAL RETAIL STORE' THEN 'ONE STORY GENERAL RETAIL STORE'
        WHEN LandUse = 'PARKING LOT' THEN 'PARKING LOT'
        WHEN LandUse = 'PARSONAGE' THEN 'PARSONAGE'
        WHEN LandUse = 'QUADPLEX' THEN 'QUADPLEX'
        WHEN LandUse = 'RESIDENTIAL COMBO/MISC' THEN 'RESIDENTIAL COMBO/MISC'
        WHEN LandUse = 'RESIDENTIAL CONDO' THEN 'RESIDENTIAL CONDO'
        WHEN LandUse = 'RESTURANT/CAFETERIA' THEN 'RESTURANT/CAFETERIA'
        WHEN LandUse = 'SINGLE FAMILY' THEN 'SINGLE FAMILY'
        WHEN LandUse = 'SMALL SERVICE SHOP' THEN 'SMALL SERVICE SHOP'
        WHEN LandUse = 'SPLIT CLASS' THEN 'SPLIT CLASS'
        WHEN LandUse = 'STRIP SHOPPING CENTER' THEN 'STRIP SHOPPING CENTER'
        WHEN LandUse = 'TERMINAL/DISTRIBUTION WAREHOUSE' THEN 'TERMINAL/DISTRIBUTION WAREHOUSE'
        WHEN LandUse = 'TRIPLEX' THEN 'TRIPLEX'
        WHEN LandUse = 'VACANT COMMERCIAL LAND' THEN 'VACANT COMMERCIAL LAND'
        WHEN LandUse = 'VACANT RES LAND' THEN 'VACANT RESIDENTIAL LAND'
        WHEN LandUse = 'VACANT RESIDENTIAL LAND' THEN 'VACANT RESIDENTIAL LAND'
        WHEN LandUse = 'VACANT RESIENTIAL LAND' THEN 'VACANT RESIDENTIAL LAND'
        WHEN LandUse = 'VACANT RURAL LAND' THEN 'VACANT RURAL LAND'
        WHEN LandUse = 'VACANT ZONED MULTI FAMILY' THEN 'VACANT ZONED MULTI FAMILY'
        WHEN LandUse = 'ZERO LOT LINE' THEN 'ZERO LOT LINE'
        ELSE 'GREENBELT/RES'
   END AS Land_Use
FROM `spatial-motif-388503.Data_cleaning.unique_records` 
ORDER BY UniqueID_


Finally, I’m going to remove the old (duplicate) columns of any values I haven’t already, along with any columns that we do not need in the analysis, and rename the column “new_address” to “property_address” for clarity. We would normally be able to use ALTER TABLE and DROP COLUMN, but these functions are not compatible with BigQuery. For ease of working with the file, I limit the results to 30,000 entries. 

SELECT *
EXCEPT(OwnerAddress, new_address, LandUse, row_num)
FROM `spatial-motif-388503.Data_cleaning.unique_records_land_use`
ORDER BY UniqueID_      
LIMIT 30000


