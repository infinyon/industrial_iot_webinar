# Pre-requisites
Install fluvio 
`curl -fsS https://packages.fluvio.io/v1/install.sh | bash`

Register for a free account at [fluvio.io](https://21264918.hs-sites.com/infinyon-cloud-onboarding-experience?utm_campaign=global%20cloud%20link&utm_source=webinar&utm_medium=global%20button&utm_term=infinyon%20cloud&utm_content=cloud-registration)

Install mosquitto client (for command line interface) 
`brew install mosquitto`

Install local Redis Stack 
```
docker run redis/redis-stack:latest
```
or register in [redis cloud](https://app.redislabs.com/#/)

Install Grafana:
```
docker run -p 3000:3000 --name=grafana -e "GF_INSTALL_PLUGINS=redis-explorer-app" grafana/grafana
```

# For Development 
install fluvio Smart Modules Development Kit (SMDK)
```
fluvio install smdk 
```
and connector development kit
```
fluvio install cdk
```
# MQTT to RedisTimeseries  
Create MQTT connector using a simple config file `mqtt_simple_config.yml`:

```
apiVersion: 0.1.0
meta:
  version: 0.2.2
  name: my-mqtt-connector
  type: mqtt-source
  topic: mqtt-topic
  create-topic: true
mqtt:
  url: "mqtt://test.mosquitto.org/"
  topic: "am-mqtt-to-fluvio"
  client_id: "am_mqtt_conn"
  timeout:
    secs: 30
    nanos: 0
  payload_output_type: json

```
Create MQTT connector
```
fluvio cloud connector -c mqtt_simple_config.yml
```
Test MQTT connector using 
```
cat ./sample-data/pre_processed_sensors.jsonl | mosquitto_pub -h test.mosquitto.org -t am-mqtt-to-fluvio --stdin-line
```

Check the last 10 records using Fluvio consume CLI
```
fluvio consume mqtt-topic -T 10                            
Consuming records from 'mqtt-topic' starting 10 from the end of log
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"timestamp":"1686585855332","value":"1.9799302149530007","key":"O_w_BLO_voltage"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"value":"59","key":"O_w_BHR_voltage","timestamp":"1686585855332"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"value":"776","key":"I_w_BHL_Weg","timestamp":"1686585855332"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"key":"I_w_BHR_Weg","timestamp":"1686585855332","value":"-937"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"value":"11","key":"O_w_BRU_voltage","timestamp":"1686585855332"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"key":"O_w_HL_voltage","timestamp":"1686585855332","value":"96"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"key":"O_w_HR_voltage","value":"234","timestamp":"1686585855332"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"value":"-356.9000000000001","key":"I_w_HL_Weg","timestamp":"1686585855332"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"key":"O_w_BHL_voltage","timestamp":"1686585855332","value":"-21.1105873657448"}}
{"mqtt_topic":"am-mqtt-to-fluvio","payload":{"timestamp":"1686585855332","key":"O_w_BLO_power","value":"9876"}}
```
The data inside topic is nearly what we need to store in RedisTimeseries, except it's now inside "payload" field. We need to extract it and convert to RedisTimeseries format.
A convenient way to do it using Jolt smart module:
```
transforms:
  - uses: infinyon/jolt@0.2.0
    with:
      spec:
        - operation: shift
          spec: 
            payload:
              key: "key"
              value: "value"
              timestamp: "timestamp"
```
above configuration is stored in `jolt.yaml` file and defines "shift" transformation which will copy "key", "value" and "timestamp" into the output record.

Download smartmodule from Infinyon Cloud:
```
fluvio hub download infinyon/jolt@0.2.0
```
and check transformation:
```
fluvio consume mqtt-topic -T 10 --transforms-file jolt.yaml
Consuming records from 'mqtt-topic' starting 10 from the end of log
{"timestamp":"1686585855332","value":"1.9799302149530007","key":"O_w_BLO_voltage"}
{"value":"59","key":"O_w_BHR_voltage","timestamp":"1686585855332"}
{"value":"776","key":"I_w_BHL_Weg","timestamp":"1686585855332"}
{"key":"I_w_BHR_Weg","timestamp":"1686585855332","value":"-937"}
{"value":"11","key":"O_w_BRU_voltage","timestamp":"1686585855332"}
{"key":"O_w_HL_voltage","timestamp":"1686585855332","value":"96"}
{"key":"O_w_HR_voltage","value":"234","timestamp":"1686585855332"}
{"value":"-356.9000000000001","key":"I_w_HL_Weg","timestamp":"1686585855332"}
{"key":"O_w_BHL_voltage","timestamp":"1686585855332","value":"-21.1105873657448"}
{"timestamp":"1686585855332","key":"O_w_BLO_power","value":"9876"}
```

Now checkout labs-redis-sink-connector via git:
```
git submodule update --init --update --recursive
```
and build it:
```
cdk build
```
Create configuration file config-example_transform.yaml:
```
meta:
  version: 0.1.0
  name: my-redis-connector-sink-connector-transform
  type: redis-sink
  topic: mqtt-topic
redis:
  prefix: mqtt-topic
  url:
    secret:
      name: "REDIS_URL"
  operation: "TS.ADD"
transforms:
  - uses: infinyon/jolt@0.2.0
    with:
      spec:
        - operation: shift
          spec: 
            payload:
              key: "key"
              value: "value"
              timestamp: "timestamp"
``` 
Test connector:
```
cdk test -c config-example_transform.yaml --secrets secrets.txt
```
or deploy locally:
```
cdk deploy start --config config-example_transform.yaml  --secrets secrets.txt
```
config-example_transform.yaml contains the same jolt transformation we have tested earlier, labs-redis-sink-connector/config-example_mqtt.yaml is the example of Redis Connector configuration without transformation. Since TS.ADD command requires key, value and timestamp fields, the connector will fail without transformation.

re-do end-to-end test:
```
cat ./sample-data/pre_processed_sensors.jsonl | mosquitto_pub -h test.mosquitto.org -t am-mqtt-to-fluvio --stdin-line
```
We can now monitor pipeline in Grafana: 
```
docker run -p 3000:3000 --name=grafana -e "GF_INSTALL_PLUGINS=redis-explorer-app" grafana/grafana
``` 
add Redis Data Store and create dashboard with Redis Dashboard and TS.RANGE command. 
