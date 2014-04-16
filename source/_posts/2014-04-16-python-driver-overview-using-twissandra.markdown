---
layout: post
title: "Python driver overview using Twissandra"
date: 2014-04-16 11:45:20 +0300
comments: true
categories: 
---
<p>Twissandra, a Twitter clone using Cassandra for storage, has had a makeover to use the new <a href=""https://github.com/datastax/python-driver" target="_blank">python driver</a>. This allowed the clone to make the switch from the thrift API to using CQL3 over the <a href="http://www.datastax.com/dev/blog/binary-protocol" target="_blank">native protocol</a>. Let's go through some examples of using the python driver, taken from the updated Twissandra code.</p>
<h2>Twissandra Datamodel Overview</h2>
Twissandra is composed of six tables that store users, tweets, tweet order (of the user and their timeline) and who users follow (and are followed by). Since we can't use joins in Cassandra, tables are partially denormalised to allow for necessary flexibility, meaning there are more writes to make reads more performant.<br/><br/>
<img src="http://www.datastax.com/wp-content/uploads/2014/04/Twissandra-ER.png" alt="Twissandra ER Diagram" />
<br/>The users table simply stores usernames and passwords:<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0"> CREATE TABLE users (
&nbsp;&nbsp;&nbsp;&nbsp;username text PRIMARY KEY,
&nbsp;&nbsp;&nbsp;&nbsp;password text
)</pre><br/>
<strong>Tracking latest tweets</strong><br/>
Tweets are stored in a simple table where the primary key is a UUID column, ensuring the tweet's uniqueness. We don't track when the tweet was added in this table as that's handled by the user's timeline (see the userline and timeline table creation below).<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">CREATE TABLE tweets (
&nbsp;&nbsp;&nbsp;&nbsp;tweet_id uuid PRIMARY KEY,
&nbsp;&nbsp;&nbsp;&nbsp;username text,
&nbsp;&nbsp;&nbsp;&nbsp;body text
)</pre><br/>
TimeUUIDs are used for tracking the time of the tweet, to ensure uniqueness in the primary key, as they are composed of a random component and a timestamp. This allows us to retrieve unique tweets by time and also allows for tracking when the tweet was added. Cassandra sorts the timeline and userline based on the clustering key time. Since the aim is to retrieve the latest tweets <strong>WITH CLUSTERING ORDER BY (time DESC)</strong> is added to the table creation statements to invert the sorting.<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">CREATE TABLE userline (
&nbsp;&nbsp;&nbsp;&nbsp;username text,
&nbsp;&nbsp;&nbsp;&nbsp;time timeuuid,
&nbsp;&nbsp;&nbsp;&nbsp;tweet_id uuid,
&nbsp;&nbsp;&nbsp;&nbsp;PRIMARY KEY (username, time)
) WITH CLUSTERING ORDER BY (time DESC)<br/>
CREATE TABLE timeline (
&nbsp;&nbsp;&nbsp;&nbsp;username text,
&nbsp;&nbsp;&nbsp;&nbsp;time timeuuid,
&nbsp;&nbsp;&nbsp;&nbsp;tweet_id uuid,
&nbsp;&nbsp;&nbsp;&nbsp;PRIMARY KEY (username, time)
) WITH CLUSTERING ORDER BY (time DESC)</pre><br/>
Because the username is the partition key, we can easily select the most recent tweets for a specific user. The LIMIT clause can then be added to enforce a limit on how many tweets are retrieved:<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">SELECT * FROM timeline WHERE username=? LIMIT 100</pre><br/>
An important note. The data-model presented here is only partially denormalized. Denormalizing the tweets table completely into the timeline and userline tables would improve query time, by letting us directly query the tweets from them, instead of requiring a second set of SELECT's to retrieve the content of the tweets<br/>
<br/><strong>Tracking Followers</strong><br/>The followers table allows for retrieval of the users that are following you. The friends table allows for retrieval of the users that you follow. The primary key for both tables is a composite key. This is important because the first component of the composite key, the partition key, decides how to split data around the cluster. One set of replicas will store all the data for a specific user. The second component is the clustering key which is used to store data in a particular order on disk. Although the ordering itself isn't important for either table, the clustering key means all rows for a particular user will be stored contiguously on disk.  This optimises reading a user's friends or followers by allowing for a sequential disk read.<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">CREATE TABLE friends (
&nbsp;&nbsp;&nbsp;&nbsp;username text,
&nbsp;&nbsp;&nbsp;&nbsp;friend text,
&nbsp;&nbsp;&nbsp;&nbsp;since timestamp,
&nbsp;&nbsp;&nbsp;&nbsp;PRIMARY KEY (username, friend)
)<br/>
CREATE TABLE followers (
&nbsp;&nbsp;&nbsp;&nbsp;username text,
&nbsp;&nbsp;&nbsp;&nbsp;follower text,
&nbsp;&nbsp;&nbsp;&nbsp;since timestamp,
&nbsp;&nbsp;&nbsp;&nbsp;PRIMARY KEY (username, follower)
)</pre><br/>
To retrieve all the followers or friends for a specific user, the username is added to the WHERE clause just like in SQL. Something worth noting is that we can use the username in the WHERE clause because <a href="http://planetcassandra.org/blog/post/flite-breaking-down-the-cql-where-clause/" target="_blank">it's part of the primary key</a>.<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">SELECT follower FROM followers WHERE username=?
SELECT friend FROM friends WHERE username=?</pre><br/>
<h2>Setting up a connection</h2>
To connect to Cassandra we first import the driver's Cluster class. The next step is to create a cluster and a session. We then supply the list of IPs for nodes in the cluster and tell the session what keyspace to connect to. Note that sessions automatically manage a pool of connections so they should be long-lived and re-used for multiple requests.<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">from cassandra.cluster import Cluster<br/>
# initialise cluster for 127.0.0.1
CLUSTER = Cluster(['127.0.0.1'])
# initialise session with the 'twissandra' keyspace
session = CLUSTER.connect('twissandra')</pre><br/>
<h2>Some CRUD</h2>
The various things that twitter can do, whether it's inserting a tweet, retrieving your followers, updating your password or unfollowing someone, are examples of create / read / update and delete operations that can be carried out on Cassandra.<br/>
<br/><strong>Tweeting - Create</strong><br/>Adding tweets is done via Twissandra's <a href="https://github.com/twissandra/twissandra/blob/master/cass.py#L224" target="_blank">save_tweet</a> function where four kinds of queries are carried out:
<ol>
    <li>Insert the tweet</li>
    <li>Update the current user’s userline with the tweet_id </li>
    <li>Update the public userline with the tweet_id </li>
    <li>Update the timelines of all of the user’s followers with the tweet_id</li>
