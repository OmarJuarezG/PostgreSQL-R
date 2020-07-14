# PostgreSQL-R
Uses DVD rental database to develop an interactive document and an app. To do this we connect R with a Postgres database.
## Analyzing the data (basic analysis)
By exploring the data I've found the following:
1. Not all films are present in the inventory. Since they're not present in the inventory the company can not rent them. The code that allows to find this is the following:
```sql
SELECT
	film.film_id              AS film_id,
	inventory.inventory_id    AS inventory_id,
	DATE(rental.rental_date)  AS rental_date,
	rental.rental_id          AS rental_id,
	category.name             AS category,
	payment.amount            AS amount
FROM 
	film
LEFT JOIN
	inventory
ON
	film.film_id = inventory.film_id
LEFT JOIN
	rental
ON
	rental.inventory_id = inventory.inventory_id
LEFT JOIN
	payment
ON
	rental.rental_id = payment.rental_id
LEFT JOIN
	film_category
ON
	film.film_id = film_category.film_id
LEFT JOIN
	category
ON
	film_category.category_id = category.category_id
WHERE 
	inventory.inventory_id IS NULL
```

![Query output](/OutputAnalysis_1.PNG)

2. Some records have missing values. For example, there are movies that are in the inventory and were rented but do not have payment_id, payment_date or customer associated with the rent. There might be some data corruption in those cases. The code to find this is the following:
```sql
SELECT 
  DATE(rental.rental_date)   AS rental_date,
  rental.rental_id           AS rental_id,
  category.name              AS category,
  payment.payment_id         AS payment_id,
  customer.customer_id       AS customer_id,
  payment.amount             AS amount
FROM 
	film
LEFT JOIN
	inventory
ON
	film.film_id = inventory.film_id
LEFT JOIN
	rental
ON
	rental.inventory_id = inventory.inventory_id
LEFT JOIN
	payment
ON
	rental.rental_id = payment.rental_id
LEFT JOIN
	customer
ON
	payment.customer_id = customer.customer_id
LEFT JOIN
	film_category
ON
	film.film_id = film_category.film_id
LEFT JOIN
	category
ON
	film_category.category_id = category.category_id
WHERE
	payment.payment_id IS NULL AND
	customer.customer_id IS NULL
```
![Query output](/OutputAnalysis_2.PNG)

## Questions to solve and answers
1. Is there any particular actor/actress that is more profitable in terms of movie rents? Perhaps the company could make an add featuring prominent actors so it can boost theirs rents and by doing so its revenues.
```sql
SELECT
	actor.first_name||' '||actor.last_name AS actor_name,
	SUM(payment.amount)                    AS amount
FROM
	actor
JOIN
	film_actor
ON
	actor.actor_id = film_actor.actor_id
JOIN
	film
ON
	film_actor.film_id = film.film_id
JOIN
	inventory
ON
	film.film_id = inventory.film_id
JOIN
	rental
ON
	inventory.inventory_id = rental.inventory_id
JOIN
	payment
ON
	rental.rental_id = payment.rental_id
GROUP BY
	actor.first_name||' '||actor.last_name
ORDER BY
	SUM(payment.amount) DESC
```

Answer: Susan Davis, Gina Degeneres and Matthew Carrey could do a commercial as an attempt to boost sales.

![Query output](/AnswerQuestion1.PNG)

2. Is the rating of the film important to the revenues? Perhaps the company could shift its attention to a more profitable market instead of having all markets.

```sql
SELECT
	film.rating                                  AS film_rating,
	COUNT(DISTINCT customer.customer_id)         AS rents,
	ROUND(SUM(payment.amount))                   AS revenue
FROM
	actor
JOIN
	film_actor
ON
	actor.actor_id = film_actor.actor_id
JOIN
	film
ON
	film_actor.film_id = film.film_id
JOIN
	inventory
ON
	film.film_id = inventory.film_id
JOIN
	rental
ON
	inventory.inventory_id = rental.inventory_id
JOIN
	payment
ON
	rental.rental_id = payment.rental_id
JOIN
	customer
ON
	payment.customer_id = customer.customer_id
GROUP BY
	film.rating
ORDER BY
	SUM(payment.amount) DESC

```
It seems that rating overall is well distributed in rents but there is a little spread between the revenues. Comparing the PG-13 and G, there is a difference of 14,544 in revenue.

![Query output](/AnswerQuestion2.PNG)

