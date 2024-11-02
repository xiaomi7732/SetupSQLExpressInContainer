# Run SQL Express in container

Trying to run SQL Express inside a container and connect to it locally.

## Prereq

* Docker Desktop (because I am using Windows here)
* Select the SKU

Various SQL Server SKUs, for example, `Develoer Editon`, `Express Edition` is set by the environment variable of `MSSQL_PID`. Refer to the docker hub page for more info: <https://hub.docker.com/r/microsoft/mssql-server>. Here's a quick reference:

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

   * `--name "mssql-express"` specify the name for the running container.
   * `-e "MSSQL_SA_PASSWORD=yourStrong(!)Password"` the environemnt variable of `MSSQL_SA_PASSWORD` will be used as the admin password. And the user name will be `sa`.
   * `-e "MSSQL_PID=Express"` sets the SKU to the Express.

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
   Different than the sqlcmd inside the container, I am using ODBC version 17 here on my localbox. Newer versions of sqlcmd (in mssql-tools18) are secure by default. If using version 18 or higher, you need to add the `No` option to sqlcmd to specify that encryption is optional, not mandatory.

## Configuring the SQL Server Container

A lot of customizations could be done by `/var/opt/mssql/mssql.conf` file, if we have configuration files avaialbe locally, the matter is about mount it inside the container. Here's an example:

1. Create the configuration file([mssql.conf](./mssql.conf)), and save it to, for example, `c:\sql-express-data\`:

   ```init
   [sqlagent]
   enabled = true
   ```

1. Delete the running container:

   ```shell
   docker rm -f mssql-express
   ```

1. Map it into the container

   ```shell
   docker run --name "mssql-express" -e 'ACCEPT_EULA=Y' -e 'MSSQL_SA_PASSWORD=<YourStrong!Passw0rd>' `
   -v c:/sql-express-data/mssql.conf:/var/opt/mssql/mssql.conf `
   -d mcr.microsoft.com/mssql/server
   ```

1. Check the configuration exists

   ```shell
   docker exec -it mssql-express /bin/bash
   
   mssql@98c798b5b1a5:/$ cat /var/opt/mssql/mssql.conf 
   
   [sqlagent]
   enabled = true
   ```

To find out what can be configured: <https://learn.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf>.



## How to Preserve the Data Stored in the MySQL Docker Container

We are going to create a volume and mount it to where the data is stored in our container. Here are the steps:

1. Create a volume:

   ```shell
   docker volume create mssql-data
   ```

   The `volume create` command creates a dedicated storage on your local file system for the volume. After the volume is mounted, all container data will be linked to it.

1. Again, if you are running the container, stop and remove it:

   ```docker
   docker stop mssql-express
   docker rm mssql-express
   ```

1. Restart the container with the volume mounted:

   ```docker
   docker run --name "mssql-express" -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=yourStrong(!)Password" -e "MSSQL_PID=Express" -v mssql-data:/var/opt/mssql -d mcr.microsoft.com/mssql/server
   ```

1. **List** the volume, or **inspect** it:

   ```shell
   > docker volume ls     
   DRIVER    VOLUME NAME
   local     mssql-data

   > docker volume inspect mssql-data
   [
      {
         "CreatedAt": "2024-11-01T17:40:21Z",
         "Driver": "local",
         "Labels": null,
         "Mountpoint": "/var/lib/docker/volumes/mssql-data/_data",
         "Name": "mssql-data",
         "Options": null,
         "Scope": "local"
      }
   ]
   ```

   Notice the `MountPoint`, that is the path **supposely** on your localbox. The thing for **Windows** users, is that its managed by the container manager, like `Docker Desktop`. And there's an additional mapping from the mount point to the physical directory. For example, on my box, the actual physical mouting point is:

   ```shell
   \\wsl.localhost\docker-desktop-data\data\docker\volumes\mssql-data\_data
   ```

   Putting that into consideration, I prefer the next method of mapping even thought it takes a few more maps than a volume.

It is also possible to map the directories manually without using `docker volume`. There are 3 directories to map, `data`, `log` and `secrets`:

1. Create a local folder to host the data

   ```shell
   mkdir c:\sql-express-data
   ```

1. Again, if you are running the container, stop and remove it:

   ```docker
   docker stop mssql-express
   docker rm mssql-express
   ```

1. Run the container with the storage mapping:

   ```powershell
   docker run --name "mssql-express" -e 'ACCEPT_EULA=Y' -e 'MSSQL_SA_PASSWORD=<YourStrong!Passw0rd>' `
   -v c:/sql-express-data/data:/var/opt/mssql/data `
   -v c:/sql-express-data/log:/var/opt/mssql/log `
   -v c:/sql-express-data/secrets:/var/opt/mssql/secrets `
   -d mcr.microsoft.com/mssql/server
   ```

1. Inspect the data:

   ```powershell
   PS > ls C:\sql-express-data\

    Directory: C:\sql-express-data

   Mode                 LastWriteTime         Length Name
   ----                 -------------         ------ ----
   d----           11/1/2024 11:13 AM                data
   d----           11/1/2024 11:13 AM                log
   d----           11/1/2024 11:13 AM                secrets
   ```

   > Note: Since the folder for the data is also the folder for `mssql.conf`, you would think a simple mapping could be:

   ```powershell
   docker run --name "mssql-express" -e 'ACCEPT_EULA=Y' -e 'MSSQL_SA_PASSWORD=<YourStrong!Passw0rd>' `
   -v c:/sql-express-data:/var/opt/mssql/ `
   -d mcr.microsoft.com/mssql/server
   ```

   That’s an issue because the `.system` folder is also being persisted, which could lead to complications down the line.

## The Final Command

Throughout the article, our `docker run` command has evolved significantly. So, let's put together all its variations into one, final command. We have to stop and remove the container again. We will remove the volume as well to start from scratch:

```bash
docker stop mssql-express; docker rm mssql-express
docker volume rm mssql-data
```

So, here is the final command:

```shell
docker run --name "mssql-express" -e 'ACCEPT_EULA=Y' -e "MSSQL_PID=Express" -e 'MSSQL_SA_PASSWORD=yourStrong(!)Password' `
-p 1434:1433 `
-v c:/sql-express-data/data:/var/opt/mssql/data `
-v c:/sql-express-data/log:/var/opt/mssql/log `
-v c:/sql-express-data/secrets:/var/opt/mssql/secrets `
-v c:/sql-express-data/mssql.conf:/var/opt/mssql/mssql.conf `
-d mcr.microsoft.com/mssql/server
```

This command mounts our previous `mssql.conf` local file to the desired location, as well as the mssql directories for data persistent. It also maps local port of `1434` to the well known `1433`.

## Conclusion

In this guide, we've successfully set up SQL Server Express in a Docker container, enabling you to run a lightweight instance of SQL Server locally. By pulling the appropriate image, configuring environment variables, and managing port mappings, we ensured that our SQL Server instance is both accessible and customizable.

We also explored advanced configuration options, such as using a custom `mssql.conf` file and persisting data through `Docker volumes`. This flexibility allows for tailored setups, whether you're experimenting with SQL Server features or developing applications.

By following the final command provided, you can easily recreate or modify your SQL Server Express container with all necessary configurations, ensuring a robust development environment. With this setup, you're now ready to leverage SQL Server's capabilities right from your local machine! If you have any further questions or need assistance, feel free to reach out. Happy coding!

Oh, give me a thumbup, please.
