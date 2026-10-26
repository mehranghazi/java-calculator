# Tomcat Calculator (.war)

A small calculator web app with a load / stress test button, packaged as a `.war`.

## What is a .war?

A **WAR** ("Web Application aRchive") is just a **zip file** with a fixed layout that Tomcat understands:

```
calculator.war
├── index.html                      <- the web page (calculator + stress test UI)
└── WEB-INF/
    ├── web.xml                     <- routing table: which Java class answers which URL
    └── classes/com/example/calc/   <- the compiled Java code (.class files)
```

To deploy, you put the file in Tomcat's `webapps/` folder. Tomcat unpacks it and starts the app.
The **file name becomes the URL path**: `calculator.war` -> `http://host:8080/calculator/`
(`ROOT.war` would be served at `/`).

## What is CATALINA_OPTS?

Tomcat is started by a script (`catalina.sh`). That script launches Java, and it adds whatever is in the
environment variable **`CATALINA_OPTS`** to the `java` command line. So `CATALINA_OPTS` is how you give
options to the JVM that runs *your* app:

| Kind of option | Example | What it does |
|---|---|---|
| Memory | `-Xms256m -Xmx512m` | start / maximum Java heap size |
| System property | `-Dapp.name=Calc-Lab` | a named setting your code reads with `System.getProperty("app.name")` |

This app reads these `-D` settings (all optional, defaults in brackets):

| Option | Meaning |
|---|---|
| `-Dapp.name=...` | Title shown on the page [`Tomcat Calculator`]. No spaces. |
| `-Dapp.stress.enabled=true/false` | Turn the stress endpoint on/off [`true`] |
| `-Dapp.stress.maxCpuMs=3000` | Max CPU burn time per request [`3000`] |
| `-Dapp.stress.maxThreads=N` | Max busy threads per request [number of CPUs] |
| `-Dapp.stress.maxAllocMb=128` | Max memory one request may hold [`128`] |

The page has a **Server & CATALINA_OPTS** panel that shows the heap size, the `CATALINA_OPTS` value and
the JVM arguments, so you can see the options really arrived.

## Which WAR file do I use?

Check your Tomcat version:

```bash
docker run --rm <your-image> catalina.sh version      # look for "Server version: Apache Tomcat/X.Y.Z"
```

| Tomcat version | File |
|---|---|
| 10.1.x or 11.x | `calculator.war` (needs Java 11+) |
| 9.x or 8.5.x | `calculator-tomcat9.war` (needs Java 8+) |

(Tomcat 10 renamed the `javax.servlet` packages to `jakarta.servlet`, so the two are not interchangeable.
Wrong file = errors about `javax.servlet` / `jakarta.servlet` in `docker logs`.)

## Run it

Mount the WAR into Tomcat's `webapps` folder (path shown is for the official `tomcat` image; check yours with
`docker exec <container> printenv CATALINA_HOME`):

```bash
docker run -d --name calc -p 8080:8080 \
  -v "$PWD/calculator.war:/usr/local/tomcat/webapps/calculator.war:ro" \
  -e CATALINA_OPTS="-Xms256m -Xmx512m -Dapp.name=Calc-Lab -Dapp.stress.maxCpuMs=3000 -Dapp.stress.maxAllocMb=128" \
  <your-tomcat-image>
```

(For Tomcat 9, mount `calculator-tomcat9.war` but keep the target name `calculator.war`.)

Open **http://localhost:8080/calculator/** and watch logs with `docker logs -f calc`.

Or bake it into your own image:

```dockerfile
FROM <your-tomcat-image>
COPY calculator.war /usr/local/tomcat/webapps/calculator.war
ENV CATALINA_OPTS="-Xms256m -Xmx512m -Dapp.name=Calc-Lab"
```

## API

```bash
curl "http://localhost:8080/calculator/api/calc?op=add&a=12&b=5"     # add sub mul div mod pow
curl "http://localhost:8080/calculator/api/info"                     # JVM / CATALINA_OPTS details
curl "http://localhost:8080/calculator/api/stress?cpuMs=200&threads=1&allocMb=0"
```

`api/stress` parameters: `cpuMs` (busy time), `threads` (busy threads inside one request),
`allocMb` (memory held while the request runs). All are capped by the `app.stress.*` limits above.

## Load and stress testing

* **Button in the page**: set total requests, concurrency, CPU / threads / memory per request, press *Start test*.
  It reports OK/failed counts, requests per second, average / p95 / max latency. Browsers only keep about 6
  connections open per host, so use *Threads per request* to push harder from the page.
* **From the command line** (heavier):
  ```bash
  hey -n 2000 -c 50 "http://localhost:8080/calculator/api/stress?cpuMs=100"
  ab  -n 2000 -c 50 "http://localhost:8080/calculator/api/stress?cpuMs=100"
  ```
* Watch the container while it runs: `docker stats calc`

Experiments that show CATALINA_OPTS at work:

1. Start with `-Xmx64m`, set *Memory per request* to 40 MB and concurrency 6 -> requests fail with
   `OutOfMemoryError` (HTTP 500) but Tomcat stays up. Restart with `-Xmx1g` -> they succeed.
2. Add `--cpus=1` to `docker run`, then compare latency with 1 vs 4 threads per request.
3. Set `-Dapp.stress.enabled=false` -> `api/stress` answers 403.

**Warning:** the stress endpoint deliberately burns CPU and memory. Do not expose it to the internet;
switch it off with `-Dapp.stress.enabled=false` when you are done.

## Rebuilding from source

Source is a standard Maven project (needs JDK 11+ and Maven):

```bash
./build.sh        # writes dist/calculator.war and dist/calculator-tomcat9.war
```
