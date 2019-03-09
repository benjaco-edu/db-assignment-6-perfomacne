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

wget https://archive.org/download/stackexchange/coffee.stackexchange.com.7z
wget http://www.mysqltutorial.org/wp-content/uploads/2018/03/mysqlsampledatabase.zip
wget https://raw.githubusercontent.com/benjaco-edu/db-assignment-6-perfomacne/master/CreateTables.sql

7z e coffee.stackexchange.com.7z 
unzip mysqlsampledatabase.zip

mysql -u root -ppass1234  --local-infile

source ./CreateTables.sql;
source ./mysqlsampledatabase.sql
```

## Cleanup

Exit the shell with `exit` then `CTRL + d`

Remove the docker container with `sudo docker rm -f my_mysql`


