---
layout: post
title: Reporting across different attribute roles in MSTR
---

Suppose you have a dimensional model about travel prices like this 

| fact_airfare | 
| ------------ | 
| depart_city_id | int |
| arrive_city_id | int |
| price | numeric |

| fact_hotel | 
| ---------- |
| city_id | int | 
| price | numeric |
 
| dim_city |
| -------- |
| id | int |
| name | varchar | 

In MicroStrategy, you need to configure your model with aliases for the underlying lookup table so it gets joined twice in the query if you want to 
report on both departure and arrival city simultaneously.  [This is a standard, well-documented process](https://community.microstrategy.com/s/article/ka1440000009I8nAAE/KB6197-How-do-Attribute-Roles-Work-in-MicroStrategy).  [There is a similar process for SSAS cubes.](http://www.jamesserra.com/archive/2011/11/role-playing-dimensions/)

What I want to talk about is what happens if you want to combine these facts and report on airfare and hotel prices together.

Airfare and hotel facts both refer to cities, but in different roles.  We have three attribute roles total: City (generic), Departure City, and Arrival City.
If you want a report like (City, Average hotel price, Average airfare price), you need to define how City will be applied to airfare.  If the business-meaningful 
way to combine these metrics are to treat arrival city as the generic city, then you would map the City attribute to the `arrive_city_id` column
in `fact_airfare`, *in addition* to the Arrival City attribute.  You can have the same column mapped from two different attributes.  

An alternative approach is to set up a parent-child relationship between Arrival City (child) and City (parent).  I don't like that nearly as much, as there isn't 
really a direct relationship between the two attributes.  It feels more appropriate to say that you can slice the facts different ways, than to say that you can roll up
one to the other.