3. What are the top and least rented movies based on categories and their total revenues? (*by Okoh Anita in freeCodeCamp*)
```sql
SELECT
	category.name                         AS category,
	COUNT (DISTINCT customer.customer_id) AS rents,
	SUM(payment.amount)                   AS amount
FROM
	category
JOIN
	film_category
ON
	category.category_id = film_category.category_id
JOIN
	film
ON
	film_category.film_id = film.film_id
JOIN
	inventory
ON
	film.film_id = inventory.film_id
JOIN
	rental
ON
	rental.inventory_id = inventory.inventory_id
JOIN
	payment
ON
	rental.rental_id = payment.rental_id
JOIN
	customer
ON
	payment.customer_id = customer.customer_id
GROUP BY
	category.name
ORDER BY
	COUNT (customer.customer_id) DESC

```
![Query output](/AnswerQuestion3.PNG)

4. Which are the most relevant countries in terms on rents and revenue for the company? Maybe we could reinforced those markets instead of spreading resources in markets that are not profitable.

```sql
SELECT
	country.country                       AS country,
	COUNT (DISTINCT customer.customer_id) AS demand,
	SUM(payment.amount)                   AS revenue
FROM
	category
JOIN
	film_category
ON
	category.category_id = film_category.category_id
JOIN
	film
ON
	film_category.film_id = film.film_id
JOIN
	inventory
ON
	film.film_id = inventory.film_id
JOIN
	rental
ON
	rental.inventory_id = inventory.inventory_id
JOIN
	payment
ON
	rental.rental_id = payment.rental_id
JOIN
	customer
ON
	payment.customer_id = customer.customer_id
JOIN
	address
ON
	customer.address_id = address.address_id
JOIN
	city
ON
	address.city_id = city.city_id
JOIN
	country
ON
	city.country_id = country.country_id
GROUP BY
	country.country
HAVING
	COUNT (DISTINCT customer.customer_id) >= 10
ORDER BY
	SUM(payment.amount) DESC
```
There are a lot of countries in where the demand is very low even having just one client renting movies. Demand >= 10 could be a treshold to evaluate future policies.

![Query output](/AnswerQuestion4.PNG)

5. How the rents have behaved per month based on movie category? Could the rents be seasonal? Obtain the values just for the year 2005
```sql

SELECT
	rent_per_day_table.rental_date,
	rent_per_day_table.movie_category,
	rent_per_day_table.total_rents_per_day,
	SUM(rent_per_day_table.total_rents_per_day) OVER 
		(PARTITION BY rent_per_day_table.movie_category ORDER BY rent_per_day_table.rental_date) AS cum_rents
FROM(
	SELECT
		temp_rent_table.rental_date AS rental_date,
		temp_rent_table.category AS movie_category,
		COUNT (DISTINCT temp_rent_table.rental_id) AS total_rents_per_day,
		SUM(temp_rent_table.amount) AS total_revenue
	FROM(
		SELECT 
			DATE(rental.rental_date) AS rental_date,
			rental.rental_id AS rental_id,
			category.name AS category,
			payment.amount AS amount
		FROM 
			film
		JOIN
			inventory
		ON
			film.film_id = inventory.film_id
		JOIN
			rental
		ON
			rental.inventory_id = inventory.inventory_id
		JOIN
			payment
		ON
			rental.rental_id = payment.rental_id
		JOIN
			film_category
		ON
			film.film_id = film_category.film_id
		JOIN
			category
		ON
			film_category.category_id = category.category_id
		WHERE 
			inventory.inventory_id IS NOT NULL AND
			rental.rental_id IS NOT NULL AND
			EXTRACT (YEAR FROM rental_date) = 2005
	) temp_rent_table
	GROUP BY
		temp_rent_table.rental_date,
		temp_rent_table.category
) rent_per_day_table

```
![Query output](/AnswerQuestion5.PNG)

6. How the renvenues have behaved per month based on movie category? This will have a high correlation with the results of previous question.

