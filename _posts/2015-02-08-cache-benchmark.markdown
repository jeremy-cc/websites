---
layout: post
title:  "Cache Benchmark"
date:   2015-02-08 08:13:20
categories: cache ruby benchmark
---

### Problem Statement
More interest is being generated in the apis we have so the time has come to put in some rate limiting to limit some of our more enthusiastic users. All our apis are hosted in Sinatra and there's a great piece of middleware called [Rack::Attack][rack_attack] that gives you fine-grained control over how you rate limit your users. 


{% highlight ruby %}
# Throttle attempts to a particular path. 2 POSTs to /login per second per IP
Rack::Attack.throttle "logins/ip", :limit => 2, :period => 1 do |req|
  req.post? && req.path == "/login" && req.ip
end
{% endhighlight %}

By default, the rate limit counter is stored using [ActiveSupport::Cache::MemoryStore][memory_store], which is threadsafe. This will work well when you have a single process but a new store will be created for each process you start. If you load balance requests across multiple processes or servers, this means that the intended rate limit will not be respected. Worse still is that a user could reach their limit on one process (or server) and have the next request accepted on another.

Fortunately, Rack::Attack has an easy way to switch out the default store for another one. 

{% highlight ruby %}
Rack::Attack.cache.store = MyCacheStore.new
{% endhighlight %}

Now it just became a question of which store to use. A common approach is to run up Memcache and point the cache (which Rack::Attack has support for in [Dalli][dalli]) in its direction. However, after discussing it amongst ourselves, the suggestion of using a MySQL Memory table came up. We thought this would be a good idea because, while we have caching in our internal systems, we did't have caching on our external-facing apps. We already had a MySQL table that kept hold of session information and we could just add a memory table to it like so:

{% highlight sql %}
DROP DATABASE IF EXISTS benchmark;
CREATE DATABASE benchmark;

CREATE TABLE benchmark.`cache` (
  `key` varchar(255) NOT NULL,
  `value` int(11) NOT NULL,
  PRIMARY KEY (`key`) USING HASH
) ENGINE=MEMORY DEFAULT CHARSET=utf8;
{% endhighlight %}

There is a caveat to this approach that should be mentioned. A cache, by definition, will evict old entries using an algorithm like LRU. A database is not normally designed to do this and MySql certainly doesn't seem to support this concept. MySql also has the limitation that when you use a Memory Store, space from a deleted row is not immediately reclaimed. You have to run a job to delete the rows and run `ALTER TABLE ENGINE=MEMORY` to reclaim the space ([reference][memory_store_docs]). However, we switched `Rack::Attack` to use this new store and everything worked well. We now had a transactional, replicated cache store for pretty much no additional effort (we were already running replication for sessions). 

However, I wondered how much of a performance hit we were taking by not using a cache like Memcache or Redis. And how far we could take this notion of using a database as a cache. Only one thing to do...

### The Benchmark
For these tests, we're not looking for absolute performance, just relative performance. To keep things simple, we're also going to ignore replication for the moment. Now, people like [@Antirez][antirez] have [previously stated][antirez_blog] that straight comparative tests are bad because rarely do they compare apples to apples. In this case, we do have a real world use case, (rate limiting) so we should be able to compare a specific functionality to perform a specific task. In this case, it was an atomic increment/read.

I wrote some multi-threaded ruby code that will try and increment the same counter as often as possible, and return the new value. The test runs for 10 mins and then the number of threads is upped by one and then run again. While Memcache/Redis/Mongo seemed to have support for atomic increments builtin, I had to hand roll a solution for MySql/PostgreSQL. You can find the source [here][cache_speed]. 

There is a [Vagrantfile][cache_speed_vagrant_file] that was used for testing out the setup of the virtual machine, but there is quite a big discrepancy between VM and baremetal so I've just gone with the baremetal benchmark. Here are the abbreviated machine specs for the baremetal machine:

{% highlight bash %}
$ lscpu
Architecture:          x86_64
CPU(s):                8
Thread(s) per core:    2
Core(s) per socket:    4
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
CPU MHz:               800.000
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              8192K
NUMA node0 CPU(s):     0-7

