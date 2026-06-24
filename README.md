# Customer Sweepstakes Data Cleaning 

**Tools:** SQL

## PROJECT OVERVIEW

This project demonstrates data cleaning techniques using SQL. The dataset contained common data quality issues such as:
- Duplicate records
- Missing values
- Inconsistent formatting
- Unneccesary white space
- Standardization issues

The goal was to transform the raw dataset into a clean and reliable dataset for analysis allowing customers to be entered into a prize draw/sweepstake.

### Removing Special Characters from Column Names

```sql
ALTER TABLE customer_sweepstakes RENAME COLUMN `ï»¿sweepstake_id` TO `sweepstake_id`;
```


### Removing & Deleting Duplicates
```sql
SELECT *
FROM (SELECT customer_id, ROW_NUMBER() OVER(PARTITION BY customer_id) AS duplicate_finder
FROM bakery.customer_sweepstakes) AS duplicates_table
WHERE duplicate_finder > 1
;

DELETE FROM customer_sweepstakes
WHERE sweepstake_id IN (
		SELECT sweepstake_id
		FROM (SELECT customer_id, 
					ROW_NUMBER() OVER(PARTITION BY customer_id) AS duplicate_finder
					FROM bakery.customer_sweepstakes) AS duplicates_table
WHERE duplicate_finder > 1
)
;
```

### Standardizing Data

Standarizing the phone column
```sql
#removing special characters
SELECT phone, REGEXP_REPLACE(phone, '[\]\\[!@#$%.&*`~^{}:;<>/\\|()-]+', '')
FROM bakery.customer_sweepstakes;

#Updating table to reflect above query
UPDATE customer_sweepstakes
SET phone = REGEXP_REPLACE(phone, '[\]\\[!@#$%.&*`~^{}:;<>/\\|()-]+', '')
;

#adding special characters back but with a consistent format
SELECT phone, CONCAT(SUBSTR(phone,1,3),'-',SUBSTR(phone,4,3),'-',SUBSTR(phone,7,4))
FROM bakery.customer_sweepstakes
WHERE phone != '';

#update phone column
UPDATE customer_sweepstakes
SET phone = CONCAT(SUBSTR(phone,1,3),'-',SUBSTR(phone,4,3),'-',SUBSTR(phone,7,4))
WHERE phone != ''

#updating blank values to show as null
UPDATE customer_sweepstakes
SET phone = NULL
WHERE phone = ''
;
```


Standarizing the birth date column
```sql
#looking for duplicates
SELECT birth_date
FROM bakery.customer_sweepstakes
WHERE birth_date IS NULL OR birth_date > CURDATE()
;

#amending the rows where the year is first
SELECT birth_date, STR_TO_DATE(birth_date, '%m/%d/%Y'),CONCAT(SUBSTR(birth_date,9,2),'/',SUBSTR(birth_date,6,2),'/', SUBSTR(birth_date,1,4))
FROM customer_sweepstakes
;

#updating the column for the rows above
UPDATE customer_sweepstakes
SET birth_date = CONCAT(SUBSTR(birth_date,9,2),'/',SUBSTR(birth_date,6,2),'/', SUBSTR(birth_date,1,4))
WHERE sweepstake_id IN (9,11)
;

#formatting dates to be the same
UPDATE customer_sweepstakes
SET birth_date = STR_TO_DATE(birth_date, '%m/%d/%Y')
;
```


Standarizing the Are you over 18 column
```sql
#checking there are any values that dont begin with a 'Y' or 'N'
SELECT  `Are you over 18?`
FROM customer_sweepstakes
WHERE `Are you over 18?` NOT REGEXP '^y|n'
;

#replacing values so they are standardized
SELECT `Are you over 18?`,
	CASE
		WHEN `Are you over 18?` REGEXP '^Y' THEN 'Yes'
        ELSE 'No'
    END
FROM customer_sweepstakes
# WHERE  `Are you over 18?` REGEXP '^Y'
;

#updating table
UPDATE customer_sweepstakes
	SET `Are you over 18?` = CASE
														WHEN `Are you over 18?` REGEXP '^Y' THEN 'Yes'
											      ELSE 'No'
												   END
;
```
### Data Transformation

Splitting the address column into multiple columns
```sql
#separating data from address column
SELECT address, 
SUBSTRING_INDEX(address, ',', 1) AS street,
SUBSTRING_INDEX(SUBSTRING_INDEX(address, ',', 2), ',', -1) AS city,
SUBSTRING_INDEX(address, ',', -1) AS state
FROM customer_sweepstakes
;

#adding new columns to the table
ALTER TABLE customer_sweepstakes
ADD COLUMN street VARCHAR(50) AFTER address,
ADD COLUMN city VARCHAR(50) AFTER street,
ADD COLUMN state VARCHAR(50) AFTER city
;

#inserting data into new tables created
UPDATE customer_sweepstakes
SET street = SUBSTRING_INDEX(address, ',', 1),
		city = SUBSTRING_INDEX(SUBSTRING_INDEX(address, ',', 2), ',', -1),
		state = SUBSTRING_INDEX(address, ',', -1)
;

#standardizing casing of the states
SELECT state, UPPER(state)
FROM customer_sweepstakes
;

#updating the table
UPDATE customer_sweepstakes
SET state = UPPER(state)
;

#removing whitespace at the front of the city and state column
SELECT city, TRIM(city), state, TRIM(state)
FROM customer_sweepstakes
;

#updating the table to reflect whitespace removal
UPDATE customer_sweepstakes
SET city = TRIM(city),
;
```

### Replacing blank values to null values

```sql
UPDATE customer_sweepstakes
SET income = NULL
WHERE income = ''
;
```

### Amending inaccurate records
Inaccurate values in the “Are you over 18” column were identified and corrected by calculating age using the “birth date” field within the same table. This approach ensured that updates were derived from existing reliable data rather than external assumptions.

```sql
UPDATE customer_sweepstakes
SET `Are you over 18?` = CASE
		WHEN YEAR(CURDATE()) - birth_date < 18 THEN 'No'
        ELSE 'Yes'
	END
    ;
```

### Deleting Unneccessary Columns
Based on the context the data was need for, the "Favourite Colour" column was not needed as it had no direct impact on the prize draw
```sql
ALTER TABLE customer_sweepstakes
DROP COLUMN favorite_color
;
```




