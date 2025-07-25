# Technical Design & Architecture Document Draft: IoT sensor simulator for testing and development

**Author**: Dmitrii Khramtsov

**App name**: SensSim IoT

_For brevity, the application’s name (“SensSim IoT”) will be referred to as “the app” throughout this document._

![logo](assets/logo_iot_sensors.png)

<!-- toc -->

- [Purpose and Context](#purpose-and-context)
- [Sensor Simulation Engine](#sensor-simulation-engine)
  - [Sensor Data Format](#sensor-data-format)
  - [Simulation Orchestration](#simulation-orchestration)
  - [Sensor Emission Timing](#sensor-emission-timing)
  - [Sensor Simulation Process](#sensor-simulation-process)
  - [Data Emission](#data-emission)
    - [Sensor API Specification](#sensor-api-specification)
    - [Authentication Options](#authentication-options)
  - [Saving signal data to Clickhouse db](#saving-signal-data-to-clickhouse-db)
- [General project description](#general-project-description)
  - [Project structure](#project-structure)
  - [Tooling](#tooling)
  - [CLI parameters](#cli-parameters)
- [Potential extensions for sending signals](#potential-extensions-for-sending-signals)
  - [gRPC gateway](#grpc-gateway)
  - [MQTT Integration](#mqtt-integration)
  - [Publishing to Apache Kafka](#publishing-to-apache-kafka)
- [Integration with microservice infrastructure](#integration-with-microservice-infrastructure)
  - [Endpoint for Prometheus](#endpoint-for-prometheus)
  - [Structured logging](#structured-logging)

<!-- tocstop -->

# Purpose and Context

Many industrial software solutions such as real-time analytics systems, SCADA, and Industrial IoT Alert Systems use data from IoT sensors. Development of such systems requires setting up testing environments that simulate real-life behaviour of these sensors.

Having a software development team on a factory is in most cases not a viable option, since factories can be distant and working directly with sensors during the development process can interfere with or even disrupt factory operations.

This solution is aiming to simulate smart sensors and IoT gateways which send data in JSON format, particularly industrial pressure, temperature, and voltage sensors.

# Sensor Simulation Engine

Solution will be developed in Go.

Each sensor runs in a separate goroutine which enables simulation of natural flow of signals, since sensors in real life send data independently.

## Sensor Data Format

Proposed structure for sensor data package. Here we will use JSON structure and corresponding constructor for smart sensor data which in future can be customized and adapted for particular sensor model.

```go
type Sensor struct {  
    SensorID  string    `json:"sensorId"`  
    Parameter ValueType `json:"type"`  
    Unit      string    `json:"unit"`  
    Timestamp time.Time `json:"timestamp"`  
    Value     float64   `json:"value"`  
}  

func New(id string, param ValueType, unit string) *Sensor {  
    return &Sensor{  
       SensorID:  id,  
       Parameter: param,  
       Unit:      unit,  
    }  
}
```

Example of JSON data sent by industrial smart pressure sensor:

```json
{ 
 "sensorId": "PS-003-PNEUMATIC-LINE-1", 
 "type": "PRESSURE", 
 "unit": "PSI", 
 "timestamp": "2024-07-22T14:30:30.000Z", 
 "value": 87.2 
}
```

Each sensor will have the following settings: emitting frequency and location (position).

> _Example: pressure sensor, location: Input valve, frequency: 100 Hz._

Sensors frequency depends on a speed of measurements which depends on a nature of measured process with temperature sensors having typically lower measurement speed compared to voltage sensors due to temperature changes being slower compared to voltage fluctuations. Also settings will include device id to assign settings to a particular sensor.

```go
type Settings struct {  
    SensorID  string  
    Frequency float64  
    Location  string  
}  
  
func NewSettings(id string, freq float64, location string) *Settings {  
    return &Settings{  
       SensorID: id,  
       // Hz  
       Frequency: freq,  
       Location:  location,  
    }  
}
```

## Simulation Orchestration

All sensors should start and finish working at the same time. Goroutines will be spawned in a loop and simulation ending can be managed by using context `WithTimeout`.

```go
ctx, cancel := context.WithTimeout(context.Background(), duration)  

defer cancel()  

var wg sync.WaitGroup

...

// spawning goroutines, one for each sensor
for i := 0; i < sensorsNum; i++ {    
    wg.Add(1)  
    go simulator.Simulate(...)  
}  
wg.Wait()
```

## Sensor Emission Timing

For frequecny simulation `ticker` will be used.

```go
for {  
    select {  
    case <-ticker.C:  
       // generate data
       // emit data
    case <-ctx.Done():  
       // finish simulation
       return  
    }  
}
```

## Sensor Simulation Process

- Spawn goroutine with sensor settings

- Inside each goroutine infinite loop `for {...}` works with `select {...}`
  - if time for the next data emission comes - emit data
  - if simulation time is over then finish simulation (`ctx.Done()`)

## Data Emission

Emitting data should be implemented separately from data generation and sensor simulation.
Emitter should be implemented as interface to make it easier to stub/mock it with testing tools like `httptest`.

Emitter interface and struct for sending sensor data via http:

```go
type Emitter interface {  
    Emit(sensor.Sensor) error  
}  

type HTTPEmitter struct {  
    client   *http.Client  
    endpoint string  
}
```

**Note:** expect server to respond with 202 code in case of success. This happens because real-time sensor monitoring/analytics platforms don't typically process incoming data instantly but retransmit it further (for example publishing it to Kafka), hence code 202 is more often used compared to 200.

### Sensor API Specification

**Endpoint:** `POST /api/v1/sensors`

**Summary**
Submit sensor data.

**Description**
Submit sensor data to the server with details about the sensor, measurement type, unit, timestamp, and value.

**Request Body**

- **Content Type**: `application/json`
- **Required Properties**:
  - `sensorId` (string): Unique identifier for the sensor (e.g., "sensor_123").
  - `type` (string): Type of parameter measured (e.g., "temperature").
  - `unit` (string): Unit of measurement (e.g., "Celsius").
  - `timestamp` (string, date-time): Time of the measurement (e.g., "2025-07-22T14:48:00Z").
  - `value` (number, float): Measured value (e.g., 23.5).

**Responses**

- **202 Accepted**: Request successfully received and accepted. Returns an empty JSON object.
- **400 Bad Request**: Invalid request format or parameters. Returns `{ "error": "Invalid request format" }`.
- **429 Too Many Requests**: Rate limit exceeded. Returns `{ "error": "Rate limit exceeded" }`.
- **500 Internal Server Error**: Unexpected server error. Returns `{ "error": "Internal server error" }`.
- **503 Service Unavailable**: Server is overloaded. Returns `{ "error": "Server is overloaded" }`.

⬇️ [OpenAPI endpoint specs](/assets/target_endpoint.openapi.yaml)

### Authentication Options

We don't expect that target server will require authentication during development stage. If there will be a need for authentication, there are numerous ways how IoT smart sensors on factories can be authenticated: from **JWT** tokens and **API keys** to more niche methods like **mTLS**. Overall it make sense to implement auth method when necessary depending on which method will be actually used.

## Saving signal data to Clickhouse db

Clickhouse database is used for storing real time sensor data in IoT analytics systems. During development the app can be used to fill it with mock data.

Proposed clickhouse table structure for storing real time sensor data. `LowCardinality(String)` will enable to keep table denormalized for quicker aggregate queries while saving storage by using `LowCardinality`  which enables to store repetitive data compactly.

```sql
CREATE TABLE sensors_data (
    id UUID,
    timestamp DateTime64(6, 'UTC'),
    device_id LowCardinality(String),
    metric_name LowCardinality(String),
    value Float64,
    unit LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (device_id, metric_name, timestamp);
```

Since clickhouse is more effective at **saving data in bulk**, mechanism for reliable data saving in bulk should be implemented. We **cannot use the same buffer** for collecting sensor simulation data and use it for flushing data into clickhouse since new data will income in buffer at the same time as data from buffer being saved into DB.

**Solution**
Consider decoupled 3 channel system, let there be channels A, B, C (B and C are buffered):

- channel A receives real time data from sensor simulation goroutines.
- channel B collects data from buffer A, when its buffer is full it sends data to channel C (`C <- B`)
- Channel B buffer is cleared (`B = make([]Sensor, 0, batchSize)`)
- `saveToClickhouse(C <-chan []Sensor)` goroutine use data from channel C and saves it in bulk into Clickhouse

# General project description

## Project structure

```
Makefile
README.md
/bin
/cmd
/internals
 /configs
 /emitter
 /sensor
 /data
```

**Notes**
Check the sample [Makefile](/assets/Makefile) from previous projects

**bin** - for compiled app binary and tooling: linter, debugger, etc.

## Tooling

- **Linter:** `golangci-lint`
- **Testing:** `testify`, `httptest`
- **Debugger:** `delve`
- **Profiler:** `pprof`
- **Documentation:** `godoc`, `go-swagger`(optional)
- **Formatters:** `gofmt`, `goimports`
- **Licenses:** `go-licenses`

## CLI parameters

Simulation app will be implemented as a CLI app. `spf13/pflag` will be used instead of standard `flag` due to its support of shorthands for flags like `-w`.

**Dry-run flag support.** Will be supported by using corresponding flag `--dry-run`. In this case app will produce logs, generate data, yet no actually data will be sent. Use this for testing that app works correctly.

**Flags to Implement**

- `--max-workers`: maximum number of concurrent sensor simulators
- `--emitter-timeout`: emitter server timeout
- `--target-address`: target server address to send simulated sensor data
- `--target-port`: target server port to send simulated sensor data
- `--target-path`: target server API endpoint path for sensor data submission
- `--sim-duration`: simulation duration in seconds
- `--expected-code`: success HTTP response code from target server
- `--dry-run`: no actual requests will be sent, only logs will be produced

# Potential extensions for sending signals

Even driven industrial platforms receives data from sensors using separate service which is responsible typically for data validation and passing it further but not for its processing. It passes it either to the next module directly (for example via gRPC) or publish it to messaging broker.

```
sensor data -> ingestion_service <-gRPC-> analytics_service

sensor data -> ingestion_service <-> Kafka <-> analytics_service
```

During development stage there can be a situation where only parts of the system are implemented or there is a need to test separate parts of a software directly. For such case it makes sense to add to the app an ability to interact with deeper levels of software directly omitting ingestion_service:

```
sensor_simulator -> gRPC -> analytics_service

sensor_simulator -> Kafka -> analytics_service
```

Here I will lay out main points about approaching implementation of these extensions.

## gRPC gateway

**Tooling**

- `protoc`: parses `.proto` file and make it suitable for processing via other tools
- `protoc-gen-go`:  generates Go code for structs, serialization/deserialization
- `protoc-gen-go-grpc`: generates interfaces

protobuf description of sensor data

```protobuf
syntax = "proto3";

import "google/protobuf/timestamp.proto";

message Sensor {
  string sensor_id = 1;
  string parameter = 2;
  string unit = 3;
  google.protobuf.Timestamp timestamp = 4;
  double value = 5;
}
```

## MQTT Integration

Message Queuing Telemetry Transport is used for communication of IoT systems, including industrial sensors. While in this case usage of MQTT is not expected, in case such necessity will appear an MQTT Go client [paho.mqtt.golang](https://github.com/eclipse-paho/paho.mqtt.golang) can be used.

## Publishing to Apache Kafka

Use `sarama` for publishing data to Kafka.

Expect Kafka to be set up with topic per physical value, not per sensor. So, values from all pressure sensors go to topic `pressure`, all temperature sensors data published in topic `temp` and so on.

Expect one broker in testing environment.

In sarama use `AsyncProducer`since IoT systems are set up to receive large volumes of data and aims for lower latency.

# Integration with microservice infrastructure

## Endpoint for Prometheus

Integration of app into the observability system by adding standard `/metrics` endpoint for Prometheus.

Data, provided for Prometheus may include:

- Disk, CPU, memory usage
- Number of requests sent, responses received

## Structured logging

Using structured logger like `zap` will make it easier integration with ELK stack.