$ hardinfo -m computer.so -m devices.so
Processor   : 8x Intel(R) Core(TM) i7-4770 CPU @ 3.40GHz
Memory    : 24643MB (12593MB used)
Operating System    : Ubuntu 14.04.1 LTS
{% endhighlight %}

Here are the results when run on bare metal:

<div id="curve_chart" style="width: 1024px; height: 500px"></div>

Firstly, I was unable to detect any missing updates so all applications made good on their atomic guarantees. Apart from PostgreSQL, all the applications seemed to follow the same trend: As contention increases, performance degrades. What is unusual about PostgreSQL is that performance peaked at 4 simultaneous threads. What was also impressive about PostgreSQL is when you disable reliable writes (i.e. turn fsync off), it's performance was right up there with Memcache. Not having to synchronise with disk also had huge effect on MySql can be seen in the difference between the Raw/ActiveRecord and InnoDb benchmarks. Spinning rust is still slow, no surprises there.

What the graph shows is that you're giving up between 200%-400% increase in performance by going with an in-memory database vs a dedicated cache.  sk 
<< link to article about token buckets >>

<div>
<script type="text/javascript"
          src="https://www.google.com/jsapi?autoload={
            'modules':[{
              'name':'visualization',
              'version':'1',
              'packages':['corechart']
            }]
          }"></script>

    <script type="text/javascript">
      google.setOnLoadCallback(drawChart);

      function drawChart() {
        var data = google.visualization.arrayToDataTable([
          ['threads', 'memcache','mongo','mysql','mysql_active_record','mysql_innodb','postgresql','redis'],
          [1,29179.64946,3524.04911,8927.773566,5287.167061,22.63884816,7551.863838,36835.47839],
          [2,25135.28179,3777.43276,9680.855079,5249.015376,23.10080608,12391.3972,35974.08789],
          [3,16787.24241,2725.825904,6644.377302,4009.478062,31.8018856,15058.59639,23178.44918],
          [4,13725.09775,2307.414301,6750.850186,3897.162178,37.47016117,16037.54373,18584.6409],
          [5,12571.70689,2168.879302,6714.937116,3772.134072,43.38051911,15246.04916,15858.28759],
          [6,11996.58841,2144.936469,6559.21691,3833.138837,48.69644796,13346.31871,16500.57247],
          [7,12357.70597,2215.80296,6591.915423,3935.496605,53.46200005,11881.92501,16161.51398],
          [8,12330.02257,2251.495459,6539.590119,3862.827351,57.71824777,11324.39877,16496.58532],
          [9,13205.48478,2319.548457,6565.990911,3749.440255,61.95733192,11000.70656,16851.1753],
          [10,12956.24833,2307.010231,6546.703689,3853.656253,65.24417233,10858.94856,16858.07616]
        ]);

        // https://google-developers.appspot.com/chart/interactive/docs/gallery/linechart#Configuration_Options
        var options = {
          title: 'Requests/sec (higher is better)',
          curveType: 'function',
          legend: { position: 'right' },
          backgroundColor: '#fdfdfd',
          chartArea:{ left:100, width: '50%'},
          hAxis: { title: 'Threads',  ticks: [1,2,3,4,5,6,7,8,9,10] },
          vAxis: { title: 'Requests' },
        };

        var chart = new google.visualization.LineChart(document.getElementById('curve_chart'));

        chart.draw(data, options);
      }
    </script>
</div>


[antirez]:                   https://twitter.com/antirez
[antirez_blog]:              http://antirez.com/news/85
[cache_speed]:               https://github.com/rjnienaber/ruby-cachespeed
[cache_speed_vagrant_file]:  https://github.com/rjnienaber/ruby-cachespeed
[rack_attack]:               https://github.com/kickstarter/rack-attack
[dalli]:                     https://github.com/mperham/dalli
[memory_store]:              https://github.com/rails/rails/blob/b03b09dc8660e26ed23a851ebda2bcbcb47d7d0a/activesupport/lib/active_support/cache/memory_store.rb#L77
[memeory_store_docs]:        http://dev.mysql.com/doc/refman/5.0/en/memory-storage-engine.html