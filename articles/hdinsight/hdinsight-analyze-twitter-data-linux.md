---
title: Analyze Twitter data with Apache Hive on HDInsight | Microsoft Docs
description: Learn how to use Python to store Tweets that contain specific keywords, then use Hive and Hadoop on HDInsight to transform the raw TWitter data into a searchable Hive table.
services: hdinsight
documentationcenter: ''
author: Blackmist
manager: jhubbard
editor: cgronlun
tags: azure-portal

ms.assetid: e1e249ed-5f57-40d6-b3bc-a1b4d9a871d3
ms.service: hdinsight
ms.workload: big-data
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 02/17/2017
ms.author: larryfr

ms.custom: H1Hack27Feb2017
---
# Analyze Twitter data using Hive on Linux-based HDInsight

Learn how to use Apache Hive on an HDInsight cluster to process the Twitter data. The result is a list of Twitter users who sent the most tweets that contain a certain word.

> [!IMPORTANT]
> The steps in this document were tested on a Linux-based HDInsight cluster.
>
> Linux is the only operating system used on HDInsight version 3.4 or greater. For more information, see [HDInsight Deprecation on Windows](hdinsight-component-versioning.md#hdi-version-32-and-33-nearing-deprecation-date).

## Prerequisites

* A **Linux-based Azure HDInsight cluster**. For information on creating a cluster, see [Get Started with Linux-based HDInsight](hdinsight-hadoop-linux-tutorial-get-started.md) for steps on creating a cluster.
* An **SSH client**. For more information on using SSH with Linux-based HDInsight, see the following articles:
  
  * [Use SSH with Linux-based Hadoop on HDInsight from Linux, Unix, or OS X](hdinsight-hadoop-linux-use-ssh-unix.md)
  * [Use SSH with Linux-based Hadoop on HDInsight from Windows](hdinsight-hadoop-linux-use-ssh-windows.md)
* **Python** and [pip](https://pypi.python.org/pypi/pip)

## Get a Twitter feed

Twitter allows you to retrieve the [data for each tweet](https://dev.twitter.com/docs/platform-objects/tweets) as a JavaScript Object Notation (JSON) document through a REST API. [OAuth](http://oauth.net) is required for authentication to the API.

### Create a Twitter application

1. From a web browser, sign in to [https://apps.twitter.com/](https://apps.twitter.com/). Click the **Sign-up now** link if you don't have a Twitter account.

2. Click **Create New App**.

3. Enter **Name**, **Description**, **Website**. You can make up a URL for the **Website** field. The following table shows some sample values to use:
   
   | Field | Value |
   |:--- |:--- |
   | Name |MyHDInsightApp |
   | Description |MyHDInsightApp |
   | Website |http://www.myhdinsightapp.com |

4. Check **Yes, I agree**, and then click **Create your Twitter application**.

5. Click the **Permissions** tab. The default permission is **Read only**.

6. Click the **Keys and Access Tokens** tab.

7. Click **Create my access token**.

8. Click **Test OAuth** in the upper-right corner of the page.

9. Write down **consumer key**, **Consumer secret**, **Access token**, and **Access token secret**.

> [!NOTE]
> When you use the curl command in Windows, use double quotes instead of single quotes for the option values.


### Download tweets

The following Python code downloads 10,000 tweets from Twitter and save them to a file named **tweets.txt**.

> [!NOTE]
> The following steps are performed on the HDInsight cluster, since Python is already installed.
> 
> 

1. Connect to the HDInsight cluster using SSH:
   
        ssh USERNAME@CLUSTERNAME-ssh.azurehdinsight.net
   
    If you used a password to secure your SSH user account, you are prompted to enter it. If you used a public key, you may have to use the `-i` parameter to specify the matching private key. For example, `ssh -i ~/.ssh/id_rsa USERNAME@CLUSTERNAME-ssh.azurehdinsight.net`.
   
    For more information on using SSH with Linux-based HDInsight, see the following articles:
   
   * [Use SSH with Linux-based Hadoop on HDInsight from Linux, Unix, or OS X](hdinsight-hadoop-linux-use-ssh-unix.md)
   * [Use SSH with Linux-based Hadoop on HDInsight from Windows](hdinsight-hadoop-linux-use-ssh-windows.md)

2. By default, the **pip** utility is not installed on the HDInsight head node. Use the following to install, and then update this utility:
   
   ```bash
   sudo apt-get install python-pip
   sudo pip install --upgrade pip
   ```

3. Use the following commands to install [Tweepy](http://www.tweepy.org/) and [Progressbar](https://pypi.python.org/pypi/progressbar/2.2):
   
   ```bash
   sudo apt-get install python-dev libffi-dev libssl-dev
   sudo apt-get remove python-openssl
   sudo pip install tweepy progressbar pyOpenSSL requests[security]
   ```

   > [!NOTE]
   > The bits about removing python-openssl, installing python-dev, libffi-dev, libssl-dev, pyOpenSSL, and requests[security] is to avoid an InsecurePlatform warning when connecting to Twitter via SSL from Python.
   > 
   > Tweepy v3.2.0 is used to avoid [an error](https://github.com/tweepy/tweepy/issues/576) that can occur when processing tweets.

4. Use the following command to create a file named **gettweets.py**:
   
   ```bash
   nano gettweets.py
   ```

5. Use the following text as the contents of the **gettweets.py** file. Replace the placeholder information for **consumer\_secret**, **consumer\_key**, **access/\_token**, and **access\_token\_secret** with the information from your Twitter application.
   
   ```python
   #!/usr/bin/python

   from tweepy import Stream, OAuthHandler
   from tweepy.streaming import StreamListener
   from progressbar import ProgressBar, Percentage, Bar
   import json
   import sys

   #Twitter app information
   consumer_secret='Your consumer secret'
   consumer_key='Your consumer key'
   access_token='Your access token'
   access_token_secret='Your access token secret'

   #The number of tweets we want to get
   max_tweets=10000

   #Create the listener class that receives and saves tweets
   class listener(StreamListener):
       #On init, set the counter to zero and create a progress bar
       def __init__(self, api=None):
           self.num_tweets = 0
           self.pbar = ProgressBar(widgets=[Percentage(), Bar()], maxval=max_tweets).start()

       #When data is received, do this
       def on_data(self, data):
           #Append the tweet to the 'tweets.txt' file
           with open('tweets.txt', 'a') as tweet_file:
               tweet_file.write(data)
               #Increment the number of tweets
               self.num_tweets += 1
               #Check to see if we have hit max_tweets and exit if so
               if self.num_tweets >= max_tweets:
                   self.pbar.finish()
                   sys.exit(0)
               else:
                   #increment the progress bar
                   self.pbar.update(self.num_tweets)
           return True

       #Handle any errors that may occur
       def on_error(self, status):
           print status

   #Get the OAuth token
   auth = OAuthHandler(consumer_key, consumer_secret)
   auth.set_access_token(access_token, access_token_secret)
   #Use the listener class for stream processing
   twitterStream = Stream(auth, listener())
   #Filter for these topics
   twitterStream.filter(track=["azure","cloud","hdinsight"])
   ```

6. Use **Ctrl + X**, then **Y** to save the file.

7. Use the following command to run the file and download tweets:
   
    ```bash
    python gettweets.py
    ```
   
    A progress indicator should appear, and count up to 100% as the tweets are downloaded and saved to file.
   
   > [!NOTE]
   > If it is taking a long time for the progress bar to advance, you should change the filter to track trending topics. When there are many tweets about the topic in your filter, you can quickly get the 10000 tweets needed.

### Upload the data

To upload the data to HDInsight storage, use the following commands:

   ```bash
   hdfs dfs -mkdir -p /tutorials/twitter/data
   hdfs dfs -put tweets.txt /tutorials/twitter/data/tweets.txt
```

These commands store the data in a location that all nodes in the cluster can access.

## Run the HiveQL job

1. Use the following command to create a file containing HiveQL statements:
   
   ```bash
   nano twitter.hql
   ```

    Use the following text as the contents of the file:
   
   ```hiveql
   set hive.exec.dynamic.partition = true;
   set hive.exec.dynamic.partition.mode = nonstrict;
   -- Drop table, if it exists
   DROP TABLE tweets_raw;
   -- Create it, pointing toward the tweets logged from Twitter
   CREATE EXTERNAL TABLE tweets_raw (
       json_response STRING
   )
   STORED AS TEXTFILE LOCATION '/tutorials/twitter/data';
   -- Drop and recreate the destination table
   DROP TABLE tweets;
   CREATE TABLE tweets
   (
       id BIGINT,
       created_at STRING,
       created_at_date STRING,
       created_at_year STRING,
       created_at_month STRING,
       created_at_day STRING,
       created_at_time STRING,
       in_reply_to_user_id_str STRING,
       text STRING,
       contributors STRING,
       retweeted STRING,
       truncated STRING,
       coordinates STRING,
       source STRING,
       retweet_count INT,
       url STRING,
       hashtags array<STRING>,
       user_mentions array<STRING>,
       first_hashtag STRING,
       first_user_mention STRING,
       screen_name STRING,
       name STRING,
       followers_count INT,
       listed_count INT,
       friends_count INT,
       lang STRING,
       user_location STRING,
       time_zone STRING,
       profile_image_url STRING,
       json_response STRING
   );
   -- Select tweets from the imported data, parse the JSON,
   -- and insert into the tweets table
   FROM tweets_raw
   INSERT OVERWRITE TABLE tweets
   SELECT
       cast(get_json_object(json_response, '$.id_str') as BIGINT),
       get_json_object(json_response, '$.created_at'),
       concat(substr (get_json_object(json_response, '$.created_at'),1,10),' ',
       substr (get_json_object(json_response, '$.created_at'),27,4)),
       substr (get_json_object(json_response, '$.created_at'),27,4),
       case substr (get_json_object(json_response,    '$.created_at'),5,3)
           when "Jan" then "01"
           when "Feb" then "02"
           when "Mar" then "03"
           when "Apr" then "04"
           when "May" then "05"
           when "Jun" then "06"
           when "Jul" then "07"
           when "Aug" then "08"
           when "Sep" then "09"
           when "Oct" then "10"
           when "Nov" then "11"
           when "Dec" then "12" end,
       substr (get_json_object(json_response, '$.created_at'),9,2),
       substr (get_json_object(json_response, '$.created_at'),12,8),
       get_json_object(json_response, '$.in_reply_to_user_id_str'),
       get_json_object(json_response, '$.text'),
       get_json_object(json_response, '$.contributors'),
       get_json_object(json_response, '$.retweeted'),
       get_json_object(json_response, '$.truncated'),
       get_json_object(json_response, '$.coordinates'),
       get_json_object(json_response, '$.source'),
       cast (get_json_object(json_response, '$.retweet_count') as INT),
       get_json_object(json_response, '$.entities.display_url'),
       array(
           trim(lower(get_json_object(json_response, '$.entities.hashtags[0].text'))),
           trim(lower(get_json_object(json_response, '$.entities.hashtags[1].text'))),
           trim(lower(get_json_object(json_response, '$.entities.hashtags[2].text'))),
           trim(lower(get_json_object(json_response, '$.entities.hashtags[3].text'))),
           trim(lower(get_json_object(json_response, '$.entities.hashtags[4].text')))),
       array(
           trim(lower(get_json_object(json_response, '$.entities.user_mentions[0].screen_name'))),
           trim(lower(get_json_object(json_response, '$.entities.user_mentions[1].screen_name'))),
           trim(lower(get_json_object(json_response, '$.entities.user_mentions[2].screen_name'))),
           trim(lower(get_json_object(json_response, '$.entities.user_mentions[3].screen_name'))),
           trim(lower(get_json_object(json_response, '$.entities.user_mentions[4].screen_name')))),
       trim(lower(get_json_object(json_response, '$.entities.hashtags[0].text'))),
       trim(lower(get_json_object(json_response, '$.entities.user_mentions[0].screen_name'))),
       get_json_object(json_response, '$.user.screen_name'),
       get_json_object(json_response, '$.user.name'),
       cast (get_json_object(json_response, '$.user.followers_count') as INT),
       cast (get_json_object(json_response, '$.user.listed_count') as INT),
       cast (get_json_object(json_response, '$.user.friends_count') as INT),
       get_json_object(json_response, '$.user.lang'),
       get_json_object(json_response, '$.user.location'),
       get_json_object(json_response, '$.user.time_zone'),
       get_json_object(json_response, '$.user.profile_image_url'),
       json_response
   WHERE (length(json_response) > 500);
   ```

2. Press **Ctrl + X**, then press **Y** to save the file.
3. Use the following command to run the HiveQL contained in the file:
   
   ```bash
   beeline -u 'jdbc:hive2://localhost:10001/;transportMode=http' -n admin -i twitter.hql
   ```

    This command runs the the **twitter.hql** file. Once the query completes, you see a `jdbc:hive2//localhost:10001/>` prompt.

4. From the beeline prompt, use the following to verify that you can select data from the **tweets** table created by the HiveQL in the **twitter.hql** file:
   
   ```hiveql
   SELECT name, screen_name, count(1) as cc
       FROM tweets
       WHERE text like "%Azure%"
       GROUP BY name,screen_name
       ORDER BY cc DESC LIMIT 10;
   ```

    This query returns a maximum of 10 tweets that contain the word **Azure** in the message text.

## Next steps

You have learned how to transform an unstructured JSON dataset into a structured Hive table. To learn more about Hive on HDInsight, see the following documents:

* [Get started with HDInsight](hdinsight-hadoop-linux-tutorial-get-started.md)
* [Analyze flight delay data using HDInsight](hdinsight-analyze-flight-delay-data-linux.md)

[curl]: http://curl.haxx.se
[curl-download]: http://curl.haxx.se/download.html

[apache-hive-tutorial]: https://cwiki.apache.org/confluence/display/Hive/Tutorial

[twitter-streaming-api]: https://dev.twitter.com/docs/streaming-apis
[twitter-statuses-filter]: https://dev.twitter.com/docs/api/1.1/post/statuses/filter
