# PostgreSQL-R
Uses DVD rental database to develop an interactive document and an app. To do this we connect R with a Postgres database.
## Analyzing the data
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
Query output:
| film_id | inventory_id | rental_date | rental_id | category | amount |
| ------- | ------------ | ----------- | --------- | -------- | ------ |
| 802 | null | null | null | Action | null |
| 497 | null | null | null | Documentary | null |
| 801 | null | null | null | Children | null |
| : | null | null | null | : | null |
| : | null | null | null | : | null |

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
Query output:
| rental_date | rental_id | category | payment_id | customer_id | amount |
| ------- | ------------ | ----------- | --------- | -------- | ------ |
| 2005-05-26 | 251 | New | null | null | null |
| 2005-06-17 | 2024 | Music | null | null | null |
| 2005-05-31 | 1101 | Sports | null | null | null |
| : | : | : | null | null | null |
| : | : | : | null | null | null |

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

Answer: Gina Degeneres and Matthew Carrey could do a commercial as an attempt to boost sales.

2. Is the rating of the film important to the revenues? Perhaps the company could shift its attention to a more profitable market instead of having all markets.

```sql
SELECT
	film.rating                                  AS film_rating,
	--actor.first_name || ' ' || actor.last_name AS actor_name,
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
	--actor.first_name || ' ' || actor.last_name
ORDER BY
	SUM(payment.amount) DESC

```
It seems that rating overall is well distributed in rents but there is a little spread between the revenues. Comparing the PG-13 and G, there is a difference of 14,544 in revenue.

Query output:

| film_rating | rents | revenue |
| ----------- | ----- | ------- |
| PG-13 | 593 | 72872 |
| PG | 591 | 71251 |
| NC-17 | 597 | 67120 |
| R | 595 | 65096 |
| G | 590 | 58328 |

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
**Paste answer here!!**

4. Which are the most relevant countries in terms on rents and revenue for the company? Maybe we could reinforced those markets instead of spreading resources in markets that are not profitable.
5. How the rents have behaved per month based on movie category? Could the rents be seasonal?
6. How the renvenues have behaved per month based on movie category? This will have a high correlation with the results of previous question. Hint: do we have nulls?
7. If the company wants to award premium users, it needs to identify their top 10. For this the compnay might need the customer name, month, year of payment and total payment amount for each month.
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
The cost is 911.55. **What is the net revenue? This is substracting the replacement costs**

10. What is the average rental rate for each category? (*by Okoh Anita in freeCodeCamp*)

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
**Paste answer here!!**

11. How many films were returned in time, late or never returned? (*by Okoh Anita in freeCodeCamp with modification*)
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
**Paste answer here!!**

12. In which countries does Rent A Film have a presence and what is the customer base in each country? What are the total sales in each country? (from most to least) (*by Okoh Anita in freeCodeCamp*)

## Merging Postgres with R to get the visuals
