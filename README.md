Data Pipelining and Real-Time Data Streaming with Apache Flink and Flink SQL
Objective
This project demonstrates a comprehensive pipeline to handle streaming data by integrating Confluent schemas, Kafka topics, Apache Flink, and Elasticsearch. The primary goal is to:

Generate and process data using Confluent schemas.
Feed the processed data into multiple Kafka topics.
Stream and analyze the data in real-time using Apache Flink and Flink SQL.
Create a dashboard for visualization and derive insights using Kibana.
Dataset Description
Three datasets were generated from Confluent Avro schemas and subsequently transformed into JSON format. The datasets represent different entities of a gaming platform:

Game Rooms (bda1)

Represents information about game rooms created within the platform.
Schema:
id: Unique identifier for the game room.
room_name: Name/type of the game room (e.g., "Classic -- Skilled").
created_date: Timestamp of when the game room was created.
Example Data:

{"id":2975,"room_name":"Classic -- Skilled","created_date":1609459200000}
{"id":3648,"room_name":"Arcade -- Expert","created_date":1609459400000}
Player Activity (bda2)

Represents gameplay activity for each player.
Schema:
player_id: Unique identifier for the player.
game_room_id: Game room ID where the player is playing.
points: Points scored by the player.
coordinates: Player's position in the game (in [x, y] format).
Example Data:

{"player_id":1001,"game_room_id":1529,"points":189,"coordinates":"[52,27]"}
{"player_id":1094,"game_room_id":2675,"points":403,"coordinates":"[80,52]"}
Players (bda3)

Contains information about the players registered on the platform.
Schema:
player_id: Unique identifier for the player.
player_name: Name of the player.
ip: IP address of the player.
Example Data:

{"player_id":1029,"player_name":"Rossie Hobben","ip":"151.228.5.139"}
{"player_id":1076,"player_name":"Lea Murrish","ip":"34.10.81.209"}
Data Pipeline Process
1. Data Generation
Data was generated from Confluent Avro schemas using the following commands:

./gendata.sh gaming_games.avro xyz1.json 10000
./gendata.sh gaming_player_activity.avro xyz2.json 10000
./gendata.sh gaming_players.avro xyz3.json 10000
gendata.sh: Extracts data from the Avro schema and converts it to JSON.
Input: Avro schema.
Output: JSON file with random synthetic data.
2. Data Transformation
The JSON files were transformed into key-value pairs using the convert.py script to prepare them for Kafka ingestion:

python $HOME/Documents/fake/convert.py
3. Kafka Ingestion
Data from the JSON files was streamed into Kafka topics using gen_sample.sh:

./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz1.json | kafkacat -b localhost:9092 -t bda1 -K: -P
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz2.json | kafkacat -b localhost:9092 -t bda2 -K: -P
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz3.json | kafkacat -b localhost:9092 -t bda3 -K: -P
gen_sample.sh: Streams transformed JSON data to Kafka topics.
Kafka Topics Created: bda1, bda2, bda3.
4. Real-Time Analysis with Apache Flink
Apache Flink and Flink SQL were used for real-time streaming analysis.

Table Creation in Flink SQL:
Game Rooms Table

CREATE TABLE game_rooms (
    id BIGINT,
    room_name STRING,
    created_date TIMESTAMP(3),
    readable_created_date AS TO_TIMESTAMP(FROM_UNIXTIME(UNIX_TIMESTAMP(created_date))),
    WATERMARK FOR created_date AS created_date - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'bda1',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);
Player Activity Table

CREATE TABLE player_activity (
    player_id BIGINT,
    game_room_id BIGINT,
    points INT,
    coordinates STRING
) WITH (
    'connector' = 'kafka',
    'topic' = 'bda2',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json'
);
Players Table

CREATE TABLE players (
    player_id BIGINT,
    player_name STRING,
    ip STRING
) WITH (
    'connector' = 'kafka',
    'topic' = 'bda3',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);
Enriched Views
A consolidated view was created by joining player_activity, game_rooms, and players:

CREATE VIEW enriched_player_game_activity AS
SELECT 
    PA.player_id,
    P.player_name,
    P.ip,
    PA.game_room_id,
    GR.room_name,
    GR.created_date,
    PA.points,
    PA.coordinates
FROM player_activity AS PA
LEFT JOIN game_rooms AS GR ON PA.game_room_id = GR.id
LEFT JOIN players AS P ON PA.player_id = P.player_id;
5. Data Insertion into Elasticsearch
Before creating the Kibana dashboard, the enriched data was inserted into an Elasticsearch index.

Data Insertion into Elasticsearch:
Elasticsearch Table Creation

CREATE TABLE player_game_dashboard (
    player_id BIGINT,
    player_name STRING,
    ip STRING,
    game_room_id BIGINT,
    room_name STRING,
    created_date BIGINT,
    points INT,
    coordinates STRING
) WITH (
    'connector' = 'elasticsearch-7',
    'hosts' = 'http://elasticsearch:9200',
    'index' = 'player_game_dashboard',
    'format' = 'json'
);
Inserting Data into Elasticsearch Index
The data from the enriched_player_game_activity view was inserted into the player_game_dashboard table:

INSERT INTO player_game_dashboard
SELECT 
    player_id,
    player_name,
    ip,
    game_room_id,
    room_name,
    created_date,
    points,
    coordinates
FROM enriched_player_game_activity;
6. Dashboard Creation with Kibana
The data in the player_game_dashboard index was visualized using Kibana.

An index pattern for player_game_dashboard was created in Kibana.
A detailed dashboard was designed to display player activity, game room details, and points scored.
Dashboard Analysis
