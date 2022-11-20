# Analysis of E_Commerce_delivery Database

A database is an organized collection of structured information, or data, typically stored electronically in a computer system. A database is usually controlled by a database management system (DBMS). These data are stored in rows and columns and this data can be retrived by using Structured Query Language (SQL)
This is a `Relational Database` which means that the data that are stored in a rows and columns and the tables are connected to each other with the help of `primary key` and `foreign key` constraints.

**The following diagram shows the stucture of the database**
 ![Database structure](Screenshot.png)

**Description of the data**
 There are 8 tables in the Database which are related to each other with the help of primary key and foreign key
 The tables are :
 * Customers 
 * Geolocation
 * Order Items 
 * Payments 
 * Reviews 
 * Orders 
 * Products 
 * Sellers 

The data of the each table were in CSV(`Comma seperated values`). So it was imported to the Sql envirnment and the analysis is done. Since size of the data is very large Google Sandbox Big Query is used since the processing speed is faster

## Concepts used

To analyze the data I have used `DML` command to retrieve the useful data from the database which include,

* The basic clauses like `SELECT`, `JOIN`, `GROUP BY`, `ORDER BY`

* Analytical functions like `AGGREGATION FUNCTIONS` and `WINDOW FUNCTIONS`


### Analysis

Getting  to know about the datatypes of each table

``` sql
select 
    column_name, data_type
from 
    `E_commerce.INFORMATION_SCHEMA.COLUMNS`
where 
    table_name = 'customers'
order by 
    ordinal_position
```
![Customer_table](Customers_Table.png) ![Geolocation_table](Geolocation_table.png) ![Order_Item_table](Orders_Item_table.png)
![Ordre_table](orders_table.png) ![Payment_table](payments_table.png) ![Database structure](Screenshot.png) 
![Produts_table](products_table.png) ![Sellers_table](sellers_table.png)





