# Setup

## Twitter developer account and app
* Create a twitter developer app and create keys. More here: https://apps.twitter.com/.

## Kafka server
* Create a kafka server on GCP. Take the Google kafka deployment from marketplace (https://console.cloud.google.com/marketplace/details/click-to-deploy-images/kafka). SCP into it using gcloud command.
```
gcloud beta compute --project <YOUR_PROJECT_ID> ssh --zone "us-west2-b" "kafka-1-vm
```

* Create stream_twitter.py and update the secrets from twitter app
```
from tweepy.streaming import StreamListener
from tweepy import OAuthHandler
from tweepy import Stream
from kafka import SimpleProducer, KafkaClient

access_token = <FILL THIS>
access_token_secret =  <FILL THIS>
consumer_key =  <FILL THIS>
consumer_secret =  <FILL THIS>

class StdOutListener(StreamListener):
    def on_data(self, data):
        producer.send_messages("apple", data.encode('utf-8'))
        print (data)
        return True
    def on_error(self, status):
        print (status)

kafka = KafkaClient("localhost:9092")
producer = SimpleProducer(kafka)
l = StdOutListener()
auth = OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
stream = Stream(auth, l)
stream.filter(track="apple")
```     

* Install python libraries
```
pip install kafka-python
pip install python-twitter
pip install tweepy
```
* Stream data feed from twitter
```
python stream_twitter.py
```

Credits: https://www.bmc.com/blogs/working-streaming-twitter-data-using-kafka/

## Greenplum cluster

* Create Greenplum cluster from marketplace: https://console.cloud.google.com/marketplace/details/pivotal-public/pivotal-greenplum-byol
```
sudo su - gpadmin
```

* Create load_twitter_data.yaml.  
```
DATABASE: twitter
USER: gpadmin
HOST: mdw
PORT: 6432
KAFKA:
   INPUT:
     SOURCE:
        BROKERS: 34.94.239.70:9092
        TOPIC: apple
     COLUMNS:
        - NAME: jdata
          TYPE: json
     FORMAT: json
     ERROR_LIMIT: 10
   OUTPUT:
     TABLE: tweets
     MAPPING: 
        - NAME: id
          EXPRESSION: (jdata->>'id')::bigint
        - NAME: created_at
          EXPRESSION: (jdata->>'created_at')::date
        - NAME: coordinates
          EXPRESSION: jdata#>'{coordinates}'
        - NAME: tweet_text
          EXPRESSION: jdata->>'text'
        - NAME: full_text
          EXPRESSION: ( jdata#>'{extended_tweet,full_text}' )::text
        - NAME: in_reply_to_status_id
          EXPRESSION: jdata->>'in_reply_to_status_id'
        - NAME: in_reply_to_screen_name
          EXPRESSION: jdata->>'in_reply_to_screen_name'
        - NAME: user_id
          EXPRESSION: ( jdata#>'{user,id}' )::text::bigint
        - NAME: user_name
          EXPRESSION: ( jdata#>'{user,name}' )::text
        - NAME: user_location
          EXPRESSION: ( jdata#>'{user,location}' )::text
        - NAME: hashtags
          EXPRESSION:  ( jdata#>'{entities,hashtags}' )
        - NAME: user_mentions
          EXPRESSION: ( jdata#>'{entities,user_mentions}' )
        - NAME: quote_count
          EXPRESSION: ( jdata->>'quote_count' )::int
        - NAME: retweet_count
          EXPRESSION: ( jdata->>'retweet_count' )::int
        - NAME: favorite_count
          EXPRESSION: ( jdata->>'favorite_count' )::int
        - NAME: favorited
          EXPRESSION: ( jdata->>'favorited' )::boolean
        - NAME: possibly_sensitive
          EXPRESSION: ( jdata->>'possibly_sensitive' )::boolean
        - NAME: lang
          EXPRESSION: (jdata->>'lang')::text
   COMMIT:
     MAX_ROW: 1000
     MINIMAL_INTERVAL: 5000
```     
* Check that the IP address is the kafka server above and make sure connectivity is ok. Firewall rule  may be needed for port 9092 to be open. I had to add this in GCP firewall rules
```
###### Targets

Target tags

kafka-1-deployment

###### Source filters

IP ranges

0.0.0.0/0

###### Protocols and ports

tcp:9092
```
* I needed to add an entry in /etc/hosts for GCP name to IP address
```
34.94.239.70  kafka-1-vm.c.data-gpdb-ud.internal
```

* Create twitter db and tweet_data table

```
createdb twitter
psql twitter

create extension gpss;

create table tweets(
  id bigint,
  created_at date,
  coordinates json,
  tweet_text text,
  full_text text,
  in_reply_to_status_id text,
  in_reply_to_user_id text,
  in_reply_to_screen_name text,
  user_id bigint,
  user_name text,
  user_location text,
  hashtags json ,
  user_mentions json ,
  quote_count int,
  reply_count int,
  retweet_count int,
  favorite_count int,
  favorited boolean,
  retweeted boolean,
  possibly_sensitive boolean,
  lang text
);
```
* Start gpss
```
gpss&
```
* Run gpsscli

```
gpsscli load load_twitter_data.yaml

```
* Select data from tweets and see the summary statistics

```
\x
select * tweets limit 5;

SELECT * FROM madlib.summary( 'tweets',
                              'tweets_summary',
                              'tweet_text,user_id',
                              'user_id',
                              TRUE,
                              TRUE,
                              NULL,
                              5,
                              FALSE
                            );

SELECT * FROM tweets_summary;

```


# Troubleshooting

These are some queries that help look at the imported data

```
# Does the text contain the word apple?
select count(*)
from tweet_data where (jdata->>'text')::text like '%apple%';
```

```
# Connections between in_reply_to_user_id and userid
select  count(*) from tweet_data where (jdata#>'{user,id}')::text in (select distinct jdata->>'in_reply_to_user_id' from tweet_data);
```

```
# Connections between in_reply_to_status_id and status_id
select  count(*) from tweet_data where (jdata->>'id')::text in (select distinct jdata->>'in_reply_to_status_id' from tweet_data);
```


```
# Tweets with coordinates
select  count(*) from tweet_data where (jdata#>'{coordinates}')::text != 'null';
```

```
# Distinct Tweets 
select  count (distinct jdata->>'id') from tweet_data ;
```

