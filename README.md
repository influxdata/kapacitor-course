# Misc TICKscript examples

## a_tick.tick 
```
var testname = 'Test'

// TestAutomation,testname=A,clustername=1 testStatus=0,other=10
// TestAutomation,testname=B,clustername=2 other=10
// TestAutomation,testname=B,clustername=2 other=10
// TestAutomation,testname=A,clustername=1 testStatus=0
// TestAutomation,testname=B,clustername=2 testStatus=0,other=10
var success = batch
    |query('select testStatus from "test"."autogen"."TestAutomation"')
        .period(20s)
        .every(10s)
        .groupBy('testname', 'clustername')
    |where(lambda: "testStatus" == 0)
    |count('testStatus')
        .as('count')
    |where(lambda: "count" > 2)
    |stats(10s)
    |derivative('emitted')
    |alert()
        .message('{{ .Level }} | Continous Fails US | {{ index .Tags "clustername" }} ' + string(testname))
        .crit(lambda: "emitted" > 0)
        .stateChangesOnly(1m)
        .log('/tmp/marathon_metrics_batch_alert_us.log')
```
## al.tick 
```
stream
    |from()
    |eval(lambda: if(((weekday("time") >= 1 AND weekday("time") <= 5) AND (hour("time") >= 8 AND hour("time") <= 17)), 'true', 'false'))
        .as('business_hours')
        .tags('business_hours')
        .keep()
    |log()
    |influxDBOut()
        .database('telegraf')
        .retentionPolicy('autogen')
        .tag('kapacitor_augmented', 'true')
```
## alert.tick 
```
var data = stream
  |from()
    .measurement('cpu')
  |window()
   .period(5s)
   .every(5s)
   |httpPost('http://localhost:6060')
     .header('example','b')
```
## any_influxdb_write_errors.tick 
```
// SELECT writeError FROM telegraf..influxdb_write WHERE tmpltime() group by host
// This assumes that you're using Telegraf and have the InfluxDB plugin enabled
// https://github.com/influxdata/telegraf/tree/master/plugins/inputs/influxdb

batch
    |query('SELECT writeError, writeOk, writeTimeout FROM "telegraf"."default".influxdb_write')
        .period(10s)
        .every(10s)
        .groupBy('host')
    |derivative('writeError')
        .nonNegative()
    |alert()
        .crit(lambda: "derivative" > 2)
        .details('')
        .log('/dev/stdout')
```
## b.tick 
```
var data = batch
  |query('select * from "telegraf".autogen.cpu')
    .every(10s)
    .period(10s)
  |log()

```
## batch.tick 
```
batch
 |query('SELECT * from "_internal"."monitor"."write"')
  .period(1m)
  .every(5s)
  .align()
 |log()
```
## c.tick 
```
// Query the last years worth of data, but include `mean` call so that timestamp
// returned will be the lower boundary of the time on the query issued. Without that
// the timestamp would be variable
var last_year = batch
   |query('SELECT last("my_field") as my_field, mean("my_field") FROM "mydb"."myrp"."mymeasurement"')
      .period(56w)
      .every(10m)
   // Shift the data up to the current time so that we can join on that timestamp
   |shift(56w)

// Query the average over the last 10m
var mean = batch
   |query('SELECT mean("my_field") as my_field FROM "mydb"."myrp"."mymeasurement"')
      .period(10m)
      .every(10m)
   // Shift the data up to the current time so that we can join on that timestamp
   |shift(10m)

// Join the last year data with the current average
last_year
    |join(mean)
      .as('last_year','mean')
      .tolerance(5m)
    |alert()
      .crit(lambda: "last_year.my_field" > 0 AND "mean.my_field" < 10)
```
## cap_stream.tick 
```
stream
   |from()
     .measurement('cpu')
     .database('telegraf')
     .retentionPolicy('autogen')
     .where(lambda: "cpu" == 'cpu-total')
     .groupBy(*)
  |window()
    .period(45s)
    .every(10s)
  |count('usage_user')
    .as('user')
  |log()
  |deadman(0.0, 10s)
    .post('http://localhost:5555')
```
## d.tick 
```
stream
    |from()
        .database('door')
        .measurement('door')
    |stateDuration(lambda: "value" == FALSE)
        .unit(1s)
    |influxDBOut()
        .database('door')
        .retentionPolicy('autogen')
        .measurement('door_open')
```
## deadman.tick 
```
stream
    |from()
        .measurement('internal_agent')
        .groupBy('host')
    |deadman(1.0, 10s)
        .id('{ index .Tags "host" }')
```
## derivative.tick 
```
stream
    |from()
        .measurement('mysql')
        .groupBy('host')
    |window()
        .period(30s)
        .every(30s)
    |max('commands_select')
        .as('commands_select')
    |derivative('commands_select')
        .unit(1s)
        // default
        .nonNegative()
        .as('div_commands_select')
    |influxDBOut()
        .database('performance')
        .retentionPolicy('short')
        .measurement('1m_mysql')
        .precision('s')
```
## e.tick 
```
stream
    |from()
        .database('stress')
        .measurement('m0')
        .groupBy(*)
    |window()
        .period(5m)
        .every(5m)
        .align()
    |mean('v0')
        .as('v0_mean')
    |influxDBOut()
        .database('stress')
        .retentionPolicy('autogen')
        .measurement('v0_mean_5_min')
        .precision('s')```
