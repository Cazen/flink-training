---
layout: page
title: Taxi Data Streams
permalink: /exercises/taxiData.html
---

The [New York City Taxi & Limousine Commission](http://www.nyc.gov/html/tlc/html/home/home.shtml) provides a public [data set](https://uofi.app.box.com/NYCtaxidata) about taxi rides in New York City from 2009 to 2015. We use a modified subset of this data to generate streams of taxi ride events.

### 1. Download the taxi data files

Download the taxi data files by running the following commands

~~~~
wget http://training.data-artisans.com/trainingData/nycTaxiRides.gz
wget http://training.data-artisans.com/trainingData/nycTaxiFares.gz
~~~~

It's not strictly necessary to use wget, but however you get the data, **do not decompress or rename the `.gz` files**.

### 2. Schema of Taxi Ride Events

Our taxi data set contains information about individual taxi rides in New York City.
Each ride is represented by two events, a trip start and an trip end event.
Each event consists of eleven fields:

~~~
rideId         : Long      // a unique id for each ride
taxiId         : Long      // a unique id for each taxi
driverId       : Long      // a unique id for each driver
isStart        : Boolean   // TRUE for ride start events, FALSE for ride end events
startTime      : DateTime  // the start time of a ride
endTime        : DateTime  // the end time of a ride,
                           //   "1970-01-01 00:00:00" for start events
startLon       : Float     // the longitude of the ride start location
startLat       : Float     // the latitude of the ride start location
endLon         : Float     // the longitude of the ride end location
endLat         : Float     // the latitude of the ride end location
passengerCnt   : Short     // number of passengers on the ride
~~~

**Note:** The data set contains records with invalid or missing coordinate information (longitude and latitude are `0.0`).

There is also a related data set containing taxi ride fare data, with these fields:

~~~
rideId         : Long      // a unique id for each ride
taxiId         : Long      // a unique id for each taxi
driverId       : Long      // a unique id for each driver
startTime      : DateTime  // the start time of a ride
paymentType    : String    // CSH or CRD
tip            : Float     // tip for this ride
tolls          : Float     // tolls for this ride
totalFare      : Float     // total fare collected
~~~

### 3. Generate a Taxi Ride Data Stream in a Flink program

We provide a Flink source function that reads a `.gz` file with taxi ride records and emits a stream of `TaxiRide` events. The source operates in [event-time]({{ site.docs }}/dev/event_time.html).

In order to generate the stream as realistically as possible, events are emitted proportional to their timestamp. Two events that occurred ten minutes after each other in reality are also served ten minutes after each other. A speed-up factor can be specified to "fast-forward" the stream, i.e., given a speed-up factor of 60, events that happened within one minute are served in one second. Moreover, one can specify a maximum serving delay which causes each event to be randomly delayed within the specified bound. This yields an out-of-order stream as is common in many real-world applications.

For these exercises, a speed-up factor of 600 or more (i.e., 10 minutes of event time for every second of processing), and a maximum delay of 60 (seconds) will work well.

All exercises should be implemented using event-time characteristics. Event-time decouples the program semantics from serving speed and guarantees consistent results even in case of historic data or data which is delivered out-of-order.

**Note:** You have to add the `flink-training-exercises` dependency to your Maven `pom.xml` file as described in the [setup instructions]({{ site.baseurl }}/devEnvSetup.html) because the `TaxiRide` class and the generator (`TaxiRideSource`) are contained in the `flink-training-exercises` dependency.

#### Java

{% highlight java %}
// get an ExecutionEnvironment
StreamExecutionEnvironment env =
  StreamExecutionEnvironment.getExecutionEnvironment();
// configure event-time processing
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

// get the taxi ride data stream
DataStream<TaxiRide> rides = env.addSource(
  new TaxiRideSource("/path/to/nycTaxiRides.gz", maxDelay, servingSpeed));
{% endhighlight %}

#### Scala

{% highlight scala %}
// get an ExecutionEnvironment
val env = StreamExecutionEnvironment.getExecutionEnvironment
// configure event-time processing
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

// get the taxi ride data stream
val rides = env.addSource(
  new TaxiRideSource("/path/to/nycTaxiRides.gz", maxDelay, servingSpeed))
{% endhighlight %}

There is also a `TaxiFareSource` that works in an analogous fashion, using the nycTaxiFares.gz file. This source creates a stream of `TaxiFare` events.

#### Java

{% highlight java %}
// get the taxi fare data stream
DataStream<TaxiFare> fares = env.addSource(
  new TaxiFareSource("/path/to/nycTaxiFares.gz", maxDelay, servingSpeed));
{% endhighlight %}

#### Scala

{% highlight scala %}
// get the taxi fare data stream
val fares = env.addSource(
  new TaxiFareSource("/path/to/nycTaxiFares.gz", maxDelay, servingSpeed))
{% endhighlight %}

Note that some of the exercises expect you to use `CheckpointedTaxiRideSource` and/or `CheckpointedTaxiFareSource` instead. Unlike `TaxiRideSource` and `TaxiFareSource`, these variants are able to checkpoint their state.
