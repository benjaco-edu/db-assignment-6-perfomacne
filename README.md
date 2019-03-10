# Database assignment 6

_Assignment:_ 
https://github.com/datsoftlyngby/soft2019spring-databases/blob/master/assignments/assignment6.md

## Get up and running

Following will download the data, and create a Docker container.

You will be put inside the docker container where you have to execute all of the following commands

**Start container and install dependencies to fetch the database**
```
sudo docker run --rm --name my_mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=pass1234 -d mysql
sudo docker exec -it my_mysql bash 

echo "Following is inside the container"
apt-get update
apt-get install wget p7zip-full -y unzip
```
**Download the data** 
```
wget https://archive.org/download/stackexchange/coffee.stackexchange.com.7z
wget http://www.mysqltutorial.org/wp-content/uploads/2018/03/mysqlsampledatabase.zip
wget https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/CreateTables.sql

7z e coffee.stackexchange.com.7z 
unzip mysqlsampledatabase.zip
```
**The rest will be executed inside of the mysql bash**

The flag is used to get access to the xml data when it has to be imported
```
mysql -u root -ppass1234  --local-infile

source ./CreateTables.sql;
source ./mysqlsampledatabase.sql

use classicmodels;
```

### Excercise 1

In the classicmodels database, write a query that picks out those customers who are in the same city as office of their sales representative.

I selected all customers where the sales rep mathes thir postalCode, city, state and contry. In some cases, the same postcal code can be in [multiple states](https://gis.stackexchange.com/a/167333), that is the reason i matched for all fields, and it sgould depend on the desired behaviour.  It did not cost extra to check all fields.

```sql
select customers.customerNumber,
       customers.customerName,

       customers.postalCode,
       customers.city,
       customers.state,
       customers.country,

       customers.salesRepEmployeeNumber,
       employees.employeeNumber,
       concat(employees.firstName, " ", employees.lastName) as "employeeName"
from customers
       left join employees on customers.salesRepEmployeeNumber = employees.employeeNumber
       left join offices on employees.officeCode = offices.officeCode
where customers.postalCode = offices.postalCode
  and customers.city = offices.city
  and customers.state = offices.state
  and customers.country = offices.country;
```
![perf](https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/e1.png)

The red box is the missing index on office code for employees for this query

### Exercise 2

Change the database schema so that the query from exercise get better performance.

I have add a index on the city with the columns I apply the fiter on. 

```sql
create index city_index
  on customers (city, state, postalCode, country);
```

When I run the query again, the cost is drasical reduces from to 42 to 7, still have a full table scan but it is on a small table.

![perf](https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/e2.png)


### Exercise 3

We want to find out how much each office has sold and the max single payment for each office. Write two queries which give this information

I take the path from offices and then go all the way down to paymants

#### using grouping

... where I find the maximum and sum of the group

```sql
select offices.officeCode, offices.city, sum(amount) as "sum", max(amount) as "max"
from offices
left join employees e on offices.officeCode = e.officeCode
left join customers c on e.employeeNumber = c.salesRepEmployeeNumber
left join payments p on c.customerNumber = p.customerNumber
group by offices.officeCode;
```

Which executes:

![perf](https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/e31.png)

#### using windowing

... where I find the maximum and sum of the partition by the office code

```sql
select offices.city as "officeCity",
       e.firstName as "employ",
       c.customerName,
       p.paymentDate,
       p.amount,
       sum(p.amount) over (partition by offices.officeCode) as "sumForOffice",
       max(p.amount) over (partition by offices.officeCode) as "maxForOffice"
from offices
left join employees e on offices.officeCode = e.officeCode
left join customers c on e.employeeNumber = c.salesRepEmployeeNumber
left join payments p on c.customerNumber = p.customerNumber
where p.amount is not null;
```

Which executes a lot slower:

![perf](https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/e32.png)

#### Summery

I think the windowing is slower because at has to make a bigger resultset, it clearly not trying to calculate it for eaxh row because it is not 500 times slower.

### Exercise 4

In the stackexchange forum for coffee (coffee.stackexchange.com), write a query which return the displayName and title of all posts which with the word groundsinthe title.

To show the cost of joining, I started without a join:

```sql
use stackoverflow;
select Title from posts where Title like "%grounds%";
```
Which executes:

![perf](https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/e41.png)

Then with the join:

```sql
select Title, DisplayName from posts
left join users on users.Id = posts.OwnerUserId
where Title like "%grounds%";
```


Which executes:

![perf](https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/e42.png)

It is only a little slower thanks to the index.



### Exercise 5

Add a full text index to the posts table and change the query from exercise 4 so it no longer scans the entire posts table.

Add the index:
```sql
create fulltext index idx_posts_Title
  on posts (Title);
```

The query has to be modified to use the index, the new filter function is MATCH

```sql
select Title, DisplayName from posts
left join users on users.Id = posts.OwnerUserId
where match(Title) against('+grounds' IN BOOLEAN MODE);
```

Boolean mode means that words can be required my adding `+` in fron of the word, and can be deselected by adding a `-` in front of a word,

Which executes:

![perf](https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/e5.png)

A whole lot faster, it can now go directly to the word instead of checking each row.


## Cleanup

Exit the shell with `exit` then `CTRL + d`

Remove the docker container with `sudo docker rm -f my_mysql`


