# Generating-a-week-number-for-any-period-of-time-
Generating a week number for any time period and for any number of years.

If necessary, get the number of the week in the year within a year, it is used the standart SQL query, like this:

SELECT
DATEPART(weekday, [Your date...]) AS WeekDays,
DATENAME(weekday, [Your date...]) AS WeekDaysName

  But if we need to get a sequence of week numbers over a year or between two years, there is a problem, because the last month of year has number=53, and the first month of next year has number=1. So it's got only one sequence from 1 to 53 by standart functions DATEPART, DATENAME. Then it is used script below to generate necessary sequence(It's used Function-date-generation dbo.GeneratingDates from previous repository). 

SET LANGUAGE ENGLISH
SET DATEFIRST 1

DECLARE @StartDate DATETIME2(0)='20161223',@endDate DATETIME2(0)='20310301'


DROP TABLE IF EXISTS #FirstStepOfResult
;WITH GenerateDates
AS
(
SELECT 
   t.dt
  ,t.WDay
  ,t.WDaysName
  ,t.Wnum
  ,YEAR(dt) AS Yr
  ,CASE WHEN t.WDay=1 AND t.Wnum=1 THEN 1 ELSE 0 END AS lab
FROM [ReportPR].[GeneratingDates] (@StartDate,@endDate) t
),
DetermineLabel
AS
(
SELECT 
   t.dt
  ,t.WDay
  ,t.WDaysName
  ,t.Wnum
  ,t.Yr
  ,MAX(t.lab) OVER(PARTITION BY t.Yr) AS lbel
FROM GenerateDates t
),
DetermineRanksOfYearOfDates
AS
(
SELECT 
  t.dt
 ,t.Yr
 ,DENSE_RANK() OVER(ORDER BY t.Yr) AS Rnk
FROM DetermineLabel t
WHERE t.lbel=1
),
LabelRanks
AS
(
SELECT 
   t.dt
  ,t.WDay
  ,t.WDaysName
  ,t.Wnum
  ,t.Yr
  ,t.lbel
  ,COALESCE(r.Rnk,0) AS Rnk
FROM DetermineLabel t
LEFT JOIN DetermineRanksOfYearOfDates r ON r.dt=t.dt
)
SELECT 
   t.dt
  ,t.WDay
  ,t.WDaysName
  ,t.Wnum
  ,t.Yr
  ,t.lbel
  ,t.Rnk
  INTO #FirstStepOfResult
FROM LabelRanks t
ORDER BY t.dt

----The last part of rows of column 'Rnk' where value=0 have to set right Rank too
----Below, there is the part of script

;WITH maxDateRank
AS
(
SELECT 
    MAX(t.dt) AS Mdt
  ,t.Rnk
FROM #FirstStepOfResult t
WHERE t.Rnk<>0
GROUP BY t.Rnk
),
DetermineRowsWithZeroRank
AS
(
SELECT TOP (1) WITH TIES
  t.dt
 ,d.Rnk
FROM #FirstStepOfResult t
INNER JOIN maxDateRank d ON t.dt>d.Mdt
WHERE t.Rnk=0
ORDER BY ROW_NUMBER() OVER(PARTITION BY t.dt ORDER BY d.Mdt DESC) 
)
UPDATE t 
  SET t.Rnk=r.Rnk
FROM #FirstStepOfResult t
INNER JOIN DetermineRowsWithZeroRank r ON r.dt=t.dt
-----------------------------------------------------------------------
SELECT 
  t.dt AS Dates
  ,t.WDay AS NumberOfDayInWeek
  ,t.WDaysName AS WeekdayName
  ,t.Wnum AS NumberOfWeekFromStandartSQLfunction
  ,CASE
     WHEN YEAR(@StartDate)<YEAR(@endDate) AND t.Yr>YEAR(@StartDate) THEN t.Wnum+53*(t.Yr-YEAR(@StartDate))-1*(t.Yr-YEAR(@StartDate))+t.Rnk
	 ELSE t.Wnum
   END AS NumberOfWeek_During_OverYear 
FROM #FirstStepOfResult t

The Result below:
Dates;	    NumberOfDayInWeek;	WeekdayName;	NumberOfWeekFromStandartSQLfunction	;NumberOfWeek_During_OverYear
2016-12-23	5	                  Friday	        52	                                52
2016-12-24	6	                  Saturday	        52	                                52
***
2016-12-30	5	                  Friday	        53	                                53
2016-12-31	6	                  Saturday	        53	                                53
2017-01-01	7                 	  Sunday	         1	                                53
2017-01-02	1	                  Monday	         2	                                54
2017-01-03	2	                  Tuesday	         2	                                54
2017-01-04	3	                  Wednesday	         2	                                54
2017-01-05	4	                  Thursday	         2	                                54
2017-01-06	5	                  Friday	         2	                                54
2017-01-07	6	                  Saturday	         2	                                54
2017-01-08	7	                  Sunday	         2	                                54
2017-01-09	1	                  Monday	         3	                                55
2017-01-10	2	                  Tuesday	         3	                                55
2017-01-11	3	                  Wednesday	         3	                                55
***
2017-12-27	3	                  Wednesday	        53	                               105
2017-12-28	4	                  Thursday	        53	                               105
2017-12-29	5	                  Friday	        53	                               105
2017-12-30	6	                  Saturday	        53	                               105
2017-12-31	7	                  Sunday	        53	                               105
2018-01-01	1	                  Monday	         1	                               106
2018-01-02	2	                  Tuesday	         1	                               106
2018-01-03	3	                  Wednesday	         1	                               106
2018-01-04	4	                  Thursday	         1	                               106
***
2018-12-28	5	                  Friday	        52	                               157
2018-12-29	6	                  Saturday	        52	                               157
2018-12-30	7	                  Sunday	        52	                               157
2018-12-31	1	                  Monday	        53	                               158
2019-01-01	2	                  Tuesday	         1	                               158
2019-01-02	3	                  Wednesday	         1	                               158
2019-01-03	4	                  Thursday	         1	                               158
2019-01-04	5	                  Friday	         1	                               158
2019-01-05	6	                  Saturday	         1	                               158
2019-01-06	7	                  Sunday	         1	                               158
2019-01-07	1	                  Monday	         2	                               159
2019-01-08	2	                  Tuesday	         2	                               159
***
2020-12-29	2	Tuesday	53	262
2020-12-30	3	Wednesday	53	262
2020-12-31	4	Thursday	53	262
2021-01-01	5	Friday	1	262
2021-01-02	6	Saturday	1	262
2021-01-03	7	Sunday	1	262
2021-01-04	1	Monday	2	263
2021-01-05	2	Tuesday	2	263
2021-01-06	3	Wednesday	2	263
2021-01-07	4	Thursday	2	263
***
2024-12-29	7	Sunday	52	470
2024-12-30	1	Monday	53	471
2024-12-31	2	Tuesday	53	471
2025-01-01	3	Wednesday	1	471
2025-01-02	4	Thursday	1	471
2025-01-03	5	Friday	1	471
2025-01-04	6	Saturday	1	471
2025-01-05	7	Sunday	1	471
2025-01-06	1	Monday	2	472
2025-01-07	2	Tuesday	2	472
***
2027-12-28	2	Tuesday	53	627
2027-12-29	3	Wednesday	53	627
2027-12-30	4	Thursday	53	627
2027-12-31	5	Friday	53	627
2028-01-01	6	Saturday	1	627
2028-01-02	7	Sunday	1	627
2028-01-03	1	Monday	2	628
2028-01-04	2	Tuesday	2	628
***

