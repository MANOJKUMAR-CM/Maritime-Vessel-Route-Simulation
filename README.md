# 🚢 Maritime Vessel Route Simulation 

## Problem Overview
This project simulates maritime vessel movement between ports using AIS (Automatic Identification System) and demonstrates a complete data engineering pipeline including simulation, ingestion, storage, and analytics

## Objectives:
1. Route Generation
2. Serve the coordinates of the vessel at the specified time interval
3. Store the coordinates served efficiently in a Database and extract information through queries/scripts

#### Route generation
- Randomly sample two ports from `UpdatedPub150.csv`, the first port is reffered to as `source` and the latter is reffered to as `destination`.
- Uses [`searoute-py`](https://pypi.org/project/searoute/) to obtain realistic navigable sea routes
- The route is obtained through `['geometry']['coordinates']` meta information from searoute function of `searoute` library
- Supports multiple vessel generation with reproducible sampling

#### AIS Message simulation and serving through websockets
- Cumulatives distance of the route checkpoints (`['geometry']['coordinates']`) from the source port is computed using **Haversine formula**
- Calculates distance travelled by ship from source port at every `t` minutes using the **Haversine formula**
- Interpolate the ships location using the ships distance from the source port and the set of coordinates within which the ship's is located using the cumulative distance of coordinates
- Generates AIS-compliant messages in **AIVDM format** using [`pyais`](https://github.com/schwehr/pyais)
- Each vessel is assigned a unique MMSI (Maritime Mobile Service Identity)
```python
NOTE: BaseMMSI: 29122001;
      Based the number of vessels N, The mmsi is obtained as BaseMMSI + route id, where route id is 0, 1, ... N.
```
``` python
Assumption:
The Ship departs from the source port when the `start_time = datetime.utcnow()` is invoked for a particular vessel.
Time stamps that is associated with the vessel's coordinates is the delta time interval t which is added to this start_time.
```
- Streams AIS messages in real-time via a WebSocket interface
- Supports **variable playback speeds**:
  - `1.0` → real-time (for example, if t=5 mins, every 5 mins only once the ship's current coordinates at that time instant is transmitted)
  - `2.0` → 2× speed (for example, if t=5 mins, every 5 mins twice the ship's current coordinates at that time instant is transmitted)
  - `-1`  → no delay (Transmits the ship's coordinates at all t intervals throughout the ship's trajectory)
- The server sends the vessel's coordinates of that particular vessel which is requested based on mmsi and would deny any transmission if its a invalid mmsi.

#### WebSocket Client & Ingestion
- WebSocket client receives AIS messages in real-time
- Decodes messages using `pyais.decode`
- Validates each record (lat/lon range check)
- Inserts valid messages into a PostgreSQL database (`ship_logs` table)
- Tracks and logs invalid entries for debugging

#### PostgreSQL Data Engineering
- Schema includes `mmsi`, `timestamp`, `latitude`, `longitude`, and raw `payload`
- Indexes added on `(mmsi)` for efficient querying

#### Querying
- Query the vessel's information based on MMSI and obtain the total distance and the actual trajectory followed by the Vessel.
```python
Assumption: 
I have assumed the ship speed at the start as 25 knots, therefore the avegrage ship speed b/n any two time stamps would
be the same and therefore that part is excluded from query section.
```

#### Visualization
Visualize vessel routes using `plotly.mapbox` with real Earth backgrounds
- Highlights source and destination points on the map
![Vessel Route Simulation](Vessel%20Route.png)

### Steps to Follow
- Execute: `CreateDataBase.ipynb`, creates the necessary database and tables required
- Execute: `Server.ipynb` to serve the ships information over websockets
- Execute: `Client.ipynb`
- Execute: `Query.ipynb`, to extract metrics

```python
Note:
conn = psycopg2.connect(
    dbname="postgres",
    user="postgres",
    password=<password>,  
    host="localhost",
    port="5432"
)

In CreateDataBase.ipynb, replace the password with your respective credentials before executing.
```

```python
Improvements:
1. Variable Ship Speed Based on Route Segments. Instead of a fixed speed throughout, I’d simulate:
- Slow departure from port
- Faster cruise in open sea
- Slow arrival at destination
- If possible, would like to include the environmental effects, such as wind speed, tide current which also plays a role in Ship's speed.

2. Here I have a made a assumption about ships starting time from the port, but I would eventually want the user to provide the ship's desired starting time
and track the vessel's location and also include the timestamps in the query section to be more precise in the response than just on the mmsi based queries.
```

