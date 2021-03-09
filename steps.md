##MySQL replication
- Process that allows you to easily maintain multiple copies of a MySQL data by having them copied automatically from a master to a slave database.

##Why use replication
- Can helpful for many reasons including facilitating a backup for the data 
- The way to analyze DB without using the main database
- The way to scale out.

##Implementation

###Notes before setup: 
- Make sure that mysql version is the same version in master DB and slaves.
- Make ssh for master DB in the slaves to download dump for DB.
- First, Make configuration in the slaves.

####Step One — Configure the Master Database

`sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf`

- Replace the standard IP address with the IP address of server.

From

`bind-address = 127.0.0.1`

To

`bind-address = 12.34.56.789`

- Uncomment server-id, 
  You can choose any number for this spot (it may just be easier to start with 1), 
  but the number must be unique and cannot match any other server-id in your replication group.
  
`server-id = 1`

- Uncomment log_bin, 
  This is where the real details of the replication are kept.
  The slave is going to copy all the changes that are registered in the log. 

`log_bin = /var/log/mysql/mysql-bin.log`

- The database that will be replicated on the slave server.
  You can include more than one database by repeating this line for all the databases you will need.

`binlog_do_db = database`


- Exit and reload mysql:

`sudo service mysql restart`

- We need to grant privileges to the slave.
- Open up the MySQL shell.

`mysql -u root -p`

`GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';`

`FLUSH PRIVILEGES;`

-We need to lock the database to prevent any new changes

USE database;

`FLUSH TABLES WITH READ LOCK;`

`mysqldump -u root -p database > database.sql`


###Step Two — Configure the Slave Database

- Set up mysql version as in the master.
- Create DB.

`CREATE DATABASE IF NOT EXISTS `database` CHARACTER SET = 'utf8' COLLATE 'utf8_general_ci';'

`mysql -u root -p database < database.sql`

`sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf`

- Remember server-id must be unique:

`server-id = 2`

`relay-log               = /var/log/mysql/mysql-relay-bin.log`
`log_bin                 = /var/log/mysql/mysql-bin.log`
`binlog_do_db            = database`

`sudo service mysql restart`

- Get these credentials from master server:

`show master status;`

Run this on the slave:

`CHANGE MASTER TO MASTER_HOST='12.34.56.789',MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=  107;`

- MASTER_HOST= The IP address or hostname of the master server
- MASTER_USER= We created user with replication privileges on the master
- MASTER_PASSWORD= The password of MASTER_USER on the master
- MASTER_LOG_FILE= The current master log file name
- MASTER_LOG_POS= The current master log file position


`START SLAVE;`

`SHOW SLAVE STATUS\G`

- make sure the value of `Slave_SQL_Running: YES`
- Replication stopped if any invalid sql query happened so 
  if you want to skip any sql errors happened:

`mysql> STOP SLAVE;`

We tell the slave to simply skip the invalid SQL query:

`mysql> SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;`

This tells the slave to skip one query (which is the invalid one that caused the replication to stop). If you'd like to skip two queries, you'd use SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 2; instead and so on.

`mysql> START SLAVE;`

- Don't forget to unlock tables in the master: 

`UNLOCK TABLES;'

- If you want to skip errors in replication with code you know:

you need to add this line in `/etc/mysql/mysql.conf.d/mysqld.cnf`

`slave-skip-errors = 1049`

- 1049 the error code from phpmyadmin, you can add more codes or all
  
`slave-skip-errors = 1049, 1062`
`slave-skip-errors = all`

and reload mysql
