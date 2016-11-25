class: center, middle

Stream-Processing In A
======================

High Traffic Environment
========================

**From Startup To Post-Acquisition**

[Daniel Pezely](https://linkedin.com/in/dpezely)  
Formerly with Splunk/BugSense  
(now with [Snagz.net](http://snagz.net/))

**Data Pipelines & Distributed Systems** Meetup  
at STAT Search Analytics, Vancouver, BC

24 November 2016

???

*Stream-Processing in a High Traffic Environment*

BugSense == crash reporting & analytics

Agenda:

1. Before acquisition
2. After acquisition / assimilation
3. Lessons Learned

---
## Context

<image src="SplunkMINTOverview_MINT_v1.0.png" width="100%" height="100%">

We'll look inside that "cloud" icon from **before and after**
[BugSense](https://web.archive.org/web/20130220160535/http://www.bugsense.com/)  
was acquired and became [Splunk MINT](http://Splunk.com/mint) ...

???

Before acquisition:  
the only dashboards to exist were those at top of image

After becoming a Splunk App:  
added the items down and to your right

---
## Background

BugSense founded in 2011 in Athens, Greece

Total funding was USD $100k plus a hosting grant

Was #2 (behind Google) for crash reporting & analytics of mobile apps

Handled many billions of requests daily, non-stop from around the world

Acquired by Splunk in 2013 with headcount of 12  
via connection made six months earlier at *Erlang Factory SF 2013*

The BugSense offering became a Splunk App-- now called MINT:  
*Mobile Intelligence* [Splunk.com/mint](http://splunk.com/mint)

( Splunk also acquired Metaphor Software in 2015,  
maintaining an office presence in Vancouver )


---
## Bio

Daniel joined BugSense after its acquisition.  
Maintained Lethe database & stream-processing system  
plus co-developed Data Collector, both of which  
are presented here.

Involved in many startups and large companies:  
Main Street, Wall Street, Silicon Valley, Sydney,  
and places in between-- now in Vancouver.

Currently:

Founder and principal developer of Snagz.net  
document and data-mining system for datasets that  
may benefit from more than Machine Learning alone.


---
class: center, middle



## Part 1:

## Original BugSense Architecture

<image src="BugSense-logo.svg">

**Launch through early days of acquisition**

**( Before BugSense gets re-written as a Splunk App )**





---
## Criteria for LetheDB

Handle crash reporting & analytics of mobile devices:

- Data was mostly linear counters and probabilistic counters (HLL)
- Maintain low burn-rate
  + Rules-out Hadoop and others needing large number of servers
  + 10,000 free-tier customers per node (on commodity server)
- Handle very high traffic, non-stop from around the world
  + Within 2 years, over 500 Million devices reporting daily
  + Traffic appeared to AWS as DDoS attack
- Each node must be relatively autonomous
  + No network calls from worker nodes
- Reply within single-digit milliseconds
  + Because mobile networks introduce enough latency
- Load customer data on-demand
  + Node boots without any data
  + Allows for an oversubscribed server
- "Let it crash"
  + Erlang motto, meaning: *revert to well-defined stable state*
  + Very effective for garbage-collection
- Malleable query language
  + Because an early stage startup is still learning about its niche

???

Lethe == river from [Greek myth](https://en.wikipedia.org/wiki/Lethe)

It's a DB that forgets:
- an in-memory database, much like
  [prevalence](https://common-lisp.net/project/cl-prevalence/)
  or [prevayler](http://prevayler.org/)
- loads customer data on-demand

Let it crash

Malleable query language

---
background-image: url(LetheDB.svg)

## LetheDB

We'll focus on the box highlighted:

???

Not shown:
- Hardware load balancers to eliminate single points of failure

Few server instances for > 30k customers:
- Spring 2014: about 20 server instances
- Used AWS `c4.4xlarge` (Haswell)
  + with much capacity to spare

---
## Design & Implementation

- Stream processor and database-- combined
  + Avoid queuing (you'll never catch-up!)
- Run everything in same memory space to reduce latency
  + One Erlang "process" per customer
  + Query language implemented in C
  + Hash table implemented in C based upon `uthash`
  + However, C serialization/deserialization can become a bottleneck
- Keep distributed systems topology simple in the beginning
  + With an all-in-one design, service discovery is unnecessary
- Manage data-flow with straight-forward layers:
  + Rules for filters
  + Finite State Machines
  + Statistical Methods; e.g., enforcing quotas
- Query language was embedded Lisp (`Scheme R4`) implemented in C
  + But migrated to native Erlang functions for 2015 release
- Store aggregate data only
  + Originally required sub-sampling for large customers
  + Sub-sampling became optional after 2015 release
- No external database
  - No socket/pipe overhead
  - No API translation overhead
- Custom replication engine
  + Intended for hot/cold fail-over only

???

Queuing == Death

Simple layers yield rich results:
- Rules
- FSM
- Stats

Everything in same memory space
- Caveats presented at Erlang Factory SF 2015

Query Language:
- Scheme R4
- Then Erlang

---
## Key Feature: Multi-Tenancy

One Erlang "process" per customer:
- Spawn() on-demand, exits when idle

One set of tables per customer:
- No single customer can block any other &ndash; can only block yourself
- Encourages heavy users to upgrade to Dedicated server

Per-customer tables persisted for N+1 days(*) of retention:
- Key/Value pairs &ndash; mostly as linear counters
- Sets (implemented as arrays)
- Probabilistic counters (HLL)

Originally tracked full history:
- Useful while deciding upon actual retention length
- Settled on 7 days
- Pruned old files via `cron`

(*) N+1 days of retention because of modulo naming scheme for re-using files
on disk and "atoms" within Erlang VM.  One day of padding keeps semantics of
midnight/date-rollover simple, considering late arrival `write` versus `read`
for report generation.


---
class: center, middle



## Part 2:

## New Architecture After Acquisition 

<image src="Splunk-logo-white.png">

**BugSense re-written as a Splunk app**




???

Circa early 2015

Sometimes, policy dictates an architectural change ...

---
## Criteria for Data Collector

- Use core Splunk for storage and processing

- Front-end ingests `150k` requests per second *per node* sustained
  + Software load balancer written in Erlang
  + App-layer router: crash dumps, key/value pairs, metrics
  + Rules for filtering
  + Rules for transforming some data
  + Rules for blocking traffic; e.g., if no longer a customer
  + Accommodate mirroring production traffic

- Feeds data into Splunk Indexer
  + Typically deployed as cluster with 3 or more Indexers
  + Indexers can take much time to recover after a crash
  + Accommodate temporary network partitions

- Therefore, must handle "live" versus "stale" data
  + Always deliver "live" data, if any
  + Throttle delivery of "stale" data, at nominal 20% max

---
background-image: url(DataCollector.svg)

## Data Collector

- LetheDB replaced by **Data Collector**
- Google App Engine replaced by **Splunk MINT** App


???

Again, not shown:
- Hardware load balancers to eliminate single points of failure

Splunk cluster shown as simplified representation

---
## Design & Implementation

First version (MVP) was single-tenant, second added multi-tenancy:
- As before with LDB, spawn customer's Erlang "processes" on-demand
- Each customer represented by bundle of processes (LDB used just one)

Processes within bundle:(*)
- Ingress
- Egress
- Stale Queue
- Live Queue
- Transaction handler
- Transaction acknowledger
- Manage replay of "stale" data

More Erlang, better Erlang:
- Developers had learned much about the language, frameworks, libraries
- Made better use of proper releases
- Proper releases facilitated use of introspection tools & monitoring
- Made better use of non-trivial Supervisor trees

(*) In Erlang, each "process" gets one mailbox.  Adding a companion process
can help with timely responses, such as when the other blocks on I/O.  

???

Again, spawn processes on demand

But using bundle of processes

e.g., process for Live versus Stale queues

n.b.,
- We each learned Erlang on the job
- With much success

---
## Key Feature: Pipeline With Multiple Priorities

Many systems only offer uni-directional queue  
because these are simpler to implement  
due to avoiding an entire class of dead-lock scenarios.

But that's for *general purpose* systems!

You know more about your subject domain  
than the library or framework author:

- Why be constrained by someone else's limitations?
- Have you spent more time tuning theirs  
  when you could have more quickly built your own?

Consider a custom Bi-directional queue/mailbox approach:

- Increase throughput by eliminating issues of blocked processes
- Thus, decrease latency of end-to-end system overall
- Even when faked with a pair of queues
- Led to creation of `gen_rpc` library:
  <https://github.com/priestjim/gen_rpc>

---
class: center, middle



## Part 3:

## Lessons Learned





---
## Founder's Perspective

Build a *deployable* Minimum Viable Product (MVP)  
Or as some call it, "minimum *valuable* product"

Plan to be `#FundedByRevenue`, then more funding simply helps you  
grow faster and into more markets

(From a tech co-founder's perspective, this is similar to  
designing without a commercial Load Balancer &mdash; knowing that  
you can always add one for more headroom later.)

*also*

Hire employees who are entrepreneurial-minded

Being entrepreneurial doesn't necessarily mean that this person will be
CEO.  It simply implies a mindset of:
- Resourcefulness counts far more than book knowledge,  
  yet has strong working knowledge to get traction quickly
- Work smarter, not harder
- Can elegantly balance: *fast, cheap, good &ndash; pick two*

---
## Product Manager's Perspective

It's all about *managing complexity* without being complicated:
- Keep it simple
- But strive for design elegance

Build versus buy versus use open source:
- *Build* was appropriate for getting traction and considering burn-rate
- Getting *everything you need, nothing you don't* worked **twice**

Consider that as an early-stage startup, you are continually  
discovering the problem space as well as the solution space:
- Malleable query language (e.g., embedded Lisp) was a huge win here

Other non-tech criteria to consider:
- There's a time to be *Lean* and a time to *throw money* at the problem,  
  thus for an established company, adding servers might be right!
- How should on-boarding new staff impact *systems design*?  
  (We learned and mastered Erlang on the job)
- New employees poached while en route for their *first day*...
  `#SiliconValleyProblems`

---
## Software Developer's Perspective

Simple layers yield rich behaviour:
- Rules for filtering
- Finite State Machines (FSM)
- Statistical methods; e.g., for managing quotas

"Let it crash" used with great success:
- Again, this means: *revert to well-defined stable state*
- Also effective for garbage-collection

Ability to mirror production traffic was huge win:
- Very fast iteration cycles
- Customer data never leaves hardened & monitored environment

Programming Methodology:
- Highly recommend *functional programming* approach for correctness
- Test early & often: required 80% code coverage, happily reached 92%

Use of "exotic" programming languages can be *strategic advantage*
- Competitive advantage for acquiring and retaining staff
- The right language accommodates doing more with less

---
## Multi-Tenancy For Multiple Priorities

*Combining the two systems architectures...*

Multi-tenant mechanics may also be used for *priority* messaging!
- Only messages of same rank/colour may go through same stream
- Backlog can only affect other messages of same priority, same customer

Instead of managing as customers or as message versus Out-Of-Band (OOB),  
Augment with attributes such as from Service Level Agreement (SLA)
- customer-Foo-alert
- customer-Foo-absolute-positioning
- customer-Foo-position-deltas
- customer-Foo-transactions
- customer-Foo-housekeeping

Offers more knobs & levers for scaling or controlling capacity

Not just for scaling up... but also scaling down later:
- Scale a service up while heading into its peak  
  (single tenant, multiple priority)
- Consolidate servers, and scale down while riding long tail before sunset  
  (multi-tenant, multiple priority)

---
## For More Information

Full series of original presentations:

1. <http://highscalability.com/blog/2012/11/26/bigdata-using-erlang-c-and-lisp-to-fight-the-tsunami-of-mobi.html>
2. <http://www.erlang-factory.com/conference/SFBay2013/speakers/DionisisKakoliris>
3. <http://www.erlang-factory.com/sfbay2014/jon-vlachogiannis>
4. <http://www.erlang-factory.com/sfbay2015/daniel-pezely>
5. <http://www.erlang-factory.com/sfbay2016/panagiotis-papadomitsos>
6. <https://github.com/priestjim/gen_rpc>

Much credit for LDB's and Data Collector's success goes to:  
Panagiotis "PJ" Papadomitsos
[Linkedin.com/in/priestjim](https://www.linkedin.com/in/priestjim)
&mdash;
[@priestjim](https://twitter.com/priestjim)

Founders of BugSense:

- Jon Vlachogiannis
  [Linkedin.com/in/johnvlachoyiannis](https://Linkedin.com/in/johnvlachoyiannis)
  &mdash;
  [@jonromero](https://twitter.com/jonromero)
- Panos Papadopoulos
  [Linkedin.com/in/panayiotis](https://Linkedin.com/in/panayiotis)
  &mdash;
  [@panosjee](https://twitter.com/panosjee)

BugSense is now Splunk Mobile Intelligence (MINT)  
[Splunk.com/mint](http://Splunk.com/mint)
&mdash;
[docs.splunk.com/Documentation/Mint](http://docs.splunk.com/Documentation/Mint)
