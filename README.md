# Australian-Airlines-Dataset
Dataset and analysis of Australian airline operations for data analytics projects( Excel, PowerBI and SQL)  
australian-airlines-dataset/

├── data/                 # raw dataset files
│   └── australian_airlines.csv
├── powerbi/              # Power BI .pbix file
│
├── images/               # screenshots of dashboards/visuals
│   └── sample-dashboard.png
│
└── README.md             # project description



------ANALYZING AUSTRALIAN AIRLINES----
SELECT *
 FROM [Australian Airlines Analysis 1];

---Total Flights Scheduled Vs Flown by Year---
SELECT
Year,
SUM(Sectors_Scheduled) AS Total_Schedules,
SUM(Sectors_Flown) AS Total_Flown
FROM [Australian Airlines Analysis 1]
GROUP BY Year
ORDER BY Year;

---Top 5 Routes with Most Cancellations--
SELECT 
TOP 5 Route,
SUM(Cancellations) AS Total_Cancellations
FROM [Australian Airlines Analysis 1]
GROUP BY Route
ORDER BY Total_Cancellations DESC;

---On-Time Performance by Airline
WITH AirlineTotals AS(
SELECT Airline,
SUM(Departures_On_Time) AS Total_departures_on_time,
SUM(Arrivals_On_Time) AS Total_arrivals_on_time,
SUM(Sectors_Flown) AS Total_Flights
FROM [Australian Airlines Analysis 1] 
GROUP BY Airline) 
SELECT
Airline,
Total_departures_on_time,
Total_arrivals_on_time,
Total_Flights,
CAST(ROUND(1.0*Total_departures_on_time/NULLIF(Total_Flights,0),2) AS decimal(5,2)) * 100 AS departure_on_time_pct,
CAST(ROUND(1.0*Total_arrivals_on_time/NULLIF(Total_Flights,0),2) AS decimal(5,2)) * 100 AS arrivals_on_time_pct
FROM AirlineTotals
ORDER BY arrivals_on_time_pct DESC;

---Monthly Flight Trends Per Year--
SELECT 
Year,
month_Num,
SUM(Sectors_Flown) AS Total_Flights
FROM [Australian Airlines Analysis 1]
GROUP BY Year,Month_Num
ORDER BY Year,Month_Num;

---Delay Statistics by Port---(Determine routes or ports with frequent delays)
SELECT Departing_Port,
SUM(Departures_Delayed) AS Total_departure_delays,
SUM(Arrivals_Delayed) AS Total_arrivals_delays
FROM [Australian Airlines Analysis 1]
GROUP BY Departing_Port
ORDER BY Total_departure_delays;

---Best Performing Route(Least Delays)
SELECT TOP 1 Route,
SUM(Arrivals_Delayed) AS Total_delayed,
SUM(Sectors_Flown) AS Total_Flights,
ROUND(SUM(Arrivals_Delayed)*100.0/NULLIF(SUM(Sectors_Flown),0),2) AS delay_rate_pct
FROM [Australian Airlines Analysis 1]
Group BY Route
ORDER BY delay_rate_pct ASC; 

----Scalar Function that Calculate the on_time performance percentage for Qantas in 2023
GO
CREATE FUNCTION dbo.fn_OnTimePerformance(
@Airline NVARCHAR(100),
@Year INT)
RETURNS DECIMAL(5,2)
AS 
BEGIN
DECLARE @OnTimePct DECIMAL(5,2);
SELECT @OnTimePct = 
CAST(SUM(arrivals_On_Time) * 100.0 / NULLIF(SUM(Sectors_Flown),0) AS DECIMAL(5,2))
FROM [Australian Airlines Analysis 1]
WHERE Airline = @Airline
AND Year = @Year;
RETURN @OnTimePct;
END;
GO


SELECT dbo.fn_OnTimePerformance('Qantas',2023) AS OnTimePct;


---Store Procedure that summarizes the Airlines total flight,total cancellations, departure on time pct,arrival on time pct.

