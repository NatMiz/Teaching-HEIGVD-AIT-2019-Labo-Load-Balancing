# AIT - Laboratoire 3 - Load Balancing

**Authors: Olivier Koffi, Nathanaël Mizutani**

### Introduction

In this lab, we will use HAProxy in different cases.<br/>
At first, we will test the proxy with default configurations, that means round-robin mode without sticky-sessions<br/>
Then, we will configure the proxy to manage sticky-sessions.<br/>
Then, we will experiment the DRAIN and MAINT mode.<br/>
Finally, we will configure and compare different balancing modes.<br/>

### Task 1: Install the tools

*1. Explain how the load balancer behaves when you open and refresh the
URL <http://192.168.42.42> in your browser. Add screenshots to
complement your explanations. We expect that you take a deeper a
look at session management.*

**Answer**

When we refresh the browser on the URL http://192.168.42.42 we can see that the request is send alternatively on each web servers. HA Proxy seems to work with a round robin configuration by default.

We also can see that the NODESESSID is refreshed for each request. This is because each server receives the token from the other and doesn't recognize it. So it sets a new cookie containing a new NODESESSID. Then, the other server do the same, etc...

In fact, we have a setup that demands sticky sessions but HA Proxy is configured as a stateless mode by default.

![firstConnection](./images/firstConnection.png)
![secondConnection](./images/secondConnection.png)

*2. Explain what should be the correct behavior of the load balancer for
session management.*

**Answer**

HA Proxy should be configured to be statefull. That means it should forwards existing sessions on the correct server. This is what we call sticky sessions.

*3. Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2.*

**Answer**

![](images/SeqDia_task1_3.png)

*4. Provide a screenshot of the summary report from JMeter.*

**Answer**

![](images/JMeterSummaryReportTask1.png)

*5. Run the following command:*

  ```bash
  $ docker stop s1
  ```

*Clear the results in JMeter and re-run the test plan. Explain what is happening when only one node remains active. Provide another sequence diagram using the same model as the previous one.*

**Answer**

![JMeter Summary Report s1 down](images/JMeter-SummaryReport-Task1-5.png)

Now HA Proxy forwards all trafic on S2 because it's the only server up.
S2 now recognizes the cookie NODESESSID it set so it doesn't need to set a new cookie.
The session is established, so the sessionViews is incremented at each new request but it's not because of HA Proxy is configured to manage sticky sessions, it's only because there is only one server up so all the trafic is always forward there.

![](images/SeqDiaTask1_5.png)

### Task 2: Sticky sessions

*1. There is different way to implement the sticky session. One possibility is to use the SERVERID provided by HAProxy. Another way is to use the NODESESSID provided by the application. Briefly explain the difference between both approaches (provide a sequence diagram with cookies to show the difference).*

**Answer**

With SERVERID, HA Proxy is configured to add a cookie (named SERVERID) in the server response to identify the server. When the client send another request, the proxy read the SERVERID cookie and forwards it to the correct server. HA Proxy inserts SERVERID cookie in the server's response only if it doesn't exist in the client's request, if it already exists it just forwards requests and responses.

With NODESESSID, it's almost the same operation. The application on the server set a NODESESSID and then the proxy concatenate the server id in the cookie. When the client send another request, the proxy read the cookie and extract its part the redirect on the correct server.

*Choose one of the both stickiness approach for the next tasks.*

We'll use the SERVERID mechanism for the following manipulations.

*2. Provide the modified `haproxy.cfg` file with a short explanation of the modifications you did to enable sticky session management.*

**Answer**

![](./images/task2_haproxy_cfg.png)

The line starting with `cookie` indicates to HA Proxy that it must insert a `Set-Cookie` named `SERVERID` in the response's header when a client requests for the first time (or rather if the client has not a SERVERID cookie already set).

We also added `cookie s1` for server s1 and `cookie s2` for server s2 to set the content of the SERVERID cookie that will be store in the client's browser.

*3. Explain what is the behavior when you open and refresh the URL <http://192.168.42.42> in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.*

**Answer**

![](./images/task2_browser.png)

![](./images/burpTask2_3.png)

Now, when the page is refreshed several times, the NODESESSID still the same and `sessionViews` is incremented. That shows that HA Proxy is now configured to manage sticky sessions.