```sql

SELECT
	rent_per_day_table.rental_date,
	rent_per_day_table.movie_category,
	rent_per_day_table.total_rents_per_day,
	SUM(rent_per_day_table.total_revenue) OVER 
		(PARTITION BY rent_per_day_table.movie_category ORDER BY rent_per_day_table.rental_date) AS cum_revenue
FROM(
	SELECT
		temp_rent_table.rental_date AS rental_date,
		temp_rent_table.category AS movie_category,
		COUNT (DISTINCT temp_rent_table.rental_id) AS total_rents_per_day,
		SUM(temp_rent_table.amount) AS total_revenue
	FROM(
		SELECT 
			DATE(rental.rental_date) AS rental_date,
			rental.rental_id AS rental_id,
			category.name AS category,
			payment.amount AS amount
		FROM 
			film
		JOIN
			inventory
		ON
			film.film_id = inventory.film_id
		JOIN
			rental
		ON
			rental.inventory_id = inventory.inventory_id
		JOIN
			payment
		ON
			rental.rental_id = payment.rental_id
		JOIN
			film_category
		ON
			film.film_id = film_category.film_id
		JOIN
			category
		ON
			film_category.category_id = category.category_id
		WHERE 
			inventory.inventory_id IS NOT NULL AND
			rental.rental_id IS NOT NULL AND
			EXTRACT (YEAR FROM rental_date) = 2005
	) temp_rent_table
	GROUP BY
		temp_rent_table.rental_date,
		temp_rent_table.category
) rent_per_day_table

```
![Query output](/AnswerQuestion6.PNG)

7. If the company wants to reward premium users, it needs to identify their top 20. For this the company might need the customer's details.

```sql
SELECT
	customer.first_name || ' ' || customer.last_name AS customer_name,
	SUM(payment.amount)                              AS total_payment,
	customer.email                                   AS email,
	address.address                                  AS address,
	address.phone                                    AS phone,
	city.city                                        AS city,
	country.country                                  AS country
FROM
	customer
JOIN
	payment
ON
	customer.customer_id = payment.customer_id
JOIN
	address
ON
	customer.address_id = address.address_id
JOIN
	city
ON
	address.city_id = city.city_id
JOIN
	country
ON
	city.country_id = country.country_id
GROUP BY
	customer.first_name || ' ' || customer.last_name,
	customer.email,
	address.address,
	address.phone,
	city.city,
	country.country
ORDER BY
	SUM(payment.amount) DESC
LIMIT 20
```
![Query output](/AnswerQuestion7.PNG)

8. How many loses or replacement cost the company is incurring by clients that are not returning the rented films? Hint: rental_duration gives the number of days the film can be rented.
```sql
SELECT
	SUM(film.replacement_cost) AS incurring_costs
FROM
	film
JOIN
	inventory
ON
	film.film_id = inventory.inventory_id
JOIN
	rental
ON
	inventory.inventory_id = rental.inventory_id
JOIN
	customer
ON
	rental.customer_id = customer.customer_id
WHERE
	rental.return_date IS NULL

```
The cost is 911.55.

![Query output](/AnswerQuestion8.PNG)

9. What is the average rental rate for each category? (*by Okoh Anita in freeCodeCamp*)

```sql
SELECT
	category.name                  AS category,
	ROUND(AVG(film.rental_rate),2) AS average_rental_rate
FROM
	film
JOIN
	film_category
ON
	film.film_id = film_category.film_id
JOIN
	category
ON
	film_category.category_id = category.category_id
GROUP BY
	category.name
ORDER BY
	ROUND(AVG(film.rental_rate),2) DESC

```

![Query output](/AnswerQuestion9.PNG)

10. How many films were returned in time, late or never returned? (*by Okoh Anita in freeCodeCamp with modification*)
```sql
SELECT
	returned_days.return_description                    AS return_description,
	COUNT(returned_days.inventory_id)                   AS number_of_films
FROM(

SELECT
	inventory.inventory_id                              AS inventory_id,
	film.rental_duration                                AS rental_duration,
	DATE(rental.rental_date)                            AS rental_date,
	DATE(rental.return_date)                            AS return_date,
	DATE(rental.return_date) - DATE(rental.rental_date) AS days_returned,
	CASE
		WHEN DATE(rental.return_date) - DATE(rental.rental_date) = film.rental_duration
			THEN 'return in time'
		WHEN DATE(rental.return_date) - DATE(rental.rental_date) > film.rental_duration
			THEN 'return late'
		WHEN DATE(rental.return_date) - DATE(rental.rental_date) < film.rental_duration
			THEN 'return early'
		WHEN rental.return_date IS NULL
			THEN 'never returned'
	END                                                 AS return_description
FROM
	film
JOIN
	inventory
ON
	film.film_id = inventory.film_id
JOIN
	rental
ON
	inventory.inventory_id = rental.inventory_id
) returned_days
GROUP BY
	returned_days.return_description
```

![Query output](/AnswerQuestion10.PNG)

## Merging Postgres with R to get the visuals
