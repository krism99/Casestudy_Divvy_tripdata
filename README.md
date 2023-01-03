# Casestudy_Divvy_tripdata
Case Study for Google Data Analytics Capstone Project - Analysis of Divvy Tripdata

3 Jan 2023

# Data loading

## Download 
  Download following files from https://divvy-tripdata.s3.amazonaws.com/index.html
  - 202112-divvy-tripdata.csv
  - 202201-divvy-tripdata.csv
  - 202202-divvy-tripdata.csv
  - 202203-divvy-tripdata.csv
  - 202204-divvy-tripdata.csv
  - 202205-divvy-tripdata.csv
  - 202206-divvy-tripdata.csv
  - 202207-divvy-tripdata.csv
  - 202208-divvy-tripdata.csv
  - 202209-divvy-tripdata.csv
  - 202210-divvy-tripdata.csv
  - 202211-divvy-tripdata.csv
  
## Clean and merge .csv files
  Remove last empty lines from .csv files.  
  Copy and paste into one large temp01.csv, remove headers
  
## Import

### Create external table to house raw data
``` 
CREATE TABLE divvy_trips_ext  
( 
rental_id  VARCHAR2(255 BYTE),  
started_at  VARCHAR2(255 BYTE),  
ended_at  VARCHAR2(255 BYTE),  
bike_id   VARCHAR2(255 BYTE),  
duration_s   VARCHAR2(255 BYTE),  
start_station_id  VARCHAR2(255 BYTE),  
start_station_name  VARCHAR2(255 BYTE),  
end_station_id  VARCHAR2(255 BYTE),  
end_station_name  VARCHAR2(255 BYTE),  
user_type  VARCHAR2(255 BYTE),  
gender  VARCHAR2(255 BYTE),  
bday_year  VARCHAR2(255 BYTE)  
)  
ORGANIZATION EXTERNAL  
    (  TYPE ORACLE_LOADER  
    DEFAULT DIRECTORY casestudy1  
    ACCESS PARAMETERS   
        ( RECORDS DELIMITED BY newline  
        BADFILE casestudy1:'temp01.bad'  
        DISCARDFILE casestudy1:'temp01.dsc'  
        LOGFILE casestudy1:'temp01.log'  
        SKIP 0  
        FIELDS TERMINATED BY ','  OPTIONALLY ENCLOSED BY '"'  
        MISSING FIELD VALUES ARE NULL  
        REJECT ROWS WITH ALL NULL FIELDS  
(  
rental_id  char,  
started_at  char,  
ended_at  char,  
bike_id   char,  
duration_s   char,  
start_station_id  char,  
start_station_name  char,  
end_station_id  char,  
end_station_name  char,  
user_type  char,  
gender  char,  
bday_year  char  
)     
                    )  
    LOCATION (casestudy1:'temp01.csv')  
    )  
REJECT LIMIT UNLIMITED  
NOPARALLEL  
NOMONITORING;  
```
### Compare rows loaded into database with rows from spreadsheets (excluding header rows)
```
Filename		
202112-divvy-tripdata.csv				247540									
202201-divvy-tripdata.csv				103770									
202202-divvy-tripdata.csv				115609									
202203-divvy-tripdata.csv				284042									
202204-divvy-tripdata.csv				371249									
202205-divvy-tripdata.csv				634858									
202206-divvy-tripdata.csv				769204									
202207-divvy-tripdata.csv				823488									
202208-divvy-tripdata.csv				785932									
202209-divvy-publictripdata.csv 701339			(renamed to 202209-divvy-tripdata.csv)
202210-divvy-tripdata.csv				558685									
202211-divvy-tripdata.csv				337735									
========================================================================================
Totals								   5733451							

> select count(*) from divvy_trips_ext   ;
	   5733451
```

### Confirm there are no .bad files, nor .dsc  present in the directory  (implies a failed load)
Confirmed

