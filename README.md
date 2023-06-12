synth generate IoT/ --collection sensors_data --size 1 --to jsonl:sensors_data.jsonl
cat sensor_data.jsonl | mosquitto_pub -h test.mosquitto.org -t mqtt-topic --stdin-line

