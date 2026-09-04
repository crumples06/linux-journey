# Week-12
*31/8 - 06/9*


## Power BI
i learned about power bi as i was told to make a report in my job from an excel sheet. 
The main part in power bi is not the graph making but data formatting. It takes a lot of time and effort to just make the data into a format that is easy to make graphs out of. A lot of time you will need to make entire tables just for making one graph. 

There are slicers in power bi that allow you to filter data based on what the user selects, its a very useful thing to put in the report. A problem arose when i put a slicer, it was only affecting the charts made from the table which i made the slice from. even though the other graphs having other table also had the slicer data in common, it was still affecting only one table's graphs.
That's when i discovered the model view, here i can make relationships between tables. By maing a new table having the common values between the two tables and making them have a many to one relationship with this new table, i was able to create a slicer that affected every graph in the report.

After i created the report, i had to worry about making it look good. I didn't really want to individually decorate everything, i am not good at it and it's also time consuming. So i downloaded premade themes from the microsoft website that i just imported in my report.


## More Docker compose notes
These are more things that i learned/encountered while containarizing an app at work.

### service name vs container name
- The top-level YAML key (api, frontend, sql_dev) is the service name — used by `docker compose exec/logs/restart/ps`, and used for internal DNS resolution between containers.
- container_name is a cosmetic override for the Docker Engine-level name (docker ps, docker logs <name>) — Compose subcommands don't accept it.
- Docker's internal DNS actually resolves both names as aliases on the same network, but docker compose <cmd> specifically wants the service name.

### Internal Networking (bridge network)
- Containers on the same custom `networks:`  entry can reach each other by service name as hostname (Docker's embedded DNS). The default bridge network doesn't offer this hence a custom network.
- `getent hosts <service>` from inside a container is how you prove DNS resolution is working. I used it to confirm that `ers_api` container was resolving the `sql_dev` container.
- Confirming DNS resolve doesn't mean confirming a real connection succeeded. Always follow up with an actual query/request.

### Environment Variables
- Root `.env` (next to `docker-compose.yml`) is used by compose itself for `${VAR}` substitutions inside the compose file.
- A service only gets a variable if it's explicitly listed under that service's `environment:` (or via `.env` file)
- Compose-injected variables are set before the app process starts, so `dotenv.config()` in node becomes a no-op for anything Compose already injected. 

### build args vs runtime environment variables
- `environment:` = runtime values, read by running process.
- `build.args` = values injected during image build, matched to `ARG` in the dockerfile.

### expose vs ports
- `ports: "80:80"` publishes a port to the host machine (reachable from outside docker)
- 'expose: ["80"]` only makes the port reachable to other containers on the same network (not published to the host at all).

### Debugging techniques i used
- `docker network ls` / `docker network inspect <name>` - see which containers are on a network and their IPs
- `docker compose exec  <service> printenv | grep <VAR>` - proves what env vars a running container actually has.
- `docker compose logs -f` (optionally scoped to specific services) - watch requests ripple through the whole stack live
- Rebuild forcibly when in doubt: `docker compose build --no-cache <service>` + `docker compose up -d --force-recreate <service>`
- A lot of the problems that occurred were because of stale dockerfiles or .env files. 


