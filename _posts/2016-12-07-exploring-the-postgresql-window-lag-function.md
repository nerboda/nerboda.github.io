---
layout: post
title:  "Exploring the PostgreSQL Window lag function"
date:   2016-12-07 09:04:43 -0700
categories: sql, postgresql
---

There’s a further exploration part of one of the Launch School SQL exercises that asks you to make use of the Window lag function to do some interesting row formatting at the database level. It gave me trouble initially and required quite a bit of digging and experimentation to reach a solution. I thought I’d share my findings for those of you who 1) aren’t Launch School students and are just looking to learn about the lag function 2) ARE fellow Launch School students that have already moved past the SQL course but didn’t bother to dig into this, or 3) ARE Launch School students who are also struggling with this problem and just want to see the solution damnit!

## The Problem

The database we’re working with consists of 3 tables:
* **customers** — id, name, payment_token
* **services** — id, description, price
* **customers_services (JOIN table)** — id, customer_id, service_id

The assignment is to combine the use of the window lag function and a CASE statement to get a list of customers and the services they subscribe to, formatted in the following way.

```
     name      |    description     
---------------+--------------------
 Chen Ke-Hua   | High Bandwidth
               | Unix Hosting
 Jim Pornot    | Dedicated Hosting
               | Unix Hosting
               | Bulk Email
 Lynn Blake    | Whois Registration
               | High Bandwidth
               | Business Support
               | DNS
               | Unix Hosting
 Nancy Monreal | 
 Pat Johnson   | Whois Registration
               | DNS
               | Unix Hosting
 Scott Lakso   | DNS
               | Dedicated Hosting
               | Unix Hosting
```

Like I said this really gave me some problems at first, so before showing you the solution I’ll explain the purpose of Window functions, then talk about the lag function, then give a brief overview of the CASE statement.

Then finally I’ll show you the complete query that gave me what I was looking for.

## PostgreSQL Window Functions

Here’s how the [Postgres docs explain Window functions](https://www.postgresql.org/docs/9.5/static/tutorial-window.html)…

> A window function performs a calculation across a set of table rows that are somehow related to the current row. This is comparable to the type of calculation that can be done with an aggregate function. But unlike regular aggregate functions, use of a window function does not cause rows to become grouped into a single output row — the rows retain their separate identities. Behind the scenes, the window function is able to access more than just the current row of the query result.

The important points here are:
1. **Window functions allow access to rows other than the current row**
2. **Unlike with Aggregate functions, the rows remain separate**

### The Window lag function

Here’s how the [Postgres docs explain the lag function](https://www.postgresql.org/docs/9.5/static/functions-window.html)…

> `lag(value anyelement [, offset integer [, defaultanyelement ]])`

> Returns value evaluated at the row that is offset rows before the current row within the partition; if there is no such row, instead return default (which must be of the same type as value). Both offset and default are evaluated with respect to the current row. If omitted, offset defaults to 1 and default to null.

So you pass it a column, then an optional row offset (which defaults to 1), and then an optional default return value (which defaults to NULL).

Another important point is that Window functions always require what’s called an OVER clause, immediately following the window function call.

This looks like `lag(customers.name) OVER (ORDER BY customers.name)`

**ORDER BY** specifies the order in which the rows will be processed. This is important because in our case we want to access the previous row to check if the customer name is the same as the current row. If the rows aren’t grouped by customer name, this wouldn’t work.

## Postgres Conditional Expressions

### The CASE statement

So the final piece to our puzzle is the CASE statement. This is pretty straight forward, much like an if/else statement in many programming languages.

Here’s the format, brackets signifying that branch is optional:

{% highlight sql %}
CASE WHEN condition THEN result
 [WHEN …]
 [ELSE result]
END
{% endhighlight %}

And an example (the ELSE return value must be of the same type as the column):

{% highlight sql %}
SELECT description,
       CASE WHEN price > 100.00 THEN price
       ELSE 5.00
       END
FROM services;
{% endhighlight %}

Putting it all together

{% highlight sql %}
SELECT CASE WHEN (customers.name IS DISTINCT FROM lag(customers.name) OVER (ORDER BY customers.name))
            THEN customers.name
       END, services.description
FROM customers
LEFT OUTER JOIN customers_services ON customers_services.customer_id = customers.id
LEFT OUTER JOIN services ON services.id = customers_services.service_id;
{% endhighlight %}

#### Breaking down the query:

1. Select the customers name only when the customers name from the previous row doesn’t equal the current name.
  * You’ll notice I use IS DISTINCT FROM rather than != because we didn’t specify a default return value for the lag function, so the default value will be NULL. Remember, the != comparison operator will always return NULL when comparing a NULL value.
  * The lag function retrieves the customers name value from the previous row, with the rows being ordered properly by the OVER clause.

2. Select the service description
  * This is added to the select statement after the case statement is ended, so it will be returned for each and every row.

3. Join the other two tables using LEFT OUTER JOINs
  * LEFT OUTER JOIN the customers_services table so even customers that don’t subscribe to a service will be included.
  * LEFT OUTER JOIN the services table so we don’t unnecessary retrieve services that won’t be included in the final dataset.

And there you have it!

One question you might have (that I certainly had) is: ok where would using a window lag function like this be useful? The Postgres docs give an example [here](https://www.postgresql.org/docs/9.1/static/tutorial-window.html).

Can you think of other situations where a Window function is better suited to the problem than an Aggregate function?
