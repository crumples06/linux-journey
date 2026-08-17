# Week-10
*17/08/26 - 23/06/26*

## sql sever docker container
i have been trying to shift my database into a container. i had the container working and i could connect to it, query data and everything from terminal but i couldnt no matter what connect to it through my node application. After hours of debugging and frustration, i figured out the problem, my sql sever in the container and the sql server on my host machine were listening to the same port, hence my application was trying to connect to the host sql server instead of the one in the container. It was a very simple error that took me hours to find.
*It's a good lesson, from now on i will always look if a port is already occupied before using it.* 

Learned a lot of new commands and small tidbits during this process,
- `docker cp` is for copying files from windows host to the container.
- Quoting a password with `$` in it depends entirely on the shell — PowerShell needs single quotes, cmd.exe needs double quotes (or none), and tools like dotenvx do their own $-expansion inside .env files.

### Container Lifecycle:
- `-e` is used to set environment variables in the container (used in docker run or exec), like here i did `-e MSSQL_SA_PASSWORD=pass` to set sql server system administrator login password.
- `-v` is for mounting a volume. Here i made one for storing sql data by `-v mssql_data:/var/opt/mssql`, it's in the format `volume name: path to where sql server stores all its data`. If the volume doesn't exist then docker creates it automatically 

### Inspecting a container
- `docker inspect <name|id>` returns low-level information on Docker objects. It is easier to see all of it in docker desktop though.
- `docker inspect sql_dev --format "{{.Config.Env}}"` i used this to see all environment variables the container was stated with.
- Like this there are a lot of data you can format for, but it's easier to just look it up on docker desktop as it's properly formatted by type there.

### Logs
- `docker logs <name|id>` gives the full logs of the container. It can aslo be seen in docker desktop
- Adding `--tail 30` will give only the last 30 log messages
- Adding `--since 5m` will give only the logs in the last 5 minutes

## Running Commands inside container 
- `docker exec` is used to run commands inside the container. There is also a terminal in docker desktop that allows to run commands directly inside the container without having to use `docker exec`
- `docker exec -it sql_dev /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "YourPassword" -C -Q "SELECT 1"` is used to run sqlcmd inside the container. The /opt path is the path to the sqlcmd utility inside the container. `-S localhost` is the SQL Server hostname to connect to, localhost because we are inside the container. `- C` means trust server certificate without validation. `- Q` is for the actual SQL query.