## elap.tick 
```
stream
  |from()
    .measurement('cpu')
  |window()
    .period(30s)
    .every(30s)
  |mean('usage_user')
    .as('usage')
  |elapsed('usage',1s)
    .as('value')
  |log()
```
## example.tick 
```
var data = stream
  |from()
    .measurement('cpu')
  |eval(lambda: "usage_idle")
    .as('value')
  |log()


```
## example_batch.tick 
```
dbrp "telegraf"."autogen"

var data = batch
  |query('select * from "telegraf".autogen.cpu')
    .every(10s)
    .period(10s)
  |log()

```
## fruit_company.tick 
```
var window_size = 20s

var m1 = stream
    |from()
        .measurement('cpu')
    |window()
        .period(window_size)
        .every(window_size)
        .align()
    |count('usage_user')
        .as('value')

var m2 = stream
    |from()
        .measurement('cpu')
    |window()
        .period(window_size)
        .every(window_size)
        .align()
    |count('usage_user')
        .as('value')

var data = m1
    |join(m2)
        .as('m1', 'm2')

data
    |alert()
        .crit(lambda: "m1.value" != "m2.value" + 1)
        .message('values were not equal m1 value is {{ index .Fields "m1.value" }} m2 value is {{ index .Fields "m2.value" }}')

data
    |eval(lambda: "m1.value" - "m2.value")
        .as('value_diff')
        .keep()
    |eval(lambda: (float("value_diff") / float("m1.value")) * 100.0, lambda: (float("value_diff") / float("m2.value")) * 100.0)
        .as('diff_percentage_m1', 'diff_percentage_m2')
    |influxDBOut()
        .measurement('diffs')
        .database('mydb')
        .create()
```
## group.tick 
```
stream
 |from()
  .measurement('my-meas')
 |default()
  .tag('gauge_name','')
  .tag('host','unknown')
 |where(lambda: "host" != 'unknown')
 |where(lambda: "gauge_name" == 'jvm_uptime')
 |groupBy('host')
 |log()
 |deadman(1.0, 15s)
  .id('/{{ .Group }}/jvm_uptime_deadman')
  .message('{{ .Level }}: {{ .ID }}')
  .log('/tmp/alerts-deadman.log')


// my-meas,gauge_name=jvm_uptime,host=A x=1
// my-meas,gauge_name=jvm_uptime,host=A x=1
// my-meas,gauge_name=jvm_uptime,host=B x=1
// my-meas,gauge_name=jvm_uptime,host=A x=1
// my-meas,gauge_name=jvm_uptime,host=B x=1
// my-meas,gauge_name=jvm_uptime,host=A x=1
// my-meas,gauge_name=jvm_uptime,host=B x=1
// my-meas,gauge_name=jvm_uptime,host=A x=1
// my-meas,gauge_name=jvm_uptime,host=B x=1
```
## high_host_disk.tick 
```
// SELECT used_percent FROM telegraf..disk WHERE tmpltime() group by host
// This TICKscript assumes that you're using Telegrafs standard system plugin

batch
    |query('SELECT used_percent FROM "telegraf"."default".disk')
        .period(10s)
        .every(10s)
        .groupBy('host')
    |alert()
        .crit(lambda: "used_percent" > 80)
        .details('')
        .stateChangesOnly()
        .log('/dev/stdout')
```
## high_host_mem.tick 
```
// This TICKscript assumes that you're using Telegrafs standard system plugin

batch
    |query('SELECT used_percent FROM "telegraf"."default".mem')
        .period(10s)
        .every(10s)
        .groupBy('host')
    |alert()
        .crit(lambda: "used_percent" > 80)
        .stateChangesOnly()
        .details('')
        .log('/dev/stdout')
        .victorOps()
        .routingKey('awstest')
```
## idk.tick 
```
// See what the count is when Kapacitor counts the raw data
batch
    |query('SELECT * FROM "demonstration"."telemetry_retention"."telemetry"')
        .period(3d)
        .cron('0 55 13 * * * *')
    |count('WK_Zen')
        .as('a_WKZen')
    |log()

// See what the count is when the count query is issued directly to InfluxDb
batch
    |query('SELECT count(*) AS b FROM "demonstration"."telemetry_retention"."telemetry"')
        .period(3d)
        .cron('0 55 13 * * * *')
    |log()

// The old query
batch
    |query('SELECT * FROM "demonstration"."telemetry_retention"."telemetry"')
        .period(3d)
        .cron('0 55 13 * * * *')
        .groupBy(*)
    |shift(24h)
    |influxDBOut()
        .database('demowork')
        .retentionPolicy('telemetry_retention')
        .measurement('telemetry')
        .precision('ms')
```
## idk_1.tick 
```
// kapacitor enable http_response_bailiff.tick
var warn = 401

var crit = 500

var threshold = 2

stream
    |from()
        .measurement('http_response')
        .where(lambda: "server" == 'http://bailiff-prd.valkyrie.net/api/v1/healthcheck')
        .groupBy('http_response_code', 'server', 'host')
    |where(lambda: "http_response_code" >= crit)
    |stats(1m)
    |derivative('emitted')
        .unit(1m)
        .nonNegative()
        .as('emitted')
    |alert()
        .crit(lambda: "emitted" >= threshold)
        .warn(lambda: "http_response_code" > warn AND "http_response_code" != 408)
        .id('{{ index .Tags "host" }}')
        .message('{{ .Level }}: HTTP Response is {{ index .Fields "http_response_code" }} for HEALTHCHECK {{ index .Tags "server"}}')
        .pagerDuty()
```
## idk_2.tick 
```
/ Dataframe
var data = stream
    |from()
        .database(database)
        .retentionPolicy(rp)
        .measurement('fluentd')
        .groupBy('host', 'plugin_type')
        .where(lambda: "region" == region)
        .where(lambda: "environment" == environment)
        .where(lambda: "envid" == envid)
        .where(lambda: "scope" == scope)
        .where(lambda: "purpose" == purpose)
    |where(lambda: isPresent("buffer_queue_length"))
    |window()
        .period(period)
        .every(every)
        .fillPeriod()
    |mean('buffer_queue_length')
        .as('stat')
    |stateDuration(lambda: "stat" > crit)
        .unit(period)
        .as('sd')
```
## issue1192.tick 
```
batch
    |query('select mean(used_percent) from "telegraf"."autogen"."mem"')
      .groupBy('tag1','missing-tag')
      .period(30s)
      .every(30s)
    |eval(lambda: trunc("mean"))
      .as('mean')
    |default()
      .tag('missing-tag', 'unknown')
    |log()
    |alert()
        .id('kapacitor/{{ index .Tags "missing-tag" }}/...')
```
## jack_join.tick 
```
var points = 12.0
var period = 1m
var every = 1m

var data = stream
    |from()
        .measurement('http_response')
        .groupBy(*)
    |eval()
        .keep('response_time')
    |window()
        .period(period)
        .every(every)
        .align

var success_rate = data
  |where(lambda: "http_response_code" < 302 AND "response_string_match" == 1)
  |count('response_time')
    .as('count')
  |eval(lambda: (float("count") / points) * 100.0)
    .as('rate')

var mean_data = data
  |mean('response_time')
    .as('response_time')
  |eval(lambda: "response_time" * 1000.0)
    .as('response_time')

var p50_data = data
  |percentile('response_time', 50.0)
    .as('response_time')
  |eval(lambda: "response_time" * 1000.0)
    .as('response_time')

var p95_data = data
  |percentile('response_time', 95.0)
    .as('response_time')
  |eval(lambda: "response_time" * 1000.0)
    .as('response_time')

var p99_data = data
  |percentile('response_time', 99.0)
    .as('response_time')
  |eval(lambda: "response_time" * 1000.0)
    .as('response_time')

success_rate
  |join(mean_data, p50_data, p95_data, p99_data)
    .as('success', 'mean', 'p50', 'p95', 'p99')
  |influxDBOut()
    .create()
    .database('telegraf_derived')
    .retentionPolicy('autogen')
    .measurement('http_response_rates')
```
## kap_stream.tick 
```
var period = 10s

var every = 10s

var data = stream
    |from()
        .measurement('m')
        .groupBy('hname')
    |window()
        .period(period)
        .every(every)
        .fillPeriod()

var stata = data
    |where(lambda: "sts" == 'ok')
    |sum('count')
        .as('value')

var statb = data
    |sum('count')
        .as('value')

stata
    |join(statb)
        .as('stata', 'statb')
    |eval(lambda: (float("stata.value") / float("statb.value")) * 100.0)
        .as('stat')
    |where(lambda: "stat" > 0.0)
    |log()
    |alert()
        // .id('{{ index .Tags "host"}}/rmqh_d-fail-ratio')
        .id('{{ index .Tags "hname"}}')
        .message('KapAlert: us-east-1-dev-devops03-glb rmqh_d fail ratio {{.Level}}')
        .crit(lambda: "stat" > 0.0)
        .stateChangesOnly()
        // .flapping(0.25, 0.5)
        .log('/Users/michaeldesa/tickscript/kap.log')
```
## l.tick 
```
var message = 'batch alerting {{.Time}} {{ index .Fields "value" }}'

var current = batch
    |query('select mean(nummeasurements) as value from "_internal"."monitor"."database"')
        .period(1m)
        .every(1m)
        .align()

var past = batch
    |query('select mean(nummeasurements) as value from "_internal"."monitor"."database"')
        .period(1m)
        .every(1m)
        .align()
        .offset(24h)
    |shift(24h)

var trigger = past
    |join(current)
        .as('past', 'current')
    |eval(lambda: float("current.value" - "past.value"))
        .keep()
        .as('value')
    |alert()
        .crit(lambda: "value" > 0)
        .stateChangesOnly()
        .message(message)
        .post('http://xxxxx')
```
## local_write_failures.tick 
```
// Fire an alert when write errors occur (grouped by database)
var period = 10s
var every = 10s
var delta = 1s

stream
    |from()
        .database('_internal')
        .measurement('shard')
        .groupBy('database')
    |window()
        .period(period)
        .every(every)
    |max('writePointsFail')
        .as('max')
    |derivative('max')
        .unit(delta)
        .nonNegative()
        .as('derivative')
    |alert()
        .id('write_failures/{{ index .Tags "database" }}')
        .message('There have been write failures on database: {{ index .Tags "database" }}')
        .details('')
        .warn(lambda: "derivative" >= 1)
        .stateChangesOnly()
        .log('/var/log/kapacitor/write_failures.log')
```
## missing_isPresent.tick 
```
var data = stream
  |from()
    .measurement('cpu')
  |eval(lambda: isPresent("usage_funny"))
    .as('missing')
```
## new_example.tick 
```
// kapacitor define e -tick example.tick -dbrp stress.autogen -type stream
// kapacitor enable e
stream
    |from()
        .measurement('mysql')
        .groupBy(*)
    |window()
        .period(1m)
        .every(1m)
        .align()
    |default()
        .field('threads_connected', 0)
    |mean('threads_connected')
        .as('mean_threads_connected')
//    |eval()
//        .keep('max_threads_connected')
    |log()
```
## new_join.tick 
```
var data = stream
    |from()
        .measurement('mysql')
        .retentionPolicy('default')
        .database('performance')
        .groupBy(*)
    |where(lambda: "host" == 'C9SQL01.CLOUD9.NVMNETWORKS.COM')

var c1 = data
    |where(lambda: isPresent("commands_select"))
    |derivative('commands_select')
        .unit(1s)
        .nonNegative()
        .as('div1')

var c2 = data
    |where(lambda: isPresent("commands_insert"))
    |derivative('commands_insert')
        .unit(1s)
        .nonNegative()
        .as('div2')

c1
    |join(c2)
        .as('c1', 'c2')
    |eval()
        .keep('c1.div1', 'c2.div2')
    |httpOut('out')
```
## old_example.tick 
```
var data = stream
  |from()
    .measurement('cpu')
  |log()

```
## other.tick 
```
var data = stream
  |from()
    .measurement('example')
  |eval(lambda: (float("x")/float("y")) * 100)
   .as('result')
  |log()
```
## other_d.tick 
```
var db = 'iotdata'

var rp = 'autogen'

var measurement = 'influxdata_sensors'

var groupBy = [*]

var whereFilter = lambda: ("location" == 'TravelingWilbury')

var name = 'relative-temp'

var idVar = name + ':{{.Group}}'

var message = ' {{ index .Fields "value" }}'

var idTag = 'alertID'

var levelTag = 'level'

var messageField = 'message'

var durationField = 'duration'

var outputDB = 'chronograf'

var outputRP = 'autogen'

var outputMeasurement = 'alerts'

var triggerType = 'relative'

var shift = 5s

0s

var crit = 0.5

var data = stream
    |from()
        .database(db)
        .retentionPolicy(rp)
        .measurement(measurement)
        .groupBy(groupBy)
        .where(whereFilter)
    |eval(lambda: "temp_f")
        .as('value')
    |window()
      .period(3s)
      .every(1s)
      .align()

var max = data|max('value').as('value')

var min = data|min('value').as('value')

var trigger = max
    |join(min)
        .as('max', 'min')
        .tolerance(100ms)
    |eval(lambda: float("max.value" - "min.value"))
        .keep()
        .as('chng')
    |log()
      .level('ERROR')

trigger
    |influxDBOut()
       .create()
       .database('derived_iotdata')
       .measurement('influxdata_sensors')

trigger
    |alert()
        .crit(lambda: abs("chng") > crit)
        .stateChangesOnly()
        .message(message)
        .id(idVar)
        .idTag(idTag)
        .levelTag(levelTag)
        .messageField(messageField)
        .durationField(durationField)
        .post('http://davidgs.com:1880/sensor-reading')

trigger
    |influxDBOut()
        .create()
        .database(outputDB)
        .retentionPolicy(outputRP)
        .measurement(outputMeasurement)
        .tag('alertName', name)
        .tag('triggerType', triggerType)

trigger
    |httpOut('output')
```
## other_sh.tick 
```
// kapacitor define nnd_net_stream -type stream -tick nnd_net.tick -dbrp telegraf.rp_2w
var db = 'telegraf'
var rp = 'rp_2w'
var measurement = 'net'
var outputDB = 'chronograf'
var outputRP = 'rp_24h'
var outputMeasurement = 'nnd_net_stream'

var netdata = stream
    |from()
        .database(db)
        .retentionPolicy(rp)
        .measurement(measurement)
        // .where(lambda: "bytes_recv" > 0)

        // .where(lambda: "bytes_sent" > 0)

        // .where(lambda: "packets_sent" > 0)
        .groupBy(*)
    |eval(lambda: float("bytes_recv"))
        .as('bytes_recv_f')
        .keep()
    |eval(lambda: float("bytes_sent"))
        .as('bytes_sent_f')
        .keep()
    |eval(lambda: float("packets_recv"))
        .as('packets_recv_f')
        .keep()
    |eval(lambda: float("packets_sent"))
        .as('packets_sent_f')
        .keep()
    // |eval(lambda: float("drop_in"))

    // .as('drop_in_f')

    // .keep()

    // |eval(lambda: float("drop_out"))

    // .as('drop_out_f')

    // .keep()

    // |eval(lambda: float("err_in"))

    // .as('err_in_f')

    // .keep()

    // |eval(lambda: float("err_out"))

    // .as('err_out_f')

    // .keep()
    |window()
        .period(20s)
        .every(10s)
        .align()

netdata
    |derivative('bytes_recv_f')
        .unit(1s)
        .nonNegative()
        .as('bytes_recv_d')

netdata
    |derivative('bytes_sent_f')
        .unit(1s)
        .nonNegative()
        .as('bytes_sent_d')

netdata
    |derivative('packets_recv_f')
        .unit(1s)
        .nonNegative()
        .as('packets_recv_d')

netdata
    |derivative('packets_sent_f')
        .unit(1s)
        .nonNegative()
        .as('packets_sent_d')
    // netdata

    // |derivative('drop_in_f')

    // .unit(1s)

    // .nonNegative()

    // .as('drop_in_d')

    // netdata

    // |derivative('drop_sent_f')

    // .unit(1s)

    // .nonNegative()

    // .as('drop_sent_d')

    // netdata

    // |derivative('err_in_f')

    // .unit(1s)

    // .nonNegative()

    // .as('err_in_d')

    // netdata

    // |derivative('err_out_f')

    // .unit(1s)

    // .nonNegative()

    // .as('err_out_d')
    |influxDBOut()
        .database(outputDB)
        .retentionPolicy(outputRP)
        .measurement(outputMeasurement)
        .precision('s')
```
## panic.tick 
```
var measurement = 'event_trend2'

var period = 1m

var every = 20s

var data = batch
    |query('SELECT count(event_id) as alert_count FROM "azeti"."autogen".State')
        .period(period)
        .every(every)
        .groupBy(time(every), *)
        .fill(0)

var mean_data = data
    |mean('alert_count')
        .as('mean')

var deriv_data = data
    |derivative('alert_count')
        .unit(every)
        .nonNegative()
        .as('derivative')
    |last('derivative')
        .as('derivative')

mean_data
    |join(deriv_data)
        .as('mean', 'deriv')
    |influxDBOut()
        .database('_azeti_analytics')
        .measurement(measurement)

mean_data
    |httpOut('result')
```
## record_series_count.tick 
```
// Fire an alert when series count exceeds threshold
// Also rewrite downsampled points back to _internal database
var period = 10s
var every = 10s
var delta = 1s

var warn_threshold = 800000

var data = stream
    |from()
    .database('_internal')
    .measurement('database')
    .groupBy('hostname')
    |window()
    .period(period)
    .every(every)
    |default()
    .field('numSeries', 0)
    |sum('numSeries')
    .as('totalNumSeries')

data
    |alert()
    .id('high_series_count/{{ index .Tags "hostname" }}')
    .message('High series count for host: {{ index .Tags "hostname" }}')
    .details('')
    .warn(lambda: "totalNumSeries" > warn_threshold)
    .stateChangesOnly()
    .log('/var/log/kapacitor/high_series_count.log')

data
    |influxDBOut()
    .database('_internal')
    .retentionPolicy('monitor')
    .measurement('database')
    .tag('source', 'kapacitor')
```
## record_throughput.tick 
```
// Fire an alert when write throughput exceeds threshold
// Also rewrite downsampled points back to _internal database
var period = 10s
var every = 10s
var delta = 1s

var warn_threshold = 500000

var data = stream
    |from()
    .database('_internal')
    .measurement('write')
    |window()
    .period(period)
    .every(every)
    |default()
    .field('pointReq', 0)
    |max('pointReq')
    .as('max')
    |derivative('max')
    .unit(delta)
    .nonNegative()
    .as('pointsPerSecond')

data
    |alert()
    .id('high_throughput/{{ index .Tags "hostname" }}')
    .message('High write throughput on host: {{ index .Tags "hostname" }}')
    .details('')
    .warn(lambda: "pointsPerSecond" > warn_threshold)
    .stateChangesOnly()
    .log('/var/log/kapacitor/high_throughput.log')

data
    |influxDBOut()
    .database('_internal')
    .retentionPolicy('monitor')
    .measurement('write')
    .tag('source', 'kapacitor')
```
## rename.tick 
```
///////////////////////////////
// writes are failing
var data = stream
            |from()
              .measurement('shard')
            |groupBy('database')
            |window()
              .period(10s)
              .every(10s)

var writeOk = data
                |max('writePointsOk')
                  .as('maxWritePointsOk')
                |derivative('maxWritePointsOk')
                  .unit(1s)
                  .nonNegative()
                  .as('derivativeMaxWritePointsOk')

var writeFail = data
                  |max('writePointsFail')
                    .as('maxWritePointsFail')
                  |derivative('maxWritePointsFail')
                    .unit(1s)
                    .nonNegative()
                    .as('derivativeMaxWritePointsFail')

writeOk
  |join(writeFail)
    .as('ok', 'fail')
    .tolerance(1s)
    .fill(0.0)
    .streamName('write_ok_fail')
  |eval(lambda: "fail.derivativeMaxWritePointsFail" / ("ok.derivativeMaxWritePointsOk" + "fail.derivativeMaxWritePointsFail"))
    .as('percentFailedWrites')
  |alert()
    .id('kapacitor/{{ index .Tags "datbase" }}') // ??
    .message('{{ .ID }} is {{ .Level }} value:{{index .Fields "value" }}') // ?
    .info(lambda: "derivativeMeanWritePointOk" > 10) // ??
    .warn(lambda: "derivativeMeanWritePointOk" > 30) // ??
    .crit(lambda: "derivativeMeanWritePointOk" > 90) // ??
    .slack() // ??
      .channel('#alerts')
  |influxDBOut()
    .database('internal_stats') // ??
    .retentionPolicy('myrp') // ??
    .measurement('percent_failed_disk_writes')
    .tag('kapacitor', 'true')




/////////////////////////////////////////////////////////
// you currently have high throughput

stream
  |from()
    .measurement('write')
  |max('pointReq')
    .as('maxPointReq')
  |derivative('maxPointReq')
    .unit(1s)
    .nonNegative()
    .as('pointsPerSecond')
  |alert()
    .message('idk?')
    .info(lambda: 'pointsPerSecond' > 150000)
    .warn(lambda: 'pointsPerSecond' > 300000)
    .crit(lambda: 'pointsPerSecond' > 500000)
  |influxDBOut()
    .database('internal_stats') // ??
    .retentionPolicy('myrp') // ??
    .measurement('throughput')
    .tag('kapacitor', 'true')

////////////////////////////////////////////////////////////
///// too many series

stream
  |from()
    .measurement('database')
  |sum('numSeries')
    .as('totalNumSeries')
  |alert()
    .message('you have this many series')
    .warn(lambda: 'totalNumSeries' > 1000000)
    .crit(lambda: 'totalNumSeries' > 10000000)
  |influxDBOut()
    .database('internal_stats') // ??
    .retentionPolicy('myrp') // ??
    .measurement('series')
    .tag('kapacitor', 'true')


```
## rollup.tick 
```
// DEPLOYAS:stream
// DBRP:telegraf.default
// Which measurement to consume
var measurement = 'cpu'

// Which database to consume
var db = 'telegraf'

// Which retention policy to consume
var rp = 'autogen'

// Which field to process
var field = 'usage_user'

var data = stream
    |from()
        .database('telegraf')
        .retentionPolicy('default')
        .measurement(measurement)
        .groupBy(*)

// 1m rollup
data
    |window()
        .period(1m)
        .every(1m)
    |mean(field)
        .as(field)
    |influxDBOut()
        .database('telegaf')
        .retentionPolicy('autogen')
        .measurement(measurement)
        .tag('resolution', '1m')
        .create()

// 5m rollup
data
    |window()
        .period(5m)
        .every(5m)
    |mean(field)
        .as(field)
    |influxDBOut()
        .database('telegaf')
        .retentionPolicy('autogen')
        .measurement(measurement)
        .tag('resolution', '5m')
        .create()
```
## s.tick 
```
var db = 'telegraf'
var rp = 'rp_2w'
var measurement = 'net'
var outputDB = 'chronograf'
var outputRP = 'rp_2w'
var outputMeasurement = 'nnd_net'

batch
|query('select non_negative_derivative(*) as nnd from telegraf.rp_2w.net')
  .period(20s)
  .every(20s)
  .groupBy(*)
|influxDBOut()
  .database(outputDB)
  .retentionPolicy(outputRP)
  .measurement(outputMeasurement)
  .precision('s')
```
## shift.tick 
```
dbrp "telegraf"."autogen"

batch
    |query('SELECT * FROM "telegraf"."autogen"."cpu" GROUP BY *')
        .period(1m)
        .every(1m)
        .groupBy(*)
    |shift(10m)
    |influxDBOut()
        .database('telegraf')
        .retentionPolicy('newrp2')
```
## st.tick 
```
stream
    |from()
        .measurement('cpu')
        .database('telegraf')
        .retentionPolicy('autogen')
        .where(lambda: "cpu" == 'cpu-total')
        .groupBy('host')
    |log()
    |deadman(0.0, 10s)
        .post('http://localhost:5555')
```
## system_down.tick 
```
// This TICKscript assumes that you have telegraf running and the Kapacitor slack plugin enabled
var period = 120s
var every = 60s

var sys_data = stream
    |from()
      .database('telegraf')
      .measurement('system')
      .groupBy('host')
    |window()
      .period(period)
      .every(every)

sys_data|deadman(1.0, period)
    .id('deadman/{{ index .Tags "host" }}')
    .message('Node {{ index .Tags "host" }} {{ if ne .Level "OK" }}has not responded in 120s!{{ else }}is back online.{{ end }}')
    .stateChangesOnly()
    .slack()
```
## telemetry_retention.tick 
```
batch
    |query('''SELECT * FROM "production"."telemetry_retention"."telemetry"''')
        .cron('0 20 11 * * * *')
        .period(4d)
        .groupBy(*)
    |count('WK_Zen')
    |log()
    |shift(24h)
    |influxDBOut()
        .database('work')
        .retentionPolicy('telemetry_retention')
        .measurement('telemetry')
        .precision('s')
```
## test_auto.tick 
```
var testname = 'Test'

// TestAutomation,testname=A,clustername=1 testStatus=0,other=10
// TestAutomation,testname=B,clustername=2 other=10
// TestAutomation,testname=B,clustername=2 other=10
// TestAutomation,testname=A,clustername=1 testStatus=0
// TestAutomation,testname=B,clustername=2 testStatus=0,other=10
var success = batch
    |query('select testStatus from "test"."autogen"."TestAutomation"')
        .period(20s)
        .every(10s)
        .groupBy('testname', 'clustername')
    |where(lambda: "testStatus" == 0)
    |count('testStatus')
        .as('count')
    |where(lambda: "count" > 2)
    |stats(10s)
    |log()
    |alert()
        .message('{{ .Level }} | Continous Fails US | {{ index .Tags "clustername" }} ' + string(testname))
        .crit(lambda: "emitted" > 0)
        .stateChangesOnly(1m)
        .log('/tmp/marathon_metrics_batch_alert_us.log')
```
## tick.tick 
```
var tolerance = 0.0001
var sq = stream
 |from()
   .measurement('m')
 |eval(lambda: 0.5 * ("x" + ("s" / "x")))
  .as('xn')
  .keep()

sq|where(lambda: abs("x" - "xn") < tolerance)
 |log()

sq|where(lambda: abs("x" - "xn") > tolerance)
  |eval(lambda: "xn")
   .as('x')
   .keep('s','x')
 |kapacitorLoopback()
  .database('mydb2')
  .retentionPolicy('autogen')
```
## time.tick 
```
var secondsPerMinute = 60
var secondsPerHour   = 60 * 60
var secondsPerDay    = 24 * secondsPerHour
var secondsPerWeek   = 7 * secondsPerDay
stream
  |from()
    .measurement('cpu')
  |eval(lambda: unixNano("time") - "usage_time")
    .as('t2')
  |log()
```
## week_on_week.tick 
```
// Query the current minute
var this_minute = batch
    |query('''
SELECT count("usage_user") FROM "telegraf"."autogen"."cpu"
''')
        .period(1m)
        .every(1m)

var last_minute = batch
    |query('''
SELECT count("usage_user") FROM "telegraf"."autogen"."cpu"
''')
        .period(1m)
        .every(1m)
        // This is what gives us the previous miniute of data
        .offset(24h)
    // we need to shift the previous minute forward
    |shift(1m)

this_minute
    |join(last_minute)
        .as('this', 'last')
    |log()
```
## working-issue1192.tick 
```
batch
    |query('select mean(used_percent) from "telegraf"."autogen"."mem"')
      .groupBy('tag1','missing-tag')
      .period(30s)
      .every(30s)
    // By using last here the edge type switches from batch to stream and the bug only exists in the batch handling, so this is a hack workaround. 
    // Using last doesn't change anything since the batch query only returns a single aggregated point anyways.
    |last('mean')
       .as('mean')
    |eval(lambda: trunc("mean"))
      .as('mean')
    |default()
      .tag('missing-tag', 'unknown')
    |log()
    |alert()
        .id('kapacitor/{{ index .Tags "missing-tag" }}/...')
```