In the database, identify the month wise highest paying passenger name and passenger id
``` sql
select 
	a.month_name as Month_name,
	a.passenger_id as Passenger_id,
	a.Passenger_name as Passenger_name,
	a.amount as amount
from
	(select
		to_char(book.book_date,'Mon-yy') as month_name,
		tic.passenger_id as passenger_id,
		tic.passenger_name as passenger_name,
		sum(book.total_amount) as amount,
		rank() over(partition by to_char(book.book_date,'Mon-yy') order by sum(book.total_amount) desc) as rnk
	from 
		bookings.bookings book join bookings.tickets tic
		on book.book_ref = tic.book_ref
		group by month_name,passenger_id,passenger_name)a
where rnk = 1
```
In the database, identify the month wise least paying passenger name and passenger id?
```sql 
select 
	a.month_name as Month_name,
	a.passenger_id as Passenger_id,
	a.Passenger_name as Passenger_name,
	a.amount as amount
from
	(select
		to_char(book.book_date,'Mon-yy') as month_name,
		tic.passenger_id as passenger_id,
		tic.passenger_name as passenger_name,
		sum(book.total_amount) as amount,
		rank() over(partition by to_char(book.book_date,'Mon-yy') order by sum(book.total_amount)) as rnk
	from 
		bookings.bookings book join bookings.tickets tic
		on book.book_ref = tic.book_ref
		group by month_name,passenger_id,passenger_name)a
where rnk = 1
```
Identify the travel details of non no stop journeys  or return journeys (having more than 1 flight.
``` sql
select 
	tic.passenger_id as passenger_id,
	tic.passenger_name as passenger_name,
	tic.ticket_no as ticket_no,
	count(tf.flight_id) as flight_count
from
	bookings.tickets tic join bookings.ticket_flights tf
	on tic.ticket_no = tf.ticket_no
group by
	 tic.passenger_name,tic.passenger_id, tic.ticket_no
having 
	count(tic.ticket_no) > 1
order by
	 count(tf.flight_id) desc
```
How many tickets are there without boarding passes?
``` sql 
select
((select count(ticket_no) from bookings.ticket_flights) - 
 (select count(ticket_no) from bookings.boarding_passes)) as not_having_boarding_passes
```
Details of the longest flight
``` sql 
select 
	*,
	(actual_arrival - actual_departure) as duration
from 
	bookings.flights
where 
	(actual_arrival - actual_departure) = (select 
					     		max((actual_arrival - actual_departure))
					       from 
							bookings.flights)
```
Categorizing the flights by the time of their departure
a.Early morning flights: 2 AM to 6AM
	b.Morning flights: 6 AM to 11 AM
	c.Noon flights: 11 AM to 4 PM
	d.Evening flights: 4 PM to 7 PM
	e.Night flights: 7 PM to 11 PM 
	f.Late Night flights: 11 PM to 2 
``` sql 
select 
	flight_id,
	flight_no,
	scheduled_departure,
	scheduled_arrival,
case
	when cast(to_char(scheduled_departure, 'HH24:MI') as TIME) between '19:00:00' and '23:00:00' then 'Night_Flight'
	when cast(to_char(scheduled_departure, 'HH24:MI') as TIME) between '2:00:00' and '6:00:00' then 'Early_Morning_Flight'
	when cast(to_char(scheduled_departure, 'HH24:MI') as TIME) between '6:00:00' and '11:00:00' then 'Morning_Flight'
	when cast(to_char(scheduled_departure, 'HH24:MI') as TIME) between '11:00:00' and '16:00:00' then 'Noon_Flight'
	when cast(to_char(scheduled_departure, 'HH24:MI') as TIME) between '16:00:00' and '19:00:00' then 'Evening_Flight'
	else 'Late_Night_Flight'
 end as timings
from 
	bookings.flights
```
Details of all the morning flights
``` sql
select
	 a.*
from
	(select *,
	case 
		when cast(to_char(scheduled_departure, 'HH24:MI') as TIME) between '6:00:00' and '11:00:00' then 'Morning_Flight'
 	else 
		'other'
	end as timings
from 
	bookings.flights)a
where 
	a.timings = 'Morning_Flight'
```
The earliest morning flight available from every airport
``` sql
select 
	b.flight_id,
	b.flight_no,
	b.scheduled_departure,
	b.scheduled_arrival,
	b.departure_airport,
	b.timings
from
	(select 
		a.*,
		rank() over(partition by departure_airport order by scheduled_departure asc) as rnk
	from(
		select flight_id,
		flight_no,
		scheduled_departure,
		scheduled_arrival,
		departure_airport,
	case 
		when cast(to_char(scheduled_departure, 'HH24:MI') as TIME) between '6:00:00' and '11:00:00' then 'Morning_Flight'
	else 
		'other'
	end as timings
	from 
		bookings.flights)a
where 
	a.timings = 'Morning_Flight')b
where
	 b.rnk = 1
```
Count of seats in various fare condition for every aircraft code
``` sql
select 	
	aircraft_code,
	fare_conditions,
	count(seat_no) as seats
from 
	bookings.seats
group 
	by aircraft_code, fare_conditions
order 
	by aircraft_code ;
```
Aircrafts codes that have at least one Business class seats
``` sql
with cte as 
(select 
	aircraft_code
from 	
	bookings.seats
group by
	aircraft_code, fare_conditions
having 
	count(seat_no) >= 1 and fare_conditions = 'Business')
	
select 
	count(*) as count_of_aircraft_code_with_business_class
from 
	cte ;
```
Name of the airport having maximum number of departure flight
``` sql
select 
	a.departure_airport
from
	(select
		departure_airport,
		count(actual_departure) as num_of_flights,
		rank() over(order by count(actual_departure) desc) as count_rank
	from 
		bookings.flights
	group by
		departure_airport)a
where 
	a.count_rank = 1 ;
```
Name of the airport having least number of scheduled departure flights
``` sql
with cte as
	(select
		departure_airport,
		count(actual_departure) as num_of_flights,
		rank() over(order by count(actual_departure)) as count_rank
	from 
		bookings.flights
	group by
		departure_airport)

select 
	departure_airport
from 
	cte
where 
	count_rank = 1 ;
```
Airport(name) has most cancelled flights (arriving)
``` sql
select
	a.airport_name
from
	(select 
		(ad.airport_name ->> 'en') as airport_name,
		count(ad.airport_name),
		rank() over (order by count(ad.airport_name) desc) as rnk
	from 
		bookings.airports_data ad join bookings.flights f
		on f.arrival_airport = ad.airport_code
	group by 
		ad.airport_name,f.status
	having 
		f.status = 'Cancelled')a
where 
	a.rnk = 1 ;
```
Flight ids which are using “Airbus aircrafts”
``` sql 
select
	a.flight_id
from 
	(select 
		f.flight_id,
		split_part((model ->> 'en'),' ',1) as model
	from 
		bookings.flights f join bookings.aircrafts_data ad
		on f.aircraft_code = ad.aircraft_code)a
where 
	a.model = 'Airbus' ;
```
Date-wise last flight id flying from every airport
``` sql
select 
	a.departure_airport,
	a.date as date,
	a.flight_id as flight_id
from
	(select
	 	departure_airport,
		flight_id,
		scheduled_departure as date,
		rank() over (partition by departure_airport,cast(scheduled_departure as date)  order by scheduled_departure desc) as rnk
	from 
		bookings.flights)a
where 
	a.rnk = 1
order by 
	a.date ;
```
Date wise first cancelled flight id flying for every airport
``` sql
select
	a.departure_airport as departure_airport,
	a.dte as date, 
	a.flight_id as flight_id
from
	(select
	 	departure_airport,
		flight_id,
		scheduled_departure as dte,
		rank() over (partition by departure_airport,cast(scheduled_departure as date)  order by scheduled_departure) as rnk
	from 
		bookings.flights
	where 
		status = 'Cancelled')a
where 
	a.rnk = 1
order by 
	a.dte ;
```
Flight ids having highest range
``` sql
with code as ((select 
			aircraft_code 
		from 
			bookings.aircrafts_data
		where 
			range = (select 
					max(range) 
	       			 from 
					bookings.aircrafts_data)))

select 
	distinct flight_id 
from 
	bookings.flights
where 
	aircraft_code = (select * from code)
```