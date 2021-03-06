# UberSocialNet - applying the Lambda Architecture

### Summary

Recently, Nathan Marz introduced the [Lambda Architecture](http://manning.com/marz/)
for realising large-scale data processing applications. In this article, we will
walk step-by-step through how to build an application based on this architecture:
The *UberSocialNet* (USN), a little helper tool that allows us to keep track of
where we know people from. 

The USN lets us record if we happen to know someone from either the digital world, 
that is, social networks such as Twitter, Facebook, LinkedIn, G+, etc. or the real life. 
The goal is that USN can serve more than one billion users while providing low-latency 
access to the annotations we keep about where and how we know people.

### Contents

* [The Raw Input Data](https://github.com/mhausenblas/usn-app#the-raw-input-data)
* [Generation of the Layers](https://github.com/mhausenblas/usn-app#generation-of-the-layers)
* [Usage](https://github.com/mhausenblas/usn-app#usage)
* [Notes and Background](https://github.com/mhausenblas/usn-app#notes-and-background)


***


## The Raw Input Data

The [raw 
data](https://github.com/mhausenblas/usn-app/blob/master/data/usn-base-data.csv)
resides in the `data/` directory and has the following shape in the
base (CSV) format:

	timestamp,originator,action,network,target,context
	2012-03-12T22:54:13-07:00,Michael,ADD,I,Ora Hatfield,Some witty stuff here 
	2012-11-23T01:53:42-08:00,Michael,REMOVE,I,Marvin Garrison,Whatever note ...

with the following schema:

* `timestamp` is an ISO 8601 formatted datetime stamp that states when the 
action was performed (range: January 2012 till May 2013)
* `originator` is the name of the person who added or removed a person to or
from one of his or her networks
* `action` MUST be either `ADD` or `REMOVE`, self-explanatory. 
* `network` is a single character, MUST be one out of `{I, T, L, F, G}`, 
indicating the respective network where the action has been performed, with: 
 * `I` … in-real-life
 * `T` … Twitter
 * `L` … LinkedIn
 * `F` … Facebook
 * `G` … Google+
* `target` is the name of the person added or removed to or from the network
* `context` is a free-text comment, providing a hint why the person has been
added or where one has met the person in the first place

All fields are always present, that is, there are no optional fields. The test
data has been generated using [generatedata.com](http://www.generatedata.com/) 
in five runs totaling some 500 rows of raw data. 

[*Back to ToC*](https://github.com/mhausenblas/usn-app#contents)

## Generation of the Layers

First, I'm going to show you the three commands you have to run to generate
the two layers (batch and serving) of the USN app and then we will have a
closer look behind the scenes of each of the commands.

To build the datastore for USN, run the following commands:

*1.* To pre-process the raw data, in the `data` dir:

	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app/data
	$ ./usn-preprocess.sh < usn-raw-data.csv > usn-base-data.csv

*2.* To build the batch layer, with the Hive Thrift service running,
   in the `batch-layer` dir:

	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app/batch-layer
	$ ./batch-layer.sh INIT
	USN batch layer created.
	$ ./batch-layer.sh CHECK
	The USN batch layer seems OK.

*3.* To build the serving layer, with both the Hive and HBase Thrift service 
   running, in the main USN dir:
	
	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app
	$ python serving-layer.py localhost INIT
	...
	2013-06-07T06:02:53 Initialized USN tables in serving layer.

As you now have an idea what to do, we will have a closer look at each of the
steps and see what is going on exactly in the next subsections.

### Batch Layer

Make sure you have Hive 0.10.0 installed or access to a setup (cluster, cloud) 
where it is running.

The raw data is first 
[pre-processed
](https://github.com/mhausenblas/usn-app/blob/master/data/usn-preprocess.sh) and 
[loaded](https://github.com/mhausenblas/usn-app/blob/master/hive-cmds.txt) into 
Hive like so:

	hive> CREATE TABLE usn_base (
	 actiontime STRING,
	 originator STRING,
	 action STRING,
	 network STRING,
	 target STRING,
	 context STRING
	) ROW FORMAT DELIMITED FIELDS TERMINATED BY '|';

	hive> LOAD DATA LOCAL INPATH 
	'/Users/mhausenblas2/Documents/repos/usn-app/data/usn-base-data.csv'
	INTO TABLE usn_base;

	hive> CREATE TABLE usn_friends AS
	      SELECT originator AS username, network, target AS friend, context AS note
	      FROM usn_base
	      WHERE action = 'ADD'
	      ORDER BY username, network, username;

So to pre-process the raw data, change into the `data` directory and execute 
the following:

	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app/data
	$ ./usn-preprocess.sh < usn-raw-data.csv > usn-base-data.csv

Then, to build the batch layer from scratch, perform the following steps.

Make sure Hive service is running from the USN batch layer directory:

	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app/batch-layer
	$ hive --service hiveserver
	Starting Hive Thrift Server
	...

And finally, from within the `batch-layer` directory:

	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app/batch-layer
	$ ./batch-layer.sh INIT
	USN batch layer created.
	$ ./batch-layer.sh CHECK
	The USN batch layer seems OK.

Now the batch layer is generated and available in HDFS. Next we will build the
serving layer in HBase.

### Serving Layer

Make sure you have HBase 0.94.x installed or access to a setup (cluster, cloud) 
where it is running.

**Preparation** First you need to launch HBase and the HBase Thrift server. 

Go to the HBase home directory (`$HBASE_HOME`) and do the following:

	$ echo $HBASE_HOME
	/Users/mhausenblas2/bin/hbase-0.94.4
	$ cd /Users/mhausenblas2/bin/hbase-0.94.4
	$ ./bin/start-hbase.sh 
	
	starting master, logging to /Users/...

	$ ./bin/hbase thrift start -p 9191
	13/05/31 09:39:09 INFO util.VersionInfo: HBase 0.94.4
	...

Also, make sure that the Hive service is running. In case you've shut it
down after the batch layer generation, restart it. So, from the USN batch 
layer directory do:

	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app/batch-layer
	$ hive --service hiveserver
	Starting Hive Thrift Server
	...

**Init** First you need to initialize the USN table.

Change to the parent directory of the batch layer directory and
execute the following:

	$ pwd
	/Users/mhausenblas2/Documents/repos/usn-app
	$ python serving-layer.py localhost INIT
	...
	2013-06-07T06:02:53 Initialized USN tables in serving layer.

You can use the HBase shell to verify if the serving layer has been 
initialized correctly:

	$ ./bin/hbase shell
	hbase(main):001:0> describe 'usn_friends'
	DESCRIPTION                                                                                          ENABLED
	 {NAME => 'usn_friends', FAMILIES => [{NAME => 'a', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'N true
	 ONE', REPLICATION_SCOPE => '0', VERSIONS => '3', COMPRESSION => 'NONE', MIN_VERSIONS => '0', TTL =>
	  '-1', KEEP_DELETED_CELLS => 'false', BLOCKSIZE => '65536', IN_MEMORY => 'false', ENCODE_ON_DISK =>
	  'true', BLOCKCACHE => 'false'}]}
	1 row(s) in 0.2450 seconds
	
	hbase(main):002:0> count 'usn_friends'
	499 row(s) in 0.0540 seconds
	
A sample query might now look as follows:

	hbase(main):001:0> scan 'usn_friends', { COLUMNS => ['a'], FILTER => "ValueFilter(=,'substring:L')", STARTROW => 'Ted_2013-01'}
	ROW                                      COLUMN+CELL
	 Ted_2013-01-17                          column=a:comment, timestamp=1370630348723, value=urna et arcu imperdiet ullamcorper. Duis at lacus. Quisque purus
	 ...
	 Ted_2013-03-25                          column=a:network, timestamp=1370630348769, value=L
	8 row(s) in 0.0460 seconds

The above query translates into: give me all acquaintances of `Ted` in the
`LinkedIn` network, starting from January 2013 on. 

You can have a look at some more queries used in the demo user interface
on the [respective Wiki page](https://github.com/mhausenblas/usn-app/wiki/Serving-Layer).

Oh. And when you're done, don't forget to shut down HBase 
(again, from HBase home):

	$ ./bin/stop-hbase.sh

The Hive service can simply be stopped by hitting `CTRL+C`.

You're now done with generating the necessary layers for the USN app and can
start using it. I'll show you how in the next section.

[*Back to ToC*](https://github.com/mhausenblas/usn-app#contents)

## Usage

### Dependencies 

The following components and tools are assumed to be available:

* Hive 0.10.x
* [Hiver](https://github.com/tebeka/hiver) for Python interaction
* HBase 0.94.x
* [HappyBase](https://github.com/wbolster/happybase) for Python interaction

Then, the pre-processing steps as explained above (batch and serving layer
generation) must be done.

### CLI Front-end

After you've prepared and init the batch and serving layers as described above
you can launch the user interface, a simple CLI for now. Make sure that HBase
and the HBase Thrift service (details, see above) are running, then, in the
main USN directory do the following:

	$ ./usn-ui.sh
	This is USN v0.0
	
	u ... user listings, n ... network listings, l ... lookup, s ... search, h ... help, q ... quit
	Your selection: s
	The name of the person you want to search?
	Name >Kevin
	User 'Ellen' has 'Kevin Bowman' from Google+ in his network.
	*** Found 1 matches in total

	u ... user listings, n ... network listings, l ... lookup, s ... search, h ... help, q ... quit
	Your selection: l
	List acquintances of which user?
	One of: Ellen, John, Karen, Michael, Steve, Ted >Ellen
	From when?
	In the form YYYY-MM-DD, such as 2013-01-01 or only 2012 >2012-05-01
	(OPTIONAL) Until when?
	In the form YYYY-MM-DD, such as 2013-01-01 or only 2012 >2012-08-01
	(OPTIONAL) From which network?
	One of: I - in-real-life, T - Twitter, L - LinkedIn, F - Facebook, G - Google+ >
	Nicole Tyson from Twitter
	Vielka Barr from Google+
	Latifah Horton from Google+
	Dorothy Roy from real life
	Myles Greer from LinkedIn
	Cecilia Vance from real life
	*** Found 6 matches in total

The three main operations the USN front-end provides are:

* `u` ... user listing: lists *all* acquaintances of a user
* `n` ... network listing: lists acquaintances of a user in a network
* `l` ... lookup listing: lists acquaintances of a user in a network and allows 
to restrict the time range (from/to) of the acquaintanceship 
* `s` ... search: provides search for an acquaintance over all users 
(with partial match)


### License

All artifacts in this repository, including data and code are donated into
the Public Domain. The author would like to thank MapR Technologies for
sponsoring the work on the USN app.

[*Back to ToC*](https://github.com/mhausenblas/usn-app#contents)

## Notes and Background

In the following some notes and background information that is not necessary
to toy around with USN but might be of interest for the one or the other.

### Improvements

The demo app USN is intentionally kept very simple. Though fully functional,
the USN app has a number of serious limitations. You're welcome to fork it
and extend it. Please let me know if you do so.

* **Big Data?** The most obvious point is not the app itself but the data size.
Only laughable 500 rows? This isn't Big Data I hear you say. Rightly so. Now,
no one stops you generating 500 million rows or more and try it out. Certain
processes such as pre-processing and the generating the layers will take longer
however there are no architectural changes necessary, and this was the whole
point of the USN app.
* **Dynamic batch layer.** Currently, the batch layer is a sort of one-shot, 
while it should really run in a loop and append new data. This requires
partitioning of the ingested data and some checks. 
[Pail](https://github.com/nathanmarz/dfs-datastores), for example, allows you to
do the ingestion and partitioning in a very elegant way.
* **Automated import.** It would be cool to automate the import of data from the
different networks. 
For example, [Google Takeout](https://www.google.com/takeout/) allows to export
all data, including G+ Circles. Similar functionality is available from Twitter
and Facebook.
* **More views**. There is only one view in the serving layer. The USN app might
benefit from different views to enable different queries most efficiently.

### HBase Schema

The HBase table `usn_friends`, used in the serving layer to drive the USN app
has the following schema:

![USN's HBase schema](https://raw.github.com/mhausenblas/usn-app/master/doc/hbase-usn-schema.png "USN's HBase schema")

Some notes regarding the key and schema design:

* There is only one CF named `a`; keep CF names short, think of footprint.
* This CF holds all the data: the target person, from where known, comment.
* The row keys encode the user name and the date of the activity, that is
when a target has been added to the user's network. This allows to query by
user and by date range; actually this design allows several date ranges,
from years over months down to days.

### Wire-level Architecture

![USN's architecture](https://raw.github.com/mhausenblas/usn-app/master/doc/usn-architecture.png "USN's architecture")

Some notes regarding architecture:

* **Build-time vs. Run-time.** In one phase, the dataspace is built, then the
application can use the data in the front-end (CLI user interface).
* **Pre-processing.** In the pre-processing phase a script cleans the CSV data
and prepares it for ingestion into HDFS. 
* **Batch Layer.** A script uses Hive to ETL the CSV data into HDFS.
* **Serving Layer.** A script uses the data from HDFS and loads it into HBase.
Further, this layer provides query capabilities to drive the front-end. 
* **CLI user interface.** The front-end interacts with the end-user, providing
`listing`, `lookup` and `search` operations.
