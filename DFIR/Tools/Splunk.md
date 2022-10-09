# DFIR - Common - Splunk

### Quick deployment with Splunk docker container

For the quick deployment of a Splunk instance, the
[Splunk docker image](https://hub.docker.com/r/splunk/splunk/) (by Splunk)
can be used.

```
docker pull splunk/splunk:latest

# Port 8000: Splunk web interface.
# Port 8088: Splunk HTTP event collectors service.
docker run -p 8000:8000 -p 8088:8088 -e "SPLUNK_PASSWORD=<PASSWORD>" -e "SPLUNK_START_ARGS=--accept-license" splunk/splunk:latest
```

### Splunk search Cheat Sheet

###### Search commands

| Command | Description | Example |
|---------|-------------|---------|
| `dedup <FIELD>` <br><br> `dedup <FIELD1> <FIELDN>` | Removes events containing an identical value(s) for the specified field(s). | `dedup index` |
| `\| eventcount [index=<* \| INDEX>]` | Returns the number of events in the specified indexes. | |
| `fields [+\|-] <FIELD>` <br><br> `fields <FIELD1> <FIELDN>` | Keeps or removes the specified fields. <br><br> Default to keeping fields (`+`). | |
| `rare [limit=<INT>] <FIELD>` <br><br> `rare <FIELD1> <FIELDN>` <br><br> `rare <FIELD> by <FIELD_GROUP_BY> [<FIELD_GROUP_BYN>]` | Displays the least common value of the specified field or the least common combination of values of the specified fields. <br><br> With the `group by` close, rare field(s) for each field(s) in the given grouped by fields are returned. | `... \| rare Process_Command_Line` <br> Returns the rare `Process_Command_Line` fields. <br><br> `... \| rare Process_Command_Line Account_Name` <br> Returns the rare combination of `Process_Command_Line` and `Account_Name` fields. <br><br> `... \| rare Process_Command_Line by Account_Name` <br> Returns the rare `Process_Command_Line` fields for each different `Account_Name`. |
| `reverse` | Reverse the order in which events are displayed (more recent to oldest by default). | |
| `sort [limit=<LIMIT_INT>] [+ \| -] <FIELD>` <br><br> `sort [+ \| -] <FIELD1> <FIELDN>` | Sort results by the specified field(s). The top 10 000 events are returned by default. <br><br> The `+` (default) and `-` sign can be used to sort respectively by ascending or descending order. <br><br> Cast functions (`nums`, `str`, etc.) can be applied to each fields if necessary. | `... \| sort -num(size)` <br> Sorts results by size in descending order. |
| `stats count by <FIELD>` <br><br> `stats count by <FIELD1> <FIELDN>` | Count the number of events by field or for a combination of the specified fields. | |
| `where <CONDITION>` | Filter results based on the specified condition(s) |

###### Example / useful search queries

| Query | Description |
|-------|-------------|
| `\| eventcount index=* summarize=false \| dedup index \| fields index` | Lists available (non-internal) indexes. |
| `index=* sourcetype=wineventlog EventCode=4688 \| rare limit=100 Process_Command_Line` | Returns the 100th rarest process execution command line (from non-default Windows Security logs). |
| `index=* sourcetype=xmlwineventlog EventCode=3 DestinationHostname=*<DOMAIN> \| stats count by DestinationHostname, Image` | Counts the number of hits on each subdomains of `<DOMAIN>` by Image (from Sysmon logs). |
| `\| tstats min(_time) as latest max(_time) as earliest WHERE index="<* \| INDEX>" by index, source \| convert timeformat="%Y-%m-%d %H:%M:%S" ctime(earliest) ctime(latest)` | Retrieves the earliest and latest events of each given types for the specified or all index. |

### Splunk apps

###### olafhartong's ThreatHunting

The [`ThreatHunting`](https://github.com/olafhartong/ThreatHunting) `Splunk`
application contains multiple dashboards, relying on telemetry from `Sysmon`
and mapped on the [`MITRE ATT&CK framework`](https://attack.mitre.org/).

The following `Splunk` applications must be installed for `ThreatHunting` to
work:
  - [`Punchcard Visualization`](https://splunkbase.splunk.com/app/3129/)
  - [`Force Directed App For Splunk`](https://splunkbase.splunk.com/app/3767/)
  - [`Splunk Sankey Diagram - Custom Visualization`](https://splunkbase.splunk.com/app/3112/)
  - [`Lookup File Editor`](https://splunkbase.splunk.com/app/1724/)

The [`Threathunting` application](https://splunkbase.splunk.com/app/4305/) can
then be installed. and will index should be configured on the indexers.