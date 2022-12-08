# Tool to evaluate RDS MySQL downtime during RDS modification

The tool connects to the specified RDS MySQL instance and executes the same query (reading available collations) every second and prints results.

If the DB is unavailable, the tool keeps reconnecting to the DB every second during one minute.

The tool prints messages on stdout wit timestamps showing when the DB is available and when the DB is not available, so the user can tell for what period the DB was unavailable.

Usage:

```bash
dotnet tool -- -e <db-endpoint> -u <username> -p <password> -d <database> -l <label>
```

To assess the DB availability via different endpoints run the tool against RDS endpoint, and RDS Proxy endpoint in separate terminal windows:

* via **RDS endpoint**:

```bash
dotnet tool -- -e <rds-endpoint> -u <username> -p <password> -l RDS | tee PgSQL_RDS.log
```

* via **RDS Proxy endpoint**:

```bash
dotnet tool -- -e <rdsproxy-endpoint> -u <username> -p <password> -l RDSProxy | tee PgSQL_RDSProxy.log
```

Then modify the RDS database, e.g. resize the instance:

```bash
./rds-resize.sh <instance-identifier> <instance-size>
```

E.g. run:

```bash
./rds-resize.sh database-1 db.t3.large
```

Keep watching the tool output to detect the downtime.

I resized my instance from `db.t3.small` to `db.t3.large` (started TBD-TBD - the operation took ~10 min) and got following results regarding the DB downtime:

* RDS endpoint: downtime 11:27:05 - 11:28:11 = 66 sec

```text
...
12/8/2022 11:27:05 PM : RDS Starting a query...
12/8/2022 11:27:05 PM : RDS ERROR: 57P01: terminating connection due to administrator command, retry #1
12/8/2022 11:27:06 PM : RDS Opening a DB connection to xxxxx
12/8/2022 11:27:06 PM : RDS ERROR: Failed to connect to 172.31.46.20:5432, retry #2
12/8/2022 11:27:07 PM : RDS Opening a DB connection to xxxxx
...
12/8/2022 11:28:10 PM : RDS ERROR: Failed to connect to 172.31.46.20:5432, retry #36
12/8/2022 11:28:11 PM : RDS Opening a DB connection to xxxxx
12/8/2022 11:28:11 PM : RDS Starting a query...
template0 en_US.UTF-8
rdsadmin en_US.UTF-8
template1 en_US.UTF-8
postgres en_US.UTF-8
```

Note that the client made TBD reconnection attempts during the DB downtime.

* RDS Proxy endpoint: 11:27:05 - 11:27:40 = 35 sec:

```text
12/8/2022 11:27:05 PM : PROXY Starting a query...
12/8/2022 11:27:05 PM : PROXY ERROR: 57P01: terminating connection due to administrator command, retry #1
12/8/2022 11:27:06 PM : PROXY Opening a DB connection to xxxxx
12/8/2022 11:27:06 PM : PROXY Starting a query...
12/8/2022 11:27:39 PM : PROXY ERROR: Exception while reading from stream, retry #2
12/8/2022 11:27:40 PM : PROXY Opening a DB connection to xxxxx
12/8/2022 11:27:40 PM : PROXY Starting a query...
template0 en_US.UTF-8
rdsadmin en_US.UTF-8
template1 en_US.UTF-8
postgres en_US.UTF-8
12/8/2022 11:28:11 PM : PROXY Starting a query...
template0 en_US.UTF-8
...
```

Note the connection behavior: the client connects successfully to the RDS Proxy during failover, however the query itself times out after 32 sec, and the 2nd query takes 11:28:11-11:27:40 = 31 sec.

Therefore we can conclude that RDS Proxy "conceals" the DB downtime behind a longer timeout: we cans see that the first data is returned almost after 1 min (which is the duration of the failover).

## Multi-AZ Cluster

TODO

## Aurora PostgreSQL

### Resize writer instance

```bash
./rds-resize.sh database-2-instance-1 db.t3.medium
```

NOTE: the resized writer will become a new reader and the previous reader will be promoted to be a writer.

* Writer endpoint - downtime TBD-TBD (TBD sec)