</ol>
<img src="http://www.datastax.com/wp-content/uploads/2014/04/Twissandra-INSERT.png" alt="Inserting a message into Twissandra" /><br/>
Inserting the tweet message is as simple as supplying the username, the message, and generating a UUID. Note, if we didn't need to save the UUID for use in later inserts, it could have been created using the <code>uuid()</code> function <a href="https://issues.apache.org/jira/browse/CASSANDRA-6473" target="_blank">available in Cassandra 2.0</a>. For a full list of CQL3 functions take a look at the <a href="http://www.datastax.com/documentation/cql/3.1/cql/cql_reference/cql_function_r.html">DataStax docs</a>.<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">if tweets_query is None:
&nbsp;&nbsp;&nbsp;&nbsp;tweets_query = session.prepare("""
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;INSERT INTO tweets (tweet_id, username, body)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;VALUES (?, ?, ?)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;""")<br/>
if timestamp is None:
&nbsp;&nbsp;&nbsp;&nbsp;now = uuid1()
else:
&nbsp;&nbsp;&nbsp;&nbsp;now = _timestamp_to_uuid(timestamp)
<br/>
# Insert the tweet
session.execute(tweets_query, (tweet_id, username, tweet,))</pre><br/>
Adding to the user's and public userlines requires a username, the tweet's ID and a time uuid: <br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">if userline_query is None:
&nbsp;&nbsp;&nbsp;&nbsp;userline_query = session.prepare("""
&nbsp;&nbsp;&nbsp;&nbsp;INSERT INTO userline (username, time, tweet_id)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;VALUES (?, ?, ?)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;""")<br/>
# Insert tweet into the user's timeline
session.execute(userline_query, (username, now, tweet_id,))
# Insert tweet into the public timeline
session.execute(userline_query, (PUBLIC_USERLINE_KEY, now, tweet_id,))</pre><br/>
Finally to complete the tweeting process, the tweet has to be inserted into each one of your  follower's timelines. This requires the username of the follower, the tweet's creation time in the form of a Time UUID and the tweet's ID in the form of a UUID.<br/><br/> 
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">
if timeline_query is None:
&nbsp;&nbsp;&nbsp;&nbsp;timeline_query = session.prepare("""
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;INSERT INTO timeline (username, time, tweet_id)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;VALUES (?, ?, ?)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;""")<br/>
futures = []
follower_usernames = [username] + get_follower_usernames(username)
for follower_username in follower_usernames:
&nbsp;&nbsp;&nbsp;&nbsp;futures.append(session.execute_async(
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;timeline_query, (follower_username, now, tweet_id)))
<br/>for future in futures:
&nbsp;&nbsp;&nbsp;&nbsp;future.result()</pre><br/>
<strong>Retrieving Tweets - Read</strong><br/>
Retrieving tweets is done using one of two functions in Twissandra. The <code>get_timeline</code> and <code>get_userline</code> functions are both calls to <a href="https://github.com/twissandra/twissandra/blob/master/cass.py#L38" target="_blank">_get_line</a>. Retrieving either all of our tweets or all of someone else's tweets is done via <code>_get_line</code>. To carry out the querying we require a username, a tweet starting time and the number of tweets to fetch. Since we don't want to fetch the entire feed, first the range of tweets that we want to retrieve is selected.
<img src="http://www.datastax.com/wp-content/uploads/2014/04/Twissandra-PublicTimeline.png" alt="Retrieving messages from Twissandra" /><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">if not start:
&nbsp;&nbsp;&nbsp;&nbsp;time_clause = ''
&nbsp;&nbsp;&nbsp;&nbsp;params = (username, limit)
else:
&nbsp;&nbsp;&nbsp;&nbsp;time_clause = 'AND time < %s'
&nbsp;&nbsp;&nbsp;&nbsp;params = (username, UUID(start), limit)</pre><br/>
If we need to start our page further back than the latest tweets, the less-than predicate, <code>time < %s</code>, can be used to retrieve tweets further back in the timeline. <br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">query = "SELECT time, tweet_id FROM {table} WHERE username=%s {time_clause} LIMIT %s"
    query = query.format(table=table, time_clause=time_clause)
    results = session.execute(query, params)</pre><br/>
