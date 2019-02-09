# vqo
# VisualQueryOptimizer
Visual Query Optimizer a product of IIS-Software, Inc. 
email: iis@iis-software.com 

Kinldy, email us for professonal assistance and for access to the downloadable tar files. 


The aim of VisualQueryOptimizer is to provide an easy to use UI interface to help optimize intercept and modify queries while in-transit towards its destination database to take advantage of the latest features of AWS Cloud related to improvements in query efficiency, server-less clustering (Spectrum) and use on S3 as external tables (Glue Data Catalog). In addition Visual Query Optimizer is designed to auto spin up additional Redshift Clusters to initiate a multi-cluster Redshift Archtecture to remedy and scale beyond the current RedShift limitation around 50 concurrent queries per Cluster. Plus, support for dynamic metadata tables/views creation at run-time per cluster in the multi-clustered environment.


VisualQueryOptimizer with pgbouncer adds two new significant features:

Routing: User Interface to help pgbouncer-rr to intelligently send queries to different database servers from one client connection; use the UI to partition or load balance across multiple servers/clusters.

Rewrite: Using the UI, visually intercept and visually change client queries before they are sent to the server: use it to optimize or otherwise alter queries without modifying your application.

One can elect to deploy multiple instances of VisualQueryOptimizer to avoid throughput bottlenecks or single points of failure, or to support multiple configurations – which is what we have done in our environment as outlined below. It can live in an Auto Scaling group, and behind an Elastic Load Balancing load balancer. It can be deployed to a public subnet while your servers reside in private subnets. You can choose to run it as a bastion server using SSH tunneling.  You can use PgBouncer-rr’s recently introduced SSL support for encryption and authentication.

VisualQueryOptimizer Architecture Flowchart


VisualQueryOptimizer + PgBouncer-rr installation procedure

Follow below steps to install VisualQueryOptimizer’s latest version –

#go to the home/ec2-user directory

# install required packages

sudo yum update

sudo yum install libevent-devel openssl-devel python-devel libtool git patch make –y

# install psql

sudo yum install epel-release

sudo yum install python-pip

sudo yum install postgresql-server postgresql-contrib

# for debugging purposes, to check that we can connect to the Redshift endpoint directly using jdbc. Make sure endpoint, port and database name is correct

psql -h <Redshift Endpoint> -p 5439 –d dev -U <user_name> 

#use ctrl-z to come out

# download the latest VisualQueryOptimizer distribution

git clone https://github.com/VisualQueryOptimizer/VisualQueryOptimizer.git

# download VisualQueryOptimizer extensions

git clone https://github.com/awslabs/pgbouncer-rr-patch.git

# download the 1.7.2.tar.gz using the following wget command:

wget https://pgbouncer-rr.github.io/downloads/files/1.7.2/pgbouncer-rr-1.7.2.tar.gz

wget https://visualQueryOptimizer.github.io/downloads/files/1.0/visualqueryoptimizer-1.0.tar.gz

# rename and unzip both folders with:   

tar -xvzf pgbouncer-rr-1.7.2.tar.gz

tar -xvzf visualqueryoptimizer-1.0.tar.gz

# remove all the contents of the pgbouncer-rr folder

cd pgbouncer-rr

rm –rf *

cd..

# move all the contents of pgbouncer-rr-1.7.2 folder to the pgbouncer-rr folder

