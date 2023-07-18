Citus provides real-time queries over large datasets. One workload we commonly see at Citus involves powering real-time dashboards of event data.

### Data Model
The data we’re dealing with is an immutable stream of log data. We’ll insert directly into Citus but it’s also common for this data to first be routed through something like **Kafka**. Doing so has the usual advantages, and makes it easier to pre-aggregate the data once data volumes become unmanageably high.