In the above image, we can see the SERVERID cookie set to s1. That's because HA Proxy set it up in the first response and now as long as the client's browser keeps the SERVERID cookie, HA proxy will receive it in each requests and forwards messages to s1.

*4. Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. We also want to see what is happening when a second browser is used.*

**Answer**

![](./images/SeqDiaTask2_4.png)

*5+6. Provide a screenshot of JMeter's summary report. Is there a difference with this run and the run of Task 1? Give a short explanation of what the load balancer is doing.*

  * *Clear the results in JMeter.*

  * *Now, update the JMeter script. Go in the HTTP Cookie Manager and
    <del>uncheck</del><ins>verify that</ins> the box `Clear cookies each iteration?`
    <ins>is unchecked</ins>.*

  * *Go in `Thread Group` and update the `Number of threads`. Set the value to 2.*

![](./images/task2_5.png)

**Answer**

This question could be tricky because it looks like there no difference between this test above and the test from the task 1 but in fact, it's completely different.

In the task 1, there is only one thread (user) sending 1000 requests and HA Proxy simply dispatch each request in round robin mode. That means request goes on s1, then s2, then s1, etc. But there is no sticky sessions. A new `Set-Cookie: NODESESSID` is added in header's response from each requests !

In this test, there is now 2 threads (users) sending 1000 requests each but now sticky sessions is configured. That means user1 sent 1000 requests on s1 and user2 sent 1000 requests on s2.

s1 and s2 added a `Set-Cookie` in the header's response only at the first request from user1 and user2 respectively. The `sessionViews` has been incremented to 1000 for each user.

### Task 3: Drain mode

*1+2. Take a screenshot of the Step 5 and tell us which node is answering.*

**Answer**
![](./images/task3_1.png)

The page has been refreshed 30 times. We can see that is the node s2 who is answering.

*2. Based on your previous answer, set the node in DRAIN mode. Take a screenshot of the HAProxy state page.*

**Answer**

![](./images/task3_2_1.png)

![](./images/task3_2_2.png)

We can see that the s2 node line changed to blue (active or backup SOFT STOPPED for maintenance)

*3. Refresh your browser and explain what is happening. Tell us if you stay on the same node or not. If yes, why? If no, why?*

**Answer**

![](./images/task3_3.png)

When we refresh the browser, we stay on the same node.
It's normal because as explain in the course, DRAIN mode let current sessions continue to make requests to the node in DRAIN mode and will redirect all other traffic to the other nodes.

*4. Open another browser and open `http://192.168.42.42`. What is happening?*

**Answer**

![](./images/task3_4.png)

As expected, all other traffic is redirect to the other node s1.

*5. Clear the cookies on the new browser and repeat these two steps multiple times. What is happening? Are you reaching the node in DRAIN mode?*

**Answer**

![](./images/task3_5.png)

The DRAIN mode node (s2) is never reach because only the current sessions when I set to node in DRAIN mode are redirect on this node. Again, all other traffic will be redirect on s1.

*6. Reset the node in READY mode. Repeat the three previous steps and explain what is happening. Provide a screenshot of HAProxy's stats page.*

**Answer**

![](./images/task3_6_1.png)

![](./images/task3_6_2.png)

Now, we can see that s2 is reached one in two with the new browser when cleaning cache between each request. This is the normal round robin mode.

*7. Finally, set the node in MAINT mode. Redo the three same steps and explain what is happening. Provide a screenshot of HAProxy's stats page.*

**Answer**

![](./images/task3_7_1.png)

![](./images/task3_7_2.png)

![](./images/task3_7_3.png)

![](./images/task3_7_4.png)

This time, we can see that both browser (firfox with current session and chrome without session) have switch to s1 and the s2 node line in the HAProxy's state page change to brown (active or backup DOWN for maintenance (MAINT)).

As expected, in MAINT mode, the node can't be reach be any requests even these with sessions already established. All traffic is redirect on the only node UP, s1.

### Task 4: Round Robin in degraded mode

*Remark*: In general, take a screenshot of the summary report in JMeter to explain what is happening.</br>
*Remark*: Make sure you have the cookies are kept between two requests.