GO
CREATE PROCEDURE dbo.sp_AirlinePerformanceSummary
    @Year INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        airline,
        SUM(Sectors_Flown) AS total_flights,
        SUM(Cancellations) AS total_cancellations,
        CAST(SUM(Departures_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS departure_on_time_pct,
        CAST(SUM(Arrivals_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS arrival_on_time_pct
    FROM [Australian Airlines Analysis 1]
    WHERE year = @Year
    GROUP BY airline
    ORDER BY arrival_on_time_pct DESC;
END;
GO

EXEC dbo.sp_AirlinePerformanceSummary @Year = 2023;





CREATE FUNCTION dbo.fn_AirlinePerformance
(
    @Airline NVARCHAR(100),
    @Year INT
)
RETURNS TABLE
AS
RETURN
(
    SELECT 
        airline,
        @Year AS report_year,
        SUM(sectors_flown) AS total_flights,
        SUM(Cancellations) AS total_cancellations,
        CAST(SUM(Departures_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS departure_on_time_pct,
        CAST(SUM(Arrivals_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS arrival_on_time_pct,
        CAST(SUM(Departures_Delayed) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS departure_delayed_pct,
        CAST(SUM(arrivals_delayed) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS arrival_delayed_pct
    FROM [Australian Airlines Analysis 1]
    WHERE airline = @Airline
      AND year = @Year
    GROUP BY airline
);


SELECT *
FROM dbo.fn_AirlinePerformance('Qantas',2025);


-- SP: Route-level performance summary
GO
CREATE PROCEDURE dbo.sp_RouteSummary
    @Year INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        airline,
        departing_port,
        arriving_port,
        departing_port + ' → ' + arriving_port AS route,
        @Year AS report_year,
        SUM(sectors_flown) AS total_flights,
        SUM(Cancellations) AS total_cancellations,
        CAST(SUM(Departures_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS departure_on_time_pct,
        CAST(SUM(Arrivals_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS arrival_on_time_pct,
        CAST(SUM(Departures_Delayed) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS departure_delayed_pct,
        CAST(SUM(arrivals_delayed) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS arrival_delayed_pct
    FROM [Australian Airlines Analysis 1]
    WHERE year = @Year
    GROUP BY airline, departing_port, arriving_port
    ORDER BY arrival_on_time_pct DESC;
END;
GO

EXEC dbo.sp_RouteSummary @Year = 2025;

GO
ALTER FUNCTION dbo.fn_AirlineRoutePerformance
(
    @Airline NVARCHAR(100),
    @Year INT,
    @DepartingPort NVARCHAR(50) = NULL,   
    @ArrivingPort NVARCHAR(50) = NULL     
)
RETURNS TABLE
AS
RETURN
(
    SELECT 
        route,
        departing_port,
        arriving_port,
        airline,
        year,
        SUM(sectors_scheduled) AS total_scheduled,
        SUM(sectors_flown) AS total_flown,
        SUM(Cancellations) AS total_cancellations,
        SUM(Departures_On_Time) AS total_departures_on_time,
        SUM(Departures_Delayed) AS total_departures_delayed,
        SUM(Arrivals_On_Time) AS total_arrivals_on_time,
        SUM(Arrivals_Delayed) AS total_arrivals_delayed,
        CAST(SUM(Arrivals_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS arrival_on_time_pct,
        CAST(SUM(Departures_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS departure_on_time_pct
    FROM [Australian Airlines Analysis 1]
    WHERE airline = @Airline
      AND year = @Year
      AND (@DepartingPort IS NULL OR departing_port = @DepartingPort)
      AND (@ArrivingPort IS NULL OR arriving_port = @ArrivingPort)
    GROUP BY route, departing_port, arriving_port, airline, year
);
GO
IF OBJECT_ID('dbo.fn_AirlineRoutePerformance', 'IF') IS NOT NULL
    DROP FUNCTION dbo.fn_AirlineRoutePerformance;
GO
CREATE FUNCTION dbo.fn_AirlineRoutePerformance
(
    @Airline NVARCHAR(100),
    @Year INT,
    @DepartingPort NVARCHAR(50) = NULL,   
    @ArrivingPort NVARCHAR(50) = NULL     
)
RETURNS TABLE
AS
RETURN
(
    SELECT 
        route,
        departing_port,
        arriving_port,
        airline,
        year,
        SUM(sectors_scheduled) AS total_scheduled,
        SUM(sectors_flown) AS total_flown,
        SUM(Cancellations) AS total_cancellations,
        SUM(Departures_On_Time) AS total_departures_on_time,
        SUM(Departures_Delayed) AS total_departures_delayed,
        SUM(Arrivals_On_Time) AS total_arrivals_on_time,
        SUM(Arrivals_Delayed) AS total_arrivals_delayed,
        CAST(SUM(Arrivals_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS arrival_on_time_pct,
        CAST(SUM(Departures_On_Time) * 100.0 / NULLIF(SUM(sectors_flown),0) AS DECIMAL(5,2)) AS departure_on_time_pct
    FROM [Australian Airlines Analysis 1]
    WHERE airline = @Airline
      AND year = @Year
      AND (@DepartingPort IS NULL OR departing_port = @DepartingPort)
      AND (@ArrivingPort IS NULL OR arriving_port = @ArrivingPort)
    GROUP BY route, departing_port, arriving_port, airline, year
);
GO

-- 1. All routes for Qantas in 2023--

sp_helptext 'dbo.fn_AirlineRoutePerformance';
-- 1. All routes for Qantas in 2023
SELECT * 
FROM dbo.fn_AirlineRoutePerformance('Qantas', 2023, NULL, NULL);

-- 2. A specific route for Virgin in 2023 (BNE → SYD)
SELECT * 
FROM dbo.fn_AirlineRoutePerformance('Virgin', 2023, 'BNE', 'SYD');

-- 3. All routes for Jetstar in 2022
SELECT * 
FROM dbo.fn_AirlineRoutePerformance('Jetstar', 2022, NULL, NULL);

---Cancellation Rate by Airline
SELECT 
Airline,
SUM(Cancellations) AS Total_Cancellations,
SUM(Sectors_Scheduled) AS Total_Scheduled,
ROUND(SUM(Cancellations) * 100.0/ NULLIF(SUM(Sectors_Scheduled),0),2) AS Cancellations_Rate_Pct
FROM [Australian Airlines Analysis 1]
GROUP BY Airline
ORDER BY Cancellations_Rate_Pct DESC;

---Busiest Routes by Month (Top 5 Each Month)
;WITH RankedRoutes AS (
SELECT
Year,
Month_Num,
Route,
SUM(Sectors_Flown) AS Total_Flights,
RANK() OVER (PARTITION BY Year,Month_Num ORDER BY SUM(Sectors_Flown)DESC) AS Route_Rank
FROM [Australian Airlines Analysis 1]
GROUP BY Year,Month_Num,Route)
SELECT *
FROM RankedRoutes
WHERE Route_Rank <= 5
ORDER BY Year,Month_Num,Route_Rank;

---Seasonal Delay Patterns(Quanterly Trends)
SELECT 
Year,
CASE 
WHEN Month_Num BETWEEN 1 AND 3 THEN 'Q1'--- let say quarter 1
WHEN Month_Num BETWEEN 4 AND 6 THEN 'Q2'--- '' '' quarter 2
WHEN Month_Num BETWEEN 7 AND 9 THEN 'Q3'---- '' '' quarter 3
ELSE 'Q4' ----------------------------------------- quarter 4 
END AS quarter,
SUM(Departures_Delayed) AS Total_departures_delayed,
SUM(Arrivals_Delayed) AS Total_arrivals_delayed
FROM [Australian Airlines Analysis 1]
GROUP BY Year,
CASE 
WHEN Month_Num BETWEEN 1 AND 3 THEN 'Q1'
WHEN Month_Num BETWEEN 4 AND 6 THEN 'Q2'
WHEN Month_Num BETWEEN 7 AND 9 THEN 'Q3'
ELSE 'Q4'
END 
ORDER BY Year, quarter;

 
----Airline with best on time performance by year---
;WITH bestairline AS (
SELECT Year, 
Airline,
ROUND(SUM(Arrivals_On_Time)* 100.0/ NULLIF(SUM(Sectors_Flown),0),2) AS Arrivals_On_Time_Pct,
RANK() OVER (PARTITION BY Year ORDER BY ROUND(SUM(Arrivals_On_Time)*  100.0/NULLIF(SUM(Sectors_Flown),0),2) DESC) AS Performance_Rank
FROM [Australian Airlines Analysis 1]
GROUP BY Year, Airline)
SELECT 
Year,
Airline,
Arrivals_On_Time_Pct,
Performance_Rank
FROM bestairline
WHERE Performance_Rank = 1--just want to get the top ranked airline for each year by on time performance only
ORDER BY Year;

---Month Over Month Change in Flight Volume 
;WITH Monthly_Totals As (SELECT Year,
Month_Num,
SUM(Sectors_Flown) AS Total_Flights
FROM [Australian Airlines Analysis 1]
GROUP BY Year,Month_Num)
 SELECT Year,
 Month_Num,
 Total_Flights,
 LAG(Total_Flights) OVER (ORDER BY Year,Month_Num) AS Previous_month_flights,
 ROUND((Total_Flights - LAG(Total_Flights) OVER (ORDER BY Year,Month_Num)) * 100.0/NULLIF(LAG(Total_Flights) OVER (ORDER BY Year,Month_Num),0),2) AS mom_growth_pct-------Month Over month growth percentage
 FROM Monthly_Totals
 ORDER BY Year,Month_Num;--------(Not really satified with output... i need some like "Growth" vs Declined in Flight flown when compare with current vs previous flight)


  WITH Monthly_totals AS (
  SELECT Year,
  Month_Num,
  SUM(Sectors_Flown) AS Total_Flights
  FROM [Australian Airlines Analysis 1]
  GROUP BY Year,Month_Num)
  SELECT Year,
  Total_Flights,
  LAG(Total_Flights) OVER (ORDER BY Year, Month_Num) AS Previous_Month_Flights,
  ROUND((Total_Flights -LAG(Total_Flights) OVER (ORDER BY Year,Month_Num)) * 100.0/NULLIF(LAG(Total_Flights) OVER (ORDER BY Year, Month_Num),0),2) AS Mom_growth_Pct,
  CASE 
  WHEN LAG(Total_Flights) OVER (ORDER BY Year, Month_Num ) IS NULL THEN 'N/A'
       WHEN (Total_Flights - LAG(Total_Flights) OVER (ORDER BY Year,Month_Num)) > 0 THEN 'Growth'
       WHEN  (Total_Flights - LAG(Total_Flights) OVER (ORDER BY Year,Month_Num)) <0 THEN 'Decline'
       ELSE 'No Change'
       END AS Growth_Status
       FROM Monthly_totals
       ORDER BY Year,
       Month_Num; ------- (This is what i want so that anyone can read my queries and for better insights too)

 ---Most Relaible Port(Lowest Average Delay Rate)--------
SELECT TOP 5
Departing_Port,
SUM(Sectors_Flown) AS Total_Flight,
SUM(Departures_Delayed + Arrivals_Delayed) AS Total_delays,
ROUND(SUM(Departures_Delayed + Arrivals_Delayed) * 100.0/NULLIF(SUM(Sectors_Flown),0),2) AS Avg_delay_rate_pct
FROM [Australian Airlines Analysis 1]
GROUP BY Departing_Port
ORDER BY Avg_delay_rate_pct ASC;

----Yearly Airline Market Share(Based on Flights Flown)
WITH Airline_totals AS (
SELECT 
Year,
Airline,
SUM(Sectors_Flown) AS Airline_Flights
FROM [Australian Airlines Analysis 1]
GROUP BY Year,Airline
),
year_totals AS (
SELECT Year,
SUM(Airline_Flights) AS Total_year_flights
FROM Airline_totals
GROUP BY Year
),
Market_share AS(
SELECT a.year,
a.airline,
a.airline_flights,
y.Total_year_flights,
ROUND(a.Airline_Flights * 100.0 / NULLIF(y.Total_year_flights,0),2) AS Market_share_pct
FROM Airline_totals a 
INNER JOIN year_totals y
ON a.Year = y.Year
)
SELECT 
Year,
Airline,
Airline_flights,
Total_year_flights,
Market_share_pct,
RANK() OVER ( PARTITION BY Year ORDER BY Market_share_pct DESC) AS Airline_rank
FROM Market_share
ORDER BY Year, Airline_rank;



