# Project:Sentimental Analysis of TwitterFeed (JSON format) for IronMan Movie using Hive

# Prep work
   - Dataset from  a set of sample Twitter data contained in a compressed (.zip) folder here:
    http://s3.amazonaws.com/hw-sandbox/tutorial13/SentimentFiles.zip
  - Upload via Hue to HDFS 
  - Download Json serial deserialer for Hive table upload as data is in JSON format 
  
# Commands
 Login to webconsole 

```sh
hadoop fs -copyToLocal /data/serde/json-serde-1.3-jar-with-dependencies.jar 
```

In Hadoop file system json data on tweets is avaialable.Open Hive and set the below parm to prevent hive checking for keywords so that fields/tables can have keyword 'name; etc.

```sh
set hive.support.sql11.reserved.keywords=false; 
add jar json-serde-1.3-jar-with-dependencies.jar; 
list jars;
create database 2017grc0_twitter_sentiment;
use 2017grc0_twitter_sentiment;
CREATE EXTERNAL TABLE tweets_raw (id BIGINT, 
   created_at STRING, 
   source STRING, 
   favorited BOOLEAN,  
   retweet_count BIGINT, 
   retweeted_status STRUCT<text:STRING,users:STRUCT <screen_name:STRING,name:STRING>>, 
   entities STRUCT<urls:ARRAY<STRUCT <expanded_url:STRING>>,
   user_mentions:ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
   hashtags:ARRAY<STRUCT<text:STRING>>>,
   text STRING,
   user STRUCT<screen_name:STRING,name:STRING,friends_count:INT,followers_count:INT,
   statuses_count:INT,verified:BOOLEAN,utc_offset:STRING,time_zone:STRING>,
   in_reply_to_screen_name STRING, year BIGINT, month BIGINT, day BIGINT, hour BIGINT) 
   ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe' WITH SERDEPROPERTIES
   ("ignore.malformed.json" = "true")
   LOCATION '/user/YOUR-USERNAME/SentimentFiles/SentimentFiles/upload/data/tweets_raw' ;
```

In Hue ,select Query editor and add the following

```sh	
ADD JAR hdfs:////data/serde/json-serde-1.3-jar-with-dependencies.jar
```
  
In Console - Hive 
CREATE table for Dictionary for the keywords and the sentiments (positive , negative)
```sh
	CREATE EXTERNAL TABLE dictionary (
        type string,
        length int,
        word string,
        pos string,
        stemmed string,
        polarity string
        )
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' 
    STORED AS TEXTFILE
    LOCATION '/user/YOUR-USERNAME/SentimentFiles/SentimentFiles/upload/data/dictionary';
```

CREATE table for time zone with  time-zone , country and comments

```sh
    CREATE EXTERNAL TABLE time_zone_map (
        time_zone string,
        country string,
        notes string
        )
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' 
    STORED AS TEXTFILE
    LOCATION '/user/YOUR-USERNAME/SentimentFiles/SentimentFiles/upload/data/time_zone_map';

   CREATE VIEW tweets_simple AS
          SELECT
         id,
        cast ( from_unixtime( unix_timestamp(concat( '2013 ', substring(created_at,5,15)), 
        'yyyy MMM dd hh:mm:ss')) as timestamp) ts,
         text,
         user.time_zone 
         FROM tweets_raw
        ;
```

Joining time_zone_map with tweets_simple

```sh
    CREATE VIEW tweets_clean AS
    SELECT
    id,
    ts,
    text,
    m.country 
    FROM tweets_simple t LEFT OUTER JOIN time_zone_map m ON t.time_zone = m.time_zone;

create view tweets_explode as select id, words from tweets_raw lateral view explode(sentences(lower(text))) dummy as words;

create view tweets_explode_words as select id, word from tweets_explode lateral view explode( words ) dummy as word ;

create table right_dictionary as select * from dictionary where word <> "stark" 
```
Dictionary uses word Stark as negative. But Tony Stark is the name of the char . Hence word is removed from new dictionary 

Calculating the polarity points 
```sh
create view tweets_polarity_pts as select 
    id, 
    tew.word, 
    case d.polarity 
    when  'negative' then -1
    when 'positive' then 1 
    else 0 end as polarity 
    from tweets_explode_words tew left outer join right_dictionary d on tew.word = d.word;

create table tweets_sentiment stored as orc as select 
    id, 
    case 
    when sum( polarity ) > 0 then 'positive' 
    when sum( polarity ) < 0 then 'negative'  
    else 'neutral' end as sentiment 
    from tweets_polarity_pts group by id;

CREATE TABLE tweetsbi 
    STORED AS ORC
    AS
    SELECT 
    t.*,
    case s.sentiment 
    when 'positive' then 2 
    when 'neutral' then 1 
    when 'negative' then 0 
    end as sentiment  
    FROM tweets_clean t LEFT OUTER JOIN tweets_sentiment s on t.id = s.id;
```



