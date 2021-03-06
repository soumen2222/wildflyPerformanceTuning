# wildlfyPerformanceTuning - Undertow subsystem

Background:

Before we jump into the tuning of the Wildfly undertow subsystem , let us discuss about what led us to this tuning. In most of the real world scenarios, sub system settings of the application server are at default and not tuned until and unless issues are seen in the sub system or system requirements are not met. The same thing happened with us.
In short, we have Wildlfy application server and Apcahe webserver with load balancer running in linux boxes and the application is created with JAVA EE8 technology stacks. We got a requirement to support a mobile application with 2000 Concurrent req/Sec. The mobile application gets/writes data from/to the application server by restful webservices.  We started checking the feasibility of the requirement and after analyzing all the components we found that the critical components, which would help us to support the scale, can be segregated and developed into small application that can run on separate wildfly docker instances.  We moved those components out of the monolithic application and created small application (sort of micro services) but few of the components need to interact with other components and the monolithic application to read and write data. We had few options such as remotejms, remoteejb or restful webservices. Each one has its own merits and demerits, we preferred to use restful webservices mostly because of interoperability with other languages. When the feature was tested with load runner with 2000 concurrent Req/sec, we saw a huge spike in the number of http connections as every request to read or write data from/to other component required a http connection, and unfortunately the application servers were not able to handle those many requests and they was going down within few minutes of surge in the incoming requests.  In the original monolithic application the interaction between the components were mostly local ejb calls, which has no overhead of remote connections.  So, this all led us to look in deep about the undertow subsystem, as undertow subsystem is responsible for managing the http, ajp and https connections. We had a great learning while solving the problem. I hope you find this article useful, if there are any errors I'd be more than happy to fix them.



a)	Overview of UnderTow 

Undertow (http://undertow.io/) is similar to JWS/Tomcat, Undertow can act both as a web server and a Java EE web container. It can also run in embedded mode, just as it does in WildFly, as well as in standalone mode. Undertow receives vital support for the speed of both blocking and non-blocking I/O. These technologies and techniques are utilized by, for example, WebSockets and asynchronous servlets. Java EE applications -servlet container and Java servlets are defined to utilize blocking I/O. WebSockets and asynchronous servlets utilizes non blocking I/O.
The following three listeners are supported out of the box: HTTP,HTTPS and AJP.

![Alt text](/images/image1.png?raw=true "Listeners")

Undertow has been designed to make full use of the ultra-high-performance XNIO framework. XNIO, in turn, builds on the Java NIO API. XNIO provides full NIO support and enhances it by providing an enriched and simplified API with a partially higher abstraction layer. XNIO also offers other related functionalities, such as callbacks, multicast, and socket support (both blocking and non-blocking).

XNIO Workers- A worker has the role of a coordinator that creates listener channels and manages thread pools. These pools are either for worker threads, that are responsible for various user-defined actions, or for I/O threads that handle things such as cancellation events and callbacks for reading or writing events. I/O threads needs to be tuned if there are websockets and asynchronous servlets.


b) Tuning of Undertow

Tuning 1: 

Two key components are configured by the Undertow subsystem. First, there is a XNIO worker pointed out by the worker attribute and named default by default. Secondly, there is a buffer pool pointed out by the buffer pool attribute. 
          1)	Our first component is the worker. 
           Sample Undertow Subsystem and Domain IO Subsystem( Taken from the standalone of wildfly version 14)

     <server name="default-server">
                <ajp-listener name="ajp" socket-binding="ajp" worker="ajpworker"  scheme="https"/>
                <http-listener name="default" socket-binding="http" worker="httpworker" redirect-socket="https" enable-http2="true"/>
                <https-listener name="https" socket-binding="https" security-realm="ApplicationRealm" enable-http2="true"/>
                <host name="default-host" alias="localhost">
                    <location name="/" handler="welcome-content"/>
                    <filter-ref name="server-header"/>
                    <filter-ref name="x-powered-by-header"/>                    
                </host>
            </server>
        <subsystem xmlns="urn:jboss:domain:io:1.1">
            <worker name="httpworker" io-threads="16" task-keepalive="90" task-max-threads="128"/>
            <worker name="ajpworker" io-threads="16" task-keepalive="90" task-max-threads="128"/>
            <worker name="default" io-threads="8" task-keepalive="60" task-max-threads="128"/>
            <buffer-pool name="default"/>
        </subsystem>
We can see  that in the undertow susbsystem we have used two worker ( ajpworker and httpworker) , these two workers are defined in the domain io subsystem. These information can be fetched from CLI command or Admin Console.
CLI Command:
/subsystem=io/worker=default:read-resource-description
/subsystem=undertow/server=default-server/http-listener=default:read-attribute(name=worker)



Worker Configuration details:

![Alt text](/images/image3.png?raw=true "Worker Configurations")

    2)	The second component is buffer pool. CLI Commands to see the configuration in Undertow and IO susbsystem.
    
    /subsystem=undertow/server=default-server/http-listener=default:read-attribute(name=buffer-pool)
    /subsystem=io/buffer-pool=default:read-resource-description

![Alt text](/images/image4.png?raw=true "Buffer Configurations")



Tuning 2: 

•	Tuning the Servlet Container
Ignoring flushes can provide better performance in most cases. CLI Command to see and update the configurations:
/subsystem=undertow/servlet-container=default:read-attribute(name=ignore-flush)
/subsystem=undertow/servlet-container=default:write-attribute(name=ignore-flush, value=true)

The standalone – Undertow subsystem will be updated:

<servlet-container name="default" ignore-flush="true">
                <jsp-config/>
                <websockets/>
  </servlet-container>

Tuning 3:

•	Use ajp protocol instead of Http protocol to communicate between frontend (apache server) and backend (application server). To enable ajp protocol, add a ajp worker in the wildlfy and ajp connector in the apache.

![Alt text](/images/image2.png?raw=true "Ajp Connections")

Tuning 4:

•	Use gunzip filter in the undertow subsystem to compress the packet. Gunzup adds CPU cycles so compressing every packet is CPU intensive it is better to compress if the packet size is more than a defined size may be 1 MB. Sample configuration which shows gunzip is enabled for packets whose size is greater than 500kb.

<host name="default-host" alias="localhost">
                    <location name="/" handler="welcome-content"/>
                    <filter-ref name="server-header"/>
                    <filter-ref name="x-powered-by-header"/>
                    <filter-ref name="zipfilter" predicate="not min-content-size[500]"/>
                </host>
         <filters>
 <response-header name="server-header" header-name="Server" header-value="WildFly/10"/>                                 
<response-header name="x-powered-by-header" header-name="X-Powered-By" header-value="Undertow/1"/>
                <gzip name="zipfilter"/>
            </filters>



Enable statistics of the undertow subsystem to see the traffic of each of the worker in the admin portal. 
/subsystem= undertow:write-attribute(name=enable-statistics,value=true)


All the configurations are available in the attached standalone.


Note:
Reference: WildFly Performance Tuning by Arnold Johansson; Anders Welén
