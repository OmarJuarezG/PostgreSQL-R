# PostgreSQL-R
Uses DVD rental database to develop an interactive document and an app. To do this we connect R with a Postgres database.
## Analyzing the data
By exploring the data I've found the following:
1. Not all films are present in the inventory. Since they're not present in the inventory the company can not rent them. The code that allows to find this is the following:

Paste code here.

2. Some records have missing values. For example, there are movies that are in the inventory and were rented but do not have payment_id, payment_date or customer associated with the rent. There might be some data corruption in those cases. The code to find this is the following:

Paste code here.

3. Are the customers still active?

Paste code here.

## Questions to solve and answers
1. Is there any particular actor/actress that is more profitable in terms of movie rents? Perhaps the company could make an add featuring prominent actors so it can boost theirs rents and by doing so its revenues.
2. Is the rating of the film important to the revenues? Perhaps the company could shift its attention to a more profitable market instead of having all markets.
3. Which are the most relevant countries in terms on rents and revenue for the company? Maybe we could reinforced those markets instead of spreading resources in markets that are not profitable.
4. How the rents have behaved per month based on movie category? Could the rents be seasonal?
5. How the renvenues have behaved per month based on movie category? This will have a high correlation with the results of previous question. Hint: do we have nulls?
6. If the company wants to award premium users, it needs to identify their top 10. For this the compnay might need the customer name, month, year of payment and total payment amount for each month.
7. How many loses or replacement cost the company is incurring by clients that are not returning the rented films?
8. What are the top and least rented movies based on categories and their total revenues? (*by Okoh Anita in freeCodeCamp*)
9. What is the average rental rate for each category? (*by Okoh Anita in freeCodeCamp*)
10. How many films were returned in time, late or never returned? (*by Okoh Anita in freeCodeCamp with modification*)
11. In which countries does Rent A Film have a presence and what is the customer base in each country? What are the total sales in each country? (from most to least) (*by Okoh Anita in freeCodeCamp*)

## Merging Postgres with R to get the visuals