### Determine what the datatypes should be and their length, then create table with correct datatypes and insert raw data into it.
```
select
  max(length(ride_id))
 ,  max(length(RIDEABLE_TYPE))
 , max(length(STARTED_AT))
 , max(length(ENDED_AT))
 , max(length(START_STATION_NAME))
 , max(length(START_STATION_ID))
 , max(length(END_STATION_NAME))
 , max(length(END_STATION_ID))
 , max(length(START_LAT))
 , max(length(START_LNG))
 , max(length(END_LAT))
 , max(length(END_LNG))
 , max(length(MEMBER_CASUAL))
  from divvy_trips_ext;
  
 
RIDE_ID					16
RIDEABLE_TYPE 			13
STARTED_AT				19
ENDED_AT				19
START_STATION_NAME		64
START_STATION_ID		44
END_STATION_NAME		64
END_STATION_ID			44
START_LAT				18
START_LNG				18
END_LAT					18
END_LNG					18
MEMBER_CASUAL			 6
```

### Create target table with correct (ample) sizing and datatypes, then insert from external table
```
create table divvy_tripdata
(
  RIDE_ID              VARCHAR2(50 BYTE),
  RIDEABLE_TYPE        VARCHAR2(50 BYTE),
  STARTED_AT           DATE,
  ENDED_AT             DATE,
  START_STATION_NAME   VARCHAR2(200 BYTE),
  START_STATION_ID     VARCHAR2(200 BYTE),
  END_STATION_NAME     VARCHAR2(200 BYTE),
  END_STATION_ID       VARCHAR2(200 BYTE),
  START_LAT            VARCHAR2(18 BYTE),
  START_LNG            VARCHAR2(18 BYTE),
  END_LAT              VARCHAR2(18 BYTE),
  END_LNG              VARCHAR2(18 BYTE),
  MEMBER_CASUAL        VARCHAR2(50 BYTE),
  RIDE_LENGTH_MINUTES  NUMBER(10) GENERATED ALWAYS AS (("ENDED_AT"-"STARTED_AT")*24*60),
  DAY_OF_WEEK          VARCHAR2(10 BYTE) GENERATED ALWAYS AS (TO_CHAR("STARTED_AT",'d'))
)...;

insert into divvy_tripdata
(RIDE_ID, RIDEABLE_TYPE, STARTED_AT, ENDED_AT, START_STATION_NAME, START_STATION_ID, END_STATION_NAME, END_STATION_ID, START_LAT, START_LNG, END_LAT, END_LNG, MEMBER_CASUAL)
(select 
  trim(RIDE_ID)
, trim(RIDEABLE_TYPE)
, to_date(STARTED_AT,'yyyy-mm-dd hh24:mi:ss')
, to_date(ENDED_AT,'yyyy-mm-dd hh24:mi:ss')
, trim(START_STATION_NAME)
, trim(START_STATION_ID)
, trim(END_STATION_NAME)
, trim(END_STATION_ID)
, to_number(START_LAT)
, to_number(START_LNG)
, to_number(END_LAT)
, to_number(END_LNG)
, trim(MEMBER_CASUAL)
from divvy_tripdata_ext
);

-- 5733451 rows inserted. -- check against previous total : OK

commit;

```

### Create a copy of the data for cleaning
```
CREATE TABLE CASESTUDY1.DIVVY_TRIPDATA_clean as select * from  divvy_tripdata;
```        

## Data cleaning

### Expected values OK?

```
Rideable_type:  
    select distinct rideable_type from divvy_tripdata; 
        electric_bike 
        classic_bike 
        docked_bike

Start_station_name and id:
    select start_station_id, count(start_station_name) from
	(
	    select start_station_id, start_station_name, count(*) from divvy_tripdata 
		group by start_station_id, start_station_name
	)
	group by start_station_id
	having count(start_station_name) > 1
	order by 1;
        1664 stations , but 322 stations with different values for start_station_id, 
		so will need to only work with the start_station_id, not the start_station_name (and presume the same for the end_stations)  

Latitudes and longitudes : do they fall withing Chicago range ?
    select * 
    from divvy_tripdata 
	where 
	start_lat not between  41.496235 and  42.338245
	or
	end_lat not between  41.496235 and  42.338245
	or
	start_lng not between -88.321761 and -87.415434
	or
	end_lng not between -88.321761 and -87.415434;
										 
	16 rows  to be deleted

        Decision was made to delete rows falling outside the boundaries if they represent a small number of the total population.

Member_casual:
    select distinct user_type from divvy_tripdata;
        Customer
        Subscriber
```


