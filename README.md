# Efficient scaling of Multi-paradigm computer systems

As a website's traffic increases, so does the load on server equipment. This can lead to long response times, instability and even complete service shutdowns, making it impossible to ensure a good experience for all of your users. By separating the hosting of your business logic and Relational Database, you can optimize workloads with a combination of vertical and horizontal scaling. This lets you deliver a stable, responsive experience to your users, at a lower cost than simply throwing money at your hosting provider.

## Introduction
One of the nicest problems that you could ever have is that you have a service that is just so popular, that it struggles to handle serving all of your users. It is certainly a much better problem to have than not having any users at all! We intend to demonstrate the effects of high load and scaling by using a project that we have made as part of the LSD-Course, but we want to emphasize the fact that the problems and methods for finding solutions are totally universal and apply to pretty much any project.

While the load put on the service we made for the LSD project is nowhere near enough for it to break a sweat, we will simulate a higher load on a weaker instance to demonstrate the issues and how they can be solved. Now obviously, the simplest and safest way to make a service faster is to just scale vertically... right? Just put some more RAM in there and all problems just go away, right? Not so fast. We'll get to that.

Steps needed to scale:

    Measure and identify bottlenecks
    Identify components that can be split up and made independent
    Load balance

So first we replicated the entire setup on a small droplet. These are the components of the service:

    main java server software
    mysql
    python server for misc perf charts

On our LSD project, all of these run on the same droplet.

## What is scaling?
Scaling an application means allocating resources to match its needs. There is no one-size-fits-all solution: For example,
most Relational Database Systems (MySQL, MS SQL Server, PostgreSQL) are designed to be monolithic and scale vertically, 
while newer paradigms like Distributed Databases are designed to make effective use of horizontal scaling. The same goes for largely stateless applications like a front-end Webserver, which can be distributed as long as a reasonably consistent dataset can be achieved, either by caching or using a Distributed Database

### Horizontal scaling
Horizontal scaling is spinning up additional instances of an application, that perform the same work concurrently. These applications can be tightly interconnected, constantly talking back and forth to synchronize their state and optimize reliability and performance, as is the case with Distributed Databases. RESTful web applications however, being largely stateless on the server side, lend themselves quite well to a somewhat centralized data store, fanning out to independent front-end instances, since whatever state they work with is confined to each individual users's session, and is in fact left up to the client to store between requests.

## Our setup
The Production system we are maintaining is composed of three web applications on a single web server, sharing a virtual machine with the relational database. The machine is very powerful, and we have yet to see any performance degradation at all, even though our userbase and their activity has increased steadily since release. This is only comforting to a certain degree, since it also means we have no idea where the limits of the system are.

#### Droplet A1 (Frankfurt):
```
4GB, 2vCPU DigitalOcean Droplet:
    - Wildfly server
    - MariaDB instance
```

Our main problem in trying to stress test our current system stemmed from the fact that it has been very generously proportioned from the start. This was an expensive solution, but deemed necessary due to uncertainty about the loads we'd be facing in production. The overpowered nature of the system means a proper stress test would be prohibitively expensive. For example, we were unable to see any effect on response time when simulating a doubling of our page-views (The maximum we were able to achieve without spinning up multiple expensive load-simulator machines). 

## Hypothesis
This led us to assume our system to be needlessly overpowered, but unable to confirm to which degree this was true, and therefore how much growth we could sustain without issues. Also, we assumed that we were paying for more computational power than we were making use of.

## Experimental setup

In order to test this hypothesis, we designed an experiment that would let us compare the performance of our system with and without the proposed changes. This involved setting up two smaller VMs, each with a quarter of the memory and half the CPUs of our production server.  

##### Droplet A2 (Frankfurt):
```
1GB, 1vCPU DigitalOcean Droplet:
    - MariaDB instance
```

##### Droplet B (Frankfurt):
```
1GB, 1vCPU DigitalOcean Droplet:
    - Wildfly server
```

##### Droplet C (Singapore)
```
1GB, 1vCPU DigitalOcean Droplet:
    - Pageview simulator
```

## Results