Again, because we want to page through the timeline rather than retrieving all of it in a single query, we want to check if we reached the end of the timeline, and if not to store a marker to tell us where to start the page during the next query.<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0"># If we didn't get to the end, return a starting point for the next page
if len(results) == limit:
# Find the oldest ID
oldest_timeuuid = min(row.time for row in results)<br/>
# Present the string version of the oldest_timeuuid for the UI
next_timeuuid = oldest_timeuuid.urn[len('urn:uuid:'):]
else:
next_timeuuid = None</pre><br/>
Once the array of tweet IDs is retrieved, they are used to fetch the actual tweets. <br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">global get_tweets_query
&nbsp;&nbsp;&nbsp;&nbsp;if get_tweets_query is None:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;get_tweets_query = session.prepare("""
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SELECT * FROM tweets WHERE tweet_id=?
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;""")
futures = []
for row in results:
&nbsp;&nbsp;&nbsp;&nbsp;futures.append(session.execute_async(
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;get_tweets_query, (row.tweet_id,)))</pre><br/>
Queries are sometimes executed using <code>session.execute</code> and other times <code>session.execute_async</code> is used instead. The difference between the two is that <strong>execute</strong> waits for a response before returning whilst <strong>execute_async</strong> returns a "future" so it can send multiple messages concurrently, without waiting for responses, therefore there is no guarantee on the order of the responses. The returned <code>ResponseFuture</code> can be used to verify the query's success for both serial and concurrent queries. On failure an exception would be raised.<br/>
<br/><strong>Changing Password - Update</strong><br/>Updates and inserts have mostly identical behavior with Cassandra.  They both blindly overwrite existing (or non-existing) data. Twissandra doesn't use UPDATE statements but for completeness here is a theoretical example of updating a password:<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">
update_password_query = session.prepare("UPDATE users SET password = ? WHERE username = ?")
session.execute(update_password_query, (password, username)))</pre><br/>
<strong>Unfollowing - Delete</strong><br/>
Removing a user from your feed requires two queries since in CQL3 there are no foreign keys to enforce relationships between the friends and followers table. The first query removes the user from your feed while the second tells them you are no longer following them. Prior tweets from this user won't however be deleted from your timeline. <br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">def remove_friends(from_username, to_usernames):
&nbsp;&nbsp;&nbsp;&nbsp;"""
&nbsp;&nbsp;&nbsp;&nbsp;Removes a friendship relationship from one user to some others.
&nbsp;&nbsp;&nbsp;&nbsp;"""
&nbsp;&nbsp;&nbsp;&nbsp;global remove_friends_query
&nbsp;&nbsp;&nbsp;&nbsp;global remove_followers_query<br/>
&nbsp;&nbsp;&nbsp;&nbsp;if remove_friends_query is None:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remove_friends_query = session.prepare("""
&nbsp;&nbsp;&nbsp;&nbsp;DELETE FROM friends WHERE username=? AND friend=?
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;""")
&nbsp;&nbsp;&nbsp;&nbsp;if remove_followers_query is None:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remove_followers_query = session.prepare("""
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;DELETE FROM followers WHERE username=? AND follower=?
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;""")<br/>
&nbsp;&nbsp;&nbsp;&nbsp;futures = []
&nbsp;&nbsp;&nbsp;&nbsp;for to_user in to_usernames:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;futures.append(session.execute_async(
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remove_friends_query, (from_username, to_user,)))
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;futures.append(session.execute_async(
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remove_followers_query, (to_user, from_username,)))<br/>
&nbsp;&nbsp;&nbsp;&nbsp;for future in futures:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;future.result()</pre><br/>
<h2>Enhancing Twissandra With New Cassandra Features</h2>
Modelling in Cassandra frequently requires denormalization as there is no joining of tables. <a href="http://en.wikipedia.org/wiki/Denormalization" target="_blank">Denormalization</a> can be summed up as the process of adding redundant data to tables in order to optimise read performance. The frequent use-case in the relational model of having users with multiple email addresses is usually modelled by creating a user table and an email table where there is a one-to-many relationship. Cassandra’s alternative is to use CQL3 <a href="http://www.datastax.com/dev/blog/cql3_collections" target="_blank">collections</a> where a column can store a list, set or a map of fields. If Twissandra’s user table also required each user’s email address (see example below) and allowed for more than one, the set collection could be used to store them.<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">
session.execute("""
&nbsp;&nbsp;&nbsp;&nbsp;CREATE TABLE IF EXISTS users (
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;username text PRIMARY KEY,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;password text,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;emails set&lt;text&gt;
&nbsp;&nbsp;&nbsp;&nbsp;)
&nbsp;&nbsp;&nbsp;&nbsp;""")<br/>
add_user_query = session.prepare("""
&nbsp;&nbsp;&nbsp;&nbsp;INSERT INTO users (username, password,emails) VALUES (%s, %s, %s)
&nbsp;&nbsp;&nbsp;&nbsp;""")
session.execute(add_user_query, ("lyubent", "passw0rd", "{'me@email.com', 'me@example.com}"))</pre><br/>
<strong>Light Weight Transactions</strong><br/>
<a href="http://www.datastax.com/dev/blog/lightweight-transactions-in-cassandra-2-0" target="_blank">Lightweight transactions</a> weight transactions (LWT) are another piece of functionality that was added to satisfy commonly used patterns requiring strong consistency, like for example the need to ensure that a username is unique before allowing someone to register said username. LWT aren't available in version 0 of the python driver but are on their way in the new version 2.0. release. But here is an example of what inserting a username would look like using a LWT from cqlsh. We execute the INSERT as usual, but also append <code>IF NOT EXISTS</code><br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">INSERT INTO users (username, password, emails) VALUES<br/>("lyubent", "passw0rd", "{'lyubent@email.com'}") IF NOT EXISTS</pre><br/>
LWT can also be used to verify a row exists by appending <code>IF EXISTS</code> to the end of the query:<br/><br/>
<pre style="background-color: #eee; font-size: 0.7em;padding: 10px 4px; margin: 0">UPDATE users SET password = 'fas$Jx' WHERE username = 'lyubent' IF EXISTS</pre>