```text
TBD
...
```

* Reader endpoint - downtime TBD-TBD (TBD sec)

```text
TBD
```

* Proxy writer endpoint: TBD

```text
...
TBD
...
```

* Proxy reader endpoint: downtime TBD-TBD (TBD sec)

```text
...
TBD
...
```

NOTE: very strange to get the downtime from the reader endpoint!!! Need to try to create a cluster with a dedicated reader endpoint.

### Resize reader instance

We resize the new reader:

```bash
./rds-resize.sh database-2-instance-1 db.t3.small
```

* Writer endpoint: no downtime

* Reader endpoint: downtime TBD-TBD (TBD sec)

```text
TBD
```

* Proxy writer endpoint: no downtime

* Proxy reader endpoint: downtime TBD-TBD (TBD sec)

```text
...
TBD
```

NOTE: very strange to get the extremely long downtime from the reader endpoint!!! Need to try to create a cluster with a dedicated reader endpoint.

## Conclusions

With the default Aurora production setup you also get 2 nodes for HA (similarly to RDS Multi-AZ), but unlike with Multi-AZ you can use the 2nd node for read-only operations.
If  you can refactor your software to split DB reads from writes (make them access different DB endpoints), then you can scale DB “write” and “read” instances independently. And unlike with RDS Multi-AZ the Writer and Reader instances in Aurora can have different sizes.

Overall the failover times with Aurora are a lot better compared to RDS MySQL (though I need to investigate why RDS Proxy ReadOnly endpoint has the anomaly long timeouts).

## Build

Install .NET Core 6 (or better) following a guide for your OS, e.g. <https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu>

Build the tool:

```bash
dotnet build
```

## Create RDS PostgreSQL database

* RDS > Create Database

* Standard Create

* Engine type: PostgreSQL

* Engine Version: 12.12-R1 (the versions 13 and 14 are not yet supported by RDS Proxy)

* Templates: Production

* Deployment options: Multi-AZ DB instance (TODO: try Cluster as well)

* DB instance identifier: `<choose your own>`

* Master username: `postgres`

* Master Password: `<choose your own>`

* DB instance class: Burstable classes, `db.t3.small` (or `<choose your own>`)

* Storage type: `gp3`

* Allocated storage: `20Gb`

* Enable storage autoscaling: Yes

* Don't connect to an EC2 compute resource.

* VPC: `<choose your own>`

* Public access: No

* VPC security group: Create new. NB! You will need to edit the security group associated with the RDS instance to allow traffic from your EC2 instance running the "downtime detection" tool.

//* Additional configuration > Database options > Initial database name: `postgres`

* RDS Proxy: Create an RDS Proxy

Create an EC2 and allow it to connect to the RDS in the RDS security group.

## Create RDS Aurora/PostgreSQL database

* RDS > Create Database

* Standard Create

* Engine type: Amazon Aurora

* Edition: Amazon Aurora PostgreSQL-Compatible Edition

* Engine Version: Aurora PostgreSQL (Compatible with PostgreSQL 12.12)

* Templates: Production

* DB cluster identifier: `<choose your own>`

* Master username: `postgres` (or `<choose your own>`)

* Master Password: `<choose your own>`

* DB instance class: Standard classes, `db.t3.medium` (or `<choose your own>`)

* Multi-AZ deployment: Create an Aurora Replica or Reader node in a different AZ (recommended for scaled availability)

TODO: Try Don't create an Aurora Replica

* Don't connect to an EC2 compute resource.

* VPC: `<choose your own>`

* Public access: No

* VPC security group: Create new. NB! You will need to edit the security group associated with the RDS instance to allow traffic from your EC2 instance running the "downtime detection" tool.

* Create an RDS Proxy: Yes

Create an EC2 and allow it to connect to the RDS in the RDS security group.

## References

* When to use RDS Proxy <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-planning.html>

* Using Amazon RDS Proxy <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html>

Some pieces of the code are borrowed from the following sources:

* <https://learn.microsoft.com/en-us/azure/azure-sql/database/connect-query-dotnet-core?view=azuresql>

* <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_SQLServerMultiAZ.html>