### Plots
##### Current setup:
Frankfurt(Client) -> Frankfurt(Server+DB) (3 x 1000 requests)

1st Run                                      |  2nd Run                                     |  3rd Run
:-------------------------------------------:|:--------------------------------------------:|:--------------------------------------------:
![Frankfurt to Frankfurt, 1st run][fra-fra1] | ![Frankfurt to Frankfurt, 2nd run][fra-fra2] | ![Frankfurt to Frankfurt, 3rd run][fra-fra3]

##### International connection
Singapore -> Frankfurt (3 x 50 requests)

1st Run                                      |  2nd Run                                     |  3rd Run
:-------------------------------------------:|:--------------------------------------------:|:--------------------------------------------:
![Singapore to Frankfurt, 1st run][sgp-fra1] | ![Singapore to Frankfurt, 2nd run][sgp-fra2] | ![Singapore to Frankfurt, 3rd run][sgp-fra3]

##### Split setup as proposed
Frankfurt(Client) -> Frankfurt(Server) -> Frankfurt(DB) (3 x 1000 requests)

1st Run                                                            |  2nd Run                                                           |  3rd Run
:-----------------------------------------------------------------:|:------------------------------------------------------------------:|:--------------------------------------------------------------------:
![Frankfurt through Frankfurt to Frankfurt, 1st run][fra-fra-fra1] | ![Frankfurt through Frankfurt to Frankfurt, 2nd run][fra-fra-fra2] | ![Frankfurt through Frankfurt to Frankfurt, 3rd run][fra-fra-fra3]

### Performance
It's apparent from our plots, that the experiment has resulted in a slightly higher minimum - and a lower maximum - response time for the short-distance case. 
You can see that by being farther (singapore) away, the minimum latency is a lot worse. It is actually about as bad as the worst case on the split setup. The interesting thing here is that by scaling horizontally you would perform MUCH better, simply because you can improve the proximity with several droplets.

### Cost
In our case, running on two smaller VMs instead of a single big one gives a 40% reduction in Web Server and Database hosting, from $25 per month down to 15$ per month.

## Conclusion
In our case, the actual improvements we've seen in performance aren't exactly mindblowing, but this was to be expected given the power of our current production server. However, we were able to identify where it is possible to shave some money off our expenses, and we hope we have shown that that this way of structuring a multi-user application can be interesting to explore, especially if your application has a large, international audience.

## Future experiments
One of the benefits we expected to reap from splitting up our system in this way is an improved ability to serve international users. With a single centralized server, we are unable to provide similar response times worldwide, since some users may be on the other side of the world. Separating our Database and Frontend opens up the possibilit of hosting the front-end in a distributed manner, all across the globe, while still referring to our centralized data store. By letting the front-end cache Database results locally for a short time, we might avoid having to send a slow, international request to the Database for every single view of the frontpage for example. Unique requests however, such as when a user creates a post, or views a thread that hasn't been opened in a while would still incur the full cost of a database look-up, but presumably no more than a long-distance request already does. This is something we would like to explore further in the future.

## References
REST statelessness explained
https://restfulapi.net/statelessness/

Cloud Scalability: Scale Up vs Scale Out
https://turbonomic.com/blog/on-technology/cloud-scalability-scale-vs-scale/

[fra-fra1]: http://46.101.171.5:8000/plot1544397324.4651082.png
[fra-fra2]: http://46.101.171.5:8000/plot1544397324.8890862.png
[fra-fra3]: http://46.101.171.5:8000/plot1544397325.1036382.png
[sgp-fra1]: http://46.101.171.5:8000/plot1544399846.311562.png
[sgp-fra2]: http://46.101.171.5:8000/plot1544399848.7226975.png
[sgp-fra3]: http://46.101.171.5:8000/plot1544399849.0818636.png
[fra-fra-fra1]: http://46.101.171.5:8000/split/plot1544401248.2294571.png
[fra-fra-fra2]: http://46.101.171.5:8000/split/plot1544401249.2195637.png
[fra-fra-fra3]: http://46.101.171.5:8000/split/plot1544401249.3552957.png