*1. Be sure the delay is of 0 milliseconds is set on `s1`. Do a run to have base data to compare with the next experiments.*

**Answer**

![](./images/task4_1_1.png)

![](./images/task4_1_2.png)

*2. Set a delay of 250 milliseconds on `s1`. Relaunch a run with the JMeter script and explain what it is happening?*

**Answer**

![](./images/task4_2_1.png)

![](./images/task4_2_2.png)

Now we can see the average response time for a request on s1 is 292 [ms]. It's because we configured s1 to put a delay of 250 [ms] before send a response.

*3. Set a delay of 2500 milliseconds on `s1`. Same than previous step.*

**Answer**

![](./images/task4_3_1.png)

![](./images/task4_3_2.png)

Now we ca see that s1 is never reached. All traffic goes on s2.

*4. In the two previous steps, are there any error? Why?*

**Answer**

![](./images/task4_4_1.png)

With 250 [ms] delay, s1 is slow but there is no error for HA Proxy. When we configure s1 with a 2500 [ms] delay, HA Proxy considers s1 as down. And that's why all traffic is redirect to the other nodes.

*5. Update the HAProxy configuration to add a weight to your nodes. For that, add `weight [1-256]` where the value of weight is between the two values (inclusive). Set `s1` to 2 and `s2` to 1. Redo a run with 250ms delay.*

**Answer**

![](./images/task4_5_1.png)

/!\ We need to rebuild images to actualize configuration.

*6. Now, what happened when the cookies are cleared between each requests and the delay is set to 250ms ? We expect just one or two sentence to summarize your observations of the behavior with/without cookies.*

**Answer**

With cookie :

![](./images/task4_6_1.png)

- We can see that there is no difference in terms of response's time because HA Proxy is configured to manage sticky sessions so all traffic is send to s1 because there is a session between the client and s1.

Without cookie :

![](./images/task4_6_2.png)

- Now HA Proxy behaves like a "normal" load balancer because there is a new session at each request.<br/>
S1 weighs 2 and S2 weighs 1, that means on 3 requests two will be send to S1 and one to S2.<br/>
So the average response's time is a bit better because S2 has "0" [ms] delay.<br/>
But it would be a better configuration to set a heavier weight to S2 because it has a better time response than S1. It would improve the average response's time.

### Task 5: Balancing strategies

**1. Briefly explain the strategies you have chosen and why you have chosen them.**

**Answer**
* static-rr:<br/> Each server is used in turns, according to their weights.<br/> This algorithm is as similar to roundrobin except that it is static, which means that changing a server's weight on the fly will have no effect. It also uses slightly less CPU to run (around -1%). I chose this one to compare performances with an algorithm that has better performances with long sessions.

* leastconn:<br/> This system configures HAProxy to choose the server with the lowest number of active connections. If several servers have the same load, then Round-robin between them is applicated.<br/> This algorithm is recommended where very long sessions are expected.<br/> I chose this one to compare the performances with an algorithm that has better performances with short sessions.


**2. Provide evidences that you have played with the two strategies (configuration done, screenshots, ...)**

**Answer**

**static-rr:**

Configuration
![](./images/task5_2_1.png)

Without sticky-session
![](./images/task5_2_5.png)

With sticky-sesison
![](./images/task5_2_6.png)

**leastconn:**

Configuration
![](./images/task5_2_2.png)

Without sticky-session
![](./images/task5_2_3.png)

With sticky-sesison
![](./images/task5_2_4.png)

**3. Compare the both strategies and conclude which is the best for this lab (not necessary the best at all).**

**Answer**

For tests, we put the total number of requests at 100'000 to better see performances.<br/> As we expect, the static-rr mode has better response's time average. This is because is the round-robin algorithm is really easy and HAProxy doesn't need to do any complex operation before forward the request and the response. And because we have only short session time for our application, rr-static is a better solution than leastconn.

### Conclusion

This lab allowed us to get familiar with the HAProxy environement and its features. We also found out some other balancing modes (e.g. first, source, uri) that can be used for specific infrastructure types or requirements. <br/>HAProxy configuration file is pretty much easy to administrate and the community edition is free. This tool could be used as reverse proxy for a small or medium infrastructure.