mv VisualQueryOptimizer-1.7.2/* VisualQueryOptimizer/

# move all the contents of visualqueryoptimizer-1.0 folder to the visualqueryoptimzer folder

# merge pgbouncer-rr extensions into pgbouncer code

cd pgbouncer-rr-patch

./install-pgbouncer-rr-patch.sh ../pgbouncer-rr

# build and install

cd ../pgbouncer-rr

git submodule init

rm -rf lib

git submodule update

./autogen.sh

./configure …

make

sudo make install

Using VisualQueryOptimizer visual UI, one can rewrite incoming queries to convert creation of temporary tables to creation of views and/or creation of external tables to take advantage of server-less spectrum clustering.

Make sure that the download files are in the VisualQueryOptimizer folder.

Make sure “pgbounder.ini” has a “databases” section that is pointing to the correct Redshift endpoint(s). To get the endpoint(s), please check the Redshift section in AWS console. Also consult “Setup for connecting to mirror clusters” section on how to map which endpoint to which database.

It should look something like this:

[VisualQueryOptimizer]

listen_port = 5439

listen_addr = *

auth_type = trust

auth_file = users.txt

logfile = /tmp/VisualQueryOptimizer.log

pidfile = /tmp/VisualQueryOptimizer.pid

admin_users = master

ignore_startup_parameters = extra_float_digits,TimeZone

pkt_buf = 13834280

max_packet_size = 8589934588

sbuf_loopcnt = 0

pool_mode = transaction

default_pool_size = 200

reserve_pool_size = 40

server_round_robin = 1

max_client_conn = 10000

routing_rules_py_module_file = ./routing_rules.py

rewrite_query_py_module_file = ./rewrite_query.py

rewrite_query_disconnect_on_failure = false

#client_tls_sslmode=allow

#client_tls_key_file = ./VisualQueryOptimizer-key.key

#client_tls_cert_file = ./VisualQueryOptimizer-key.crt

#client_tls_protocols=tlsv1.2

[databases]

dev = host=xxx.us-east-2.redshift.amazonaws.com port=5439 dbname=dev

dev.1 = xxx.us-east-2.redshift.amazonaws.com port=5439 dbname=dev

dev.2 = host=xxx.us-east-2.redshift.amazonaws.com port=5439 dbname=dev

rds-dev = host=dev-east-xxx.us-east-2.rds.amazonaws.com port=5432 dbname=postgres

rds-prod = host=prod-east-xxx.us-east-2.rds.amazonaws.com port=5432 dbname=postgres

Make sure “users.txt” has the correct credentials and all the credentials needed:

“master” “md5xxxxxxxxxxx”

“dbuser” “******”

Make sure that “rewrite_query.py” points to the required Redshift endpoint and has the correct username, password, port and database in the file header. Also consult “Setup for connecting to mirror clusters” section on which endpoints to use. It should look something like this:

import re

import datetime

import string

import os

import sys

import time

import psycopg2

from psycopg2 import pool

login_passwd=”none”

threaded_postgreSQL_pool=[0]

threaded_postgreSQL_pool[0] = psycopg2.pool.ThreadedConnectionPool(50, 500,user = “db_user”, password = “xxxxxr”,host = “dbname-mirror.us-east-2.redshift.amazonaws.com”, port = “5439”, database = “dev”)

Start VisualQueryOptimizer and pgbouncer-rr to run in daemon mode using below command –

$ cd ~/VisualQueryOptimizer

$ ./VisualQueryOptimizer -d VisualQueryOptimizer.ini

$ cd ~/pgbouncer-rr

$ ./pgbouncer -d pgbouncer.ini

Check to see if it is running:

ps -ef | grep pg

ps –ef | grep visual

and it should look something like this:

ec2-user 23964     1  0 15:34 ?        00:00:00 ./VisualQueryOptimizer -d VisualQueryOptimizer.ini

ec2-user 23969 23846  0 15:36 pts/0    00:00:00 grep –color=auto pg

ec2-user 23964     1  0 15:34 ?        00:00:00 ./pgbouncer -d pgboouncer.ini

ec2-user 23969 23846  0 15:36 pts/0    00:00:00 grep –color=auto pg

Check with psql that you are able to connect to Redshift using the Network Load Balancer (NLB)

# for debugging purposes, to check that we can connect to the Redshift endpoints through NLB using jdbc. Make                                        # sure DNS/IP, port and database name is correct

psql -h hadbproxy.xxx.dbname.com -p <port> –d <dbname> -U <db_user>

VisualQueryOptimizer, Pgbouncer-rr and HAproxy Integration and setup

Client connections to Redshift should use HAProxy, so that connections are routed through appropriate VisualQueryOptimizer, pgbouncer-rr to corresponding redshift cluster.

Connections to the appropriate cluster should be based on the port number configured to connect to the cluster.

listen DevRedshift<dbname>2 *:5539

    timeout connect 15s

    timeout client 60m

    timeout server 60m

    mode tcp

    server <dbname>2 <dbname>-mirror.us-east-2.redshift.amazonaws.com:5439

listen ProdRedshiftDBMirror *:5442

    timeout connect 15s

    timeout client 60m

    timeout server 60m

    mode tcp

    server <dbname> <dbname>-mirror.us-east-2.redshift.amazonaws.com:5439

listen ProdRedshiftDB *:5441

    timeout connect 15s

    timeout client 60m

    timeout server 60m

    mode tcp

    server <dbname> <dbname>.us-east-2.redshift.amazonaws.com:5439

listen DevRedshift<dbname> *:5439

        timeout connect 15s

        timeout client 60m

        timeout server 60m

        mode tcp

        server <dbname>.us-east-2.redshift.amazonaws.com:5439

#PG Bouncer HA

listen VisualQueryOptimizerHA *:5434

    timeout connect 15s

    timeout client 60m

    timeout server 60m

    mode tcp

    balance roundrobin

    server VisualQueryOptimizer1 VisualQueryOptimizer1.com:5439

    server VisualQueryOptimizer2 VisualQueryOptimizer2.com:5439

#PG Bouncer1

listen VisualQueryOptimizer1 *:5435

    timeout connect 15s

    timeout client 60m

    timeout server 60m

    mode tcp

    server VisualQueryOptimizer1 VisualQueryOptimizer1.ccdh.comcast.com:5439

#PG Bouncer2

listen VisualQueryOptimizer2 *:5436

    timeout connect 15s

    timeout client 60m

    timeout server 60m

    mode tcp

    server VisualQueryOptimizer VisualQueryOptimizer2.ccdh.comcast.com:5439

For example to connect to Dev database through VisualQueryOptimizer issue psql as below.

[ec2-user@ip-96-99-99-99 VisualQueryOptimizer]$ psql -h dbproxy.dev.com -p 5435 -d <dbname> -U admin

psql (9.2.24, server 8.0.2)

WARNING: psql version 9.2, server version 8.0.

         Some psql features might not work.

Type “help” for help.

<dbname>=# \q

VisualQueryOptimizer Query rewrite

Overview

Databases create temporary tables in their processing. Temporary tables occupy storage and limit the scalability of applications, since the redshift clusters in the environment do not have lot of storage. To overcome this limitation, VisualQueryOptimizer’s query rewrite feature is being leveraged.

Instead of creating temporary tables, VisualQueryOptimizer will convert the create table SQL into create view statements and/or Create External tables depeninding on selection made on its UI by the user. And the created view/external table on S3 will be used in the subsequent processing.

Below are the source query patterns which will be converted

  ###################################

  #  Query patterns

  ###################################

  q1= ‘SET DATESTYLE=ISO;SET ENABLE_SEQSCAN to off’

  q2= “CREATE TABLE “

  q2a = ” FROM wkf”

  q3= “INSERT INTO “

  q4= “DROP TABLE IF EXISTS “

  q5= “DROP TABLE “

  q6= “Analyze “

  q7= “BEGIN; DECLARE”

  q8= “CREATE UNLOGGED TABLE “

  q9= “COPY “

HA Redshift Clusters

To provide high availability of Redshift cluster, its allowed to create “N” number of Redshift db clusters created for each legacy software (incoming queries). When the application connects to the database via HAproxy, it does load balancing and alternates the connection to each cluster. Since the application is now creating views or external table in place of temporary tables, the views should be created on both the clusters because subsequent connections to the database from the workflow might get connected to other database due to load balancing.

                For the views/decoy tables to be created in both the clusters, the VisualQueryOptimizer rewrite program uses psycopg2 library to make threaded connection to the other cluster and the view/decoy table gets created on both the clusters. Below is the code which performs the creation on both the clusters.

import psycopg2

from psycopg2 import pool

login_passwd=”none”

threaded_postgreSQL_pool=[0]

threaded_postgreSQL_pool[0] = psycopg2.pool.ThreadedConnectionPool(5, 20,user = “db_user”, password = “password”,host =”dbname.us-east-2-mirror.redshift.amazonaws.com”, port = “5439”, database = “dev”)

elif re.match(q2, create_table_part):

….

      ext_table = “CREATE TABLE IF NOT EXISTS ” + schema_name + “.” + table_name.strip() + “_decoy (dummy1 varchar(1000)); “

      new_query = ext_table

      print(‘new_query=’ + new_query)

      jdbc_conn_threaded(new_query)

….

def jdbc_conn_threaded(query):

    #Second jdbc threaded connection pool

    import psycopg2

    from psycopg2 import pool

….

         ps_connection  = threaded_postgreSQL_pool[i].getconn()

       if(ps_connection):

          print(“successfully received connection from connection pool “)

          ps_cursor = ps_connection.cursor()

          print(“Query to execute= ” + query + “\n”)

          result=ps_cursor.execute(query)

          ps_connection.commit()

          ps_cursor.close()

          #Use this method to release the connection object and send back to connection pool

          threaded_postgreSQL_pool[i].putconn(ps_connection)

          print(“Put away a PostgreSQL connection”)

….

Linux and PgBouncer-rr tcp settings to increase flow of data

To increase the flow of data at the linux system level below parameters needs to be set in the /etc/sysctl.conf file

net.core.wmem_max=12582912

net.core.rmem_max=12582912

net.core.rmem_max=12582912

net.ipv4.tcp_rmem= 10240 87380 12582912

net.ipv4.tcp_wmem= 10240 87380 12582912

net.ipv4.tcp_window_scaling = 1

net.ipv4.tcp_timestamps = 1

net.ipv4.tcp_sack = 1

net.core.netdev_max_backlog = 5000

net.ipv4.tcp_no_metrics_save = 1

In pgbouncer.ini file, below tcp parameters needs to be set to increase flow of data via VisualQueryOptimizer.

pkt_buf = 13834280

max_packet_size = 8589934588

Setup for connecting to mirror clusters

To setup “N” number of redshift clusters, i.e. one primary and another mirror to make it a HA redshift cluster. And the way each VisualQueryOptimizer connects to its mirror is via an entry on its UI and .ini files. For example below an entry in ini file.

threaded_postgreSQL_pool[0] = psycopg2.pool.ThreadedConnectionPool(5, 20,user = “db_user”, password = “password”,host = “<dbname>-mirror.us-east-2.redshift.amazonaws.com”, port = “5439”, database = “dev”)

To accommodate addition of future clusters and simplify maintenance, rather than hard coding mirror clusters, they can be obtained from VisualQueryOptimizer.ini file. For this the convention to be followed is as follows –

Each VisualQueryOptimizer should have “dbname” db entry as its primary db and all the mirrors should be named as mirror1, mirror2 and so on. On pgbouncer-rr’s pgbouncer.ini’s database section – make “<dbname>” database entry point to primary and “mirror1” to point to mirror cluster. The rewrite process will read pgbouncer.ini and make threaded connection to <dbname2> database.

<dbname> = host=<dbname>.us-east-2.redshift.amazonaws.com port=5439 dbname=dev

<dbname2> = host=<dbname>-mirror.us-east-2.redshift.amazonaws.com port=5439 dbname=dev

Similarly on VisualQueryOptimizer2, mirror cluster will be its primary db entry and the other will be mirror.

<dbname> = host=<dbname>-mirror.us-east-2.redshift.amazonaws.com port=5439 dbname=dev

<dbname2> = host=<dbname>.us-east-2.redshift.amazonaws.com port=5439 dbname=dev

When third redshift cluster will be added, add <dbname3> database entry in pbouncer 1/2 section of the pgbouncer.ini file. And pgbouncer3’s ini will need to have the new cluster as its primary and the other 2 as its mirror.
