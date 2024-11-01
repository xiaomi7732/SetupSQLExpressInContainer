# Run SQL Express in container

Trying to run SQL Express inside a container and connect to it locally.

## Prereq

* Docker Desktop
* Select the SKU

Various SQL Server SKUs, for example, Develoer Editon, Express Edition are set by the environment variable of `MSSQL_PID`. Refer to the docker hub page for more info: <https://hub.docker.com/r/microsoft/mssql-server>. Here's a quick reference:

* Developer : This will run the container using the Developer Edition (this is the default if no MSSQL_PID environment variable is supplied)
* Express : This will run the container using the Express Edition
* Standard : This will run the container using the Standard Edition
* Enterprise : This will run the container using the Enterprise Edition
* EnterpriseCore : This will run the container using the Enterprise Edition Core : This will run the container with the edition that is associated with the PID

## Getting Started

1. Pull the image

    ```shell
    docker pull mcr.microsoft.com/mssql/server
    ```

1. Run the docker container, as express editon

   ```shell
   docker run --name "mssql-express" -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=yourStrong(!)Password" -e "MSSQL_PID=Express" -d mcr.microsoft.com/mssql/server
   ```

1. Check the container is up & running

   ```shell
   > docker container ls
   CONTAINER ID   IMAGE                            COMMAND                  CREATED         STATUS         PORTS      NAMES
   ca4e0681de75   mcr.microsoft.com/mssql/server   "/opt/mssql/bin/perm…"   6 seconds ago   Up 5 seconds   1433/tcp   mssql-express
   ...
   ```

1. Run SQL command to verify it is up and running:

   1. Connect to `sqlcmd` interactively:

      ```shell
      docker exec -it mssql-express /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "yourStrong(!)Password" -No
      ```

      * `mssql-express` is the docker container name.
      * `/opt/mssql-tools18/bin/sqlcmd` is the binary to execute. Notice this is replacing the old `/*pt/mssql-tools/bin/sqlcmd`.
      * `-S` specifies the server address, in this case, the localhost of the container
      * `-U` is followed by the user name of `sa`
      * `-P` is followed by the password
      * `-N` is encryption options, it can be followed by `[s|m|o]`, where `s` stands for strict, `m` for mandatory, and `o` for optional. For more info, refer to [Connecting with sqlcmd - ODBC Drvier for SQL Server](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/connecting-with-sqlcmd?view=sql-server-ver16).

   1. Execute a query, for example:

      ```sql
      1> SELECT @@VERSION;
      2> GO
      ```

      and the result looks like this:

      ```shell
      Microsoft SQL Server 2022 (RTM-CU15-GDR) (KB5046059) - 16.0.4150.1 (X64)
        Sep 25 2024 17:34:41
        Copyright (C) 2022 Microsoft Corporation
        Express Edition (64-bit) on Linux (Ubuntu 22.04.5 LTS) <X64>
      ```

      The SQL Server Express is running in the container. However it is not accessable from the local box at this point.

      Notice that `1433/tcp` is the port the container's listening on, but there's no local port working for it.

## Connecting to the SQL Server Container Locally

1. Install sqlcmd in your system, for exmaple

   ```shell
   winget install sqlcmd --source winget
   ```

1. After installation, if you are running the container, stop and remove it:

   ```docker
   docker stop mssql-express
   docker rm mssql-express
   ```

1. Next, we will restart the container by **mapping a port** from the container to a port on our local machine:

   ```shell
   docker run --name "mssql-express" -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=yourStrong(!)Password" -e "MSSQL_PID=Express" -p 1434:1433 -d mcr.microsoft.com/mssql/server
   ```

   * `-p 1434:1433`: Maps the container's port 1433 (the default port for SQLServer) to your local port 1434. This means any traffic sent to your local port 1434 will be forwarded to the container's port 1433 and your SQL server will be accessible on that port.

1. After terminal outputs a new ID for the container, we can check the port mappings:

   ```shell
   docker port mssql-express
   1433/tcp -> 0.0.0.0:1434
   ```

   It was successful! Now, from your local machine, you can connect to the server on port 1434 using mysql client:

   ```shell
   sqlcmd -S localhost,1434 -U sa -P "yourStrong(!)Password"
   1> SELECT @@VERSION
   2> go
   Microsoft SQL Server 2022 (RTM-CU15-GDR) (KB5046059) - 16.0.4150.1 (X64)
      Sep 25 2024 17:34:41
      Copyright (C) 2022 Microsoft Corporation
      Express Edition (64-bit) on Linux (Ubuntu 22.04.5 LTS) <X64>
   (1 rows affected)
   ...
   ```

   * `-S localhost,1434` the server address is `localhost`, port of `1434`, the separator is a comma(`,`), not a semi-colon(`;`).

   > Note  
   Newer versions of sqlcmd (in mssql-tools18) are secure by default. If using version 18 or higher, you need to add the `No` option to sqlcmd to specify that encryption is optional, not mandatory.

## Configuring the SQL Server Container

## How to Preserve the Data Stored in the MySQL Docker Container