### Duplicate values ?

```
Ride_id: Unique due to successful created a PK constraint and index. (not shown)
Columns other than Ride_id: there were 31 rows with same content (except for PK), possibility of couple rides, so did not delete the data, also it's a small % of the total data (5m+ rows)
```

### Missing values ?
```
All columns except start_station and end_station columns had no missing values.
Start_station and End_station missing values: +- 15% of dataset.
It will have no impact on analysis, so not deleting them.
```

### Data ranges ?
```
Checked min and max dates : all withing expected date range.
Negative values in trip_duration: 79 rows deleted.
Ride_length_minutes: 5% of rows > 100 , deleted those. (64395 rows)
```

## Analysis

### Total count
```
select count(*) from  DIVVY_TRIPDATA_CLEAN;
	    5733356
```

### Member percentage
```
		select
			member_casual, count(*), round(count(*)/5733356*100,2) as percent 
		 from DIVVY_TRIPDATA_CLEAN
		 group by member_casual;
	 
		member    3386530    59.07
		casual    2346826    40.93
```    

### Averages/Medians/Totals
```
		-- Get averages/medians/totals
		SELECT 
         round(min(ride_length_minutes),2),
         round(max(ride_length_minutes),2), 
         round(avg(ride_length_minutes),2), 
         round(median(ride_length_minutes),0) 
        from DIVVY_TRIPDATA_CLEAN
		where ride_length_minutes > 0;

		0    100    14.44    10

		-- Get average by member type
		SELECT
		member_casual,
		 round(min(ride_length_minutes),3), 
         round(max(ride_length_minutes),3), 
         round(avg(ride_length_minutes),3), 
         round(median(ride_length_minutes),3)
		from DIVVY_TRIPDATA_CLEAN
		  group by member_casual
		;
		
		casual	0	100	18.245	13
		member	0	100	11.86	9  

 		-- Get average by member and day of the week.
		select day_of_week, decode(day_of_week,1,'Sun',2,'Mon',3,'Tue',4,'Wed',5,'Thu',6,'Fri',7,'Sat')
		, member_casual
		, round(avg(ride_length_minutes),2)
		from DIVVY_TRIPDATA_CLEAN
		group by day_of_week, 
				  member_casual
		order by 1  asc, 3 asc;
		
		1	Sun	casual	20.48
		1	Sun	member	13.03
		2	Mon	casual	18.48
		2	Mon	member	11.47
		3	Tue	casual	16.5
		3	Tue	member	11.29
		4	Wed	casual	16.13
		4	Wed	member	11.37
		5	Thu	casual	16.55
		5	Thu	member	11.51
		6	Fri	casual	17.33
		6	Fri	member	11.66
		7	Sat	casual	20.29
		7	Sat	member	13.16             



		-- Also get no. of rides into the mix.
		select day_of_week, decode(day_of_week,1,'Sun',2,'Mon',3,'Tue',4,'Wed',5,'Thu',6,'Fri',7,'Sat') as DayName
		, member_casual
		, round(avg(ride_length_minutes),2) as Average_Minutes
		, count(*) as No_of_rides
		from DIVVY_TRIPDATA_CLEAN
		group by day_of_week 
				  , member_casual          
		order by 1  asc, 3 asc;

		1	Sun	casual	20.48	380278
		1	Sun	member	13.03	389198
		2	Mon	casual	18.48	273121
		2	Mon	member	11.47	475829
		3	Tue	casual	16.5	258664
		3	Tue	member	11.29	517482
		4	Wed	casual	16.13	274376
		4	Wed	member	11.37	536693
		5	Thu	casual	16.55	307766
		5	Thu	member	11.51	539097
		6	Fri	casual	17.33	333472
		6	Fri	member	11.66	475706
		7	Sat	casual	20.29	463168
		7	Sat	member	13.16	444111
```        
## Export results to .csv files for use in Tableau.
