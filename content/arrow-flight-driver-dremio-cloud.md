+++
title = "Python, Arrow Flight and Dremio Cloud"
date = 2024-05-21
+++

## ODBC+M1 Mac pain

Connect to Dremio Cloud from Python is challenging for certain platforms namely the M1 mac. When using ODBC on Apple Silicon one needs to:

* Install Rosetta `/usr/sbin/softwareupdate --install-rosetta --agree-to-license`
* Install homebrew as amd64 `arch -x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
* Using the same trick install Python `arch -x86_64 /usr/local/bin/brew install python`
* Install PyODBC `pip install pyodbc`
* Install [Dremio Mac driver](https://www.dremio.com/drivers/odbc/)  
* Figure out the magic incantation for the connection string including referencing your driver correctly.

## JDBC Cross-Platform but Limited

[JayDeBeAPI](https://pypi.org/project/JayDeBeApi/) at least is available and does some amazing slight of hand using [JPype](https://jpype.readthedocs.io/en/latest/) to permit easy access to JDBC drivers into Python.
This effectively solves the cross platform development challenges. But it hasn't been updated in 3 years and does not work with JDK 9+. 

* Install JDK 8 [since one cannot easily pass through JVM settings](https://github.com/baztian/jaydebeapi/pull/116)
* Download [The JDBC driver](https://www.dremio.com/drivers/jdbc/)
* Install JayDeBeAPI `pip install jaydebeapi'
* Make sure you [get the initialization correct or else you get a lot of errors](https://github.com/baztian/jaydebeapi/issues/238)
* Figure out again the magic incantation for the connection string and how it translates through JayDeBeAPI.

## Trivial with Arrow Flight

Since the ODBC and JDBC drivers are just clients implementing the Arrow Flight protocol I figured I'd look into a native client version and see if I can get more flexibility in my deployment model and better throughput. The steps looked like this

* Install pyarrow `pip install pyarrow`
* Delete a bunch of row mapping code and end up with some pretty compact code overall.

That's it! I got the following benefits

* Faster multi-threaded performance for queries
* Smaller Docker image
* Less overall lines of code (row mapping is a treat in PyArrow)
* Easy options to output to [Pandas](https://arrow.apache.org/docs/python/generated/pyarrow.Table.html#pyarrow.Table.to_pandas), [Python objects](https://arrow.apache.org/docs/python/generated/pyarrow.Table.html#pyarrow.Table.to_pylist), etc.

Below you can see the code change needed, but even if the code was worse, or terrible, with the deployment story everything is so much more maintainable it's hard to see why I would go back.

### Query code with JayDeBeAPI

```python
import jaydebeapi

## escape token
safe_token = urllib.parse.quote_plus(token)

## connect: note connection string and jar reference
conn = jaydebeapi.connect(
            "org.apache.arrow.driver.jdbc.ArrowFlightJdbcDriver",
            f"jdbc:arrow-flight-sql://data.eu.dremio.cloud:443/?useEncryption=true&useSystemTrustStore=true&disableCertificateVerification=true&token={safe_token}",
            {},
            "flight-sql-jdbc-driver-10.0.0.jar",
        )
## Create a return object to map too
results: list[dict[str, object]] = []
## Open cursor
with conn.cursor() as cur:
    ## Execute query
    cur.execute(query)
    ## we need cursor description or we won't know the column names
    if cur.description is None:
        raise ValueError(
                    "cursor description is none, this is a critical error and indicates a bug in the jdbc driver"
        )
    ## Map based on cursor description the column names
    columns = [column[0] for column in cur.description]
    ## Iterate through results
    for row in cur.fetchall():
        results.append(dict(zip(columns, row)))
    ## Finally return the results
    return results
```

### Query code with PyArrow

What lines of code we add in query setup, we get back in automapping of result set and we get an easier connection string to boot.

```python
from pyarrow import flight
from pyarrow.flight import FlightClient

## Create Arrow Flight Client
#location = "grpc+tls://data.dremio.cloud:443" # US
location = "grpc+tls://data.eu.dremio.cloud:443" # EU
client = FlightClient(location=(location))

## Create Flight Call Options
headers = [(b"authorization", f"bearer {token}".encode("utf-8"))]
options = flight.FlightCallOptions(headers=headers)

 ## Send Query
flight_info = client.get_flight_info(
    flight.FlightDescriptor.for_command(query), options
)

## Get Query Results
query_result = client.do_get(flight_info.endpoints[0].ticket, options)
## Read them to a table
table = query_result.read_all()
## Convert them to a list or rows
return table.to_pylist()
```

