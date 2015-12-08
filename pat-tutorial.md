# PIAX Tutorial for PIAX testbed
In this tutorial, we will create an agent that operates on PIAX by following  Collection example attached to the SDK.


Collection searches for an agent having the sensor function on LL-Net and collects data(temperature data) to create statistical information(average temperature). The source code is in the Collection/src in SDK.

The following topics can be learned through this tutorial.

- How to create agents:
    - An agent that does not move and one that can move between peers
- How to search for agents using LL-Net
- How to move an agent
- How to use RPC call

The four different agents: WeatherAgent, CollectStatisticsAgent, MakeStatisticsAgent and MainAgent, will be created in this tutorial.

We will provide a brief explanation of each agent.

### WeatherAgent

WeatherAgent is for collecting and measuring weather information. In this tutorial, prepared temperature data are used.

### MakeStatisticsAgent

MakeStatisticsAgent is for moving to the peer selected by the user to caliculate the average value of temperature data from collected from the near WeatherAgent

### CollectStatisticsAgent

`CollectStatitiscsAgent` finds the `WeatherAgent`, generates and moves the MakeStatisticsAgent to the peer. Then it calls the method of the MakeStatisticsAgent, and calculates the average value of the temperature.


### MainAgent

In PIAX for PIAX testbed, the `main()` function is never called. Therefore, we will create a `MainAgent` class that take on the role of `main()` function.


## 1. WeatherAgent

Create a file, `WeatherAgent.java` with following code.

```
package agents;

import java.util.logging.Logger;

import org.piax.agent.Agent;
import org.piax.agent.NoSuchAgentException;
import org.piax.common.CalleeId;
import org.piax.common.Location;

public class WeatherAgent extends Agent implements WeatherAgentIf {
    private static final Logger logger = Logger.getLogger("WeatherAgent");
    private String city;

    /*
     * This method is called by MakeStatisticsAgent.Use dummy data in this
     * sample program.
     */
    @Override
    public Double getTemperature() {
        // This is dummy data
        Location loc = (Location) getAttribValue("$location");
        double d = PointInfo.getTemperature(loc);
        return Double.valueOf(d);
    }

    @Override
    public void setCity(String city) {
        logger.fine("- setCity(" + city + ") - ID:" + getId());
        this.city = city;
    }

    @Override
    public String getCity() {
        logger.fine("- getCity() - ID:" + getId());
        return this.city;
    }

    @Override
    public CalleeId getWeatherCalleeId() {
        CalleeId id = null;
        try {
            id = getHome().getCalleeId(getId());
        } catch (NoSuchAgentException e) {
            logger.severe("- getWeatherCalleeId() - ID:" + getId()
                    + " exception:" + e.getMessage());
            e.printStackTrace();
        }
        return id;
    }
}
```
This is a class definition of `WeatherAgent`. The class definition is described below.

A PIAX agent is created by inheriting the `Agent` class.

In addition, to enable remote method invocations on PIAX, we need to create a Java interface that has methods which are used in remote method invocations. An example interface file `WeatherAgentIf.java` is described hereafter.

`getWeatherCalleeId()` method is called by `CollectStatisticsAgent` for searching for `WeatherAgent` and returns its own `CalleeId`. `CalleeID` class is an identifier object of an agent on a peer to referred from remote peers. This class is implemented as Serializable in order to send its object data via network.

We do not cover the own class definition. Therefore you need to implement Serializable in your class when you create your own class and use it as a return value of a method in an agent.

`getTemperature()` is a method called by `MakeStatisticAgent`.

As dummy data, predefined data is used. The predefined data is in Collection/src/agents/PointInfo.java. This includes information on the latitude and longitude of various places in Japan and the temperature data on a certain day. These data will be used for acquiring a name of a place or the atmospheric temperature data based on their latitude and longitude.

The explanation of `WeatherAgent` is over. We will create `WeatherAgentIF.java` next.

WeatherAgent has `setCity()`, `getCity()`, `getTemperature()` and `getWeatherCalleeId()`. All of these methods are called from the exteranl.

We will create `WeatherAgentIf.java` with the following code.

```
package agents;

import org.piax.agent.AgentIf;
import org.piax.common.CalleeId;
import org.piax.gtrans.RemoteCallable;

public interface WeatherAgentIf extends AgentIf {
    @RemoteCallable
    public void setCity(String city);

    @RemoteCallable
    public String getCity();

    @RemoteCallable
    public Double getTemperature();

    @RemoteCallable
    public CalleeId getWeatherCalleeId();

}
```
The interface of the agent is created by inheriting the `AgentIf.'

### Points
1. PIAX agents inherit an `Agent` class.
2. PIAX agents implement an interface that is inherited from `AgentIf`.
3. The return value of the method of the agent implements `Serializable`.


## 2. MakeStatisticsAgent

We will create `MakeStatisticsAgent.java` with following code.

```
package agents;

import java.util.List;
import java.util.logging.Logger;

import org.piax.agent.MobileAgent;
import org.piax.common.CalleeId;
import org.piax.common.Location;
import org.piax.gtrans.RPCMode;

public class MakeStatisticsAgent extends MobileAgent implements
        MakeStatisticsAgentIf {
    private static final long serialVersionUID = 1L;
    private static final Logger logger = Logger
            .getLogger("MakeStatisticsAgent");

    CalleeId collectorCid = null;

    // This method is called by CollectStatisticsAgent before travel.
    public void init(CalleeId cid) {
        logger.info("MakeStatisticsAgent init(" + cid + ")");
        collectorCid = cid;
    }

    @Override
    public void onArrival() {
        try {
            /*
             * RPC call CollectStatisticsAgent's arrived method to tell Iarrived
             * the destination.
             */
            getStub(CollectStatisticsAgentIf.class, collectorCid).arrived();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public Double doMake(double r) throws Exception {
        /*
         * Search neighbor WeaatherAgents's getTemperature() methods using
         * LL-Net,and RPC call them.
         */
        Location loc = (Location) getAttribValue("$location");
        String queryCond = String.format(
                "$location in circle(%.1f, %.1f, %.1f)", loc.getX(),
                loc.getY(), r);

        List<Double> results = null;
        try {
            results = getList(getDCStub(queryCond, WeatherAgentIf.class,
                    RPCMode.SYNC).getTemperature());
        } catch (Exception e) {
            e.printStackTrace();
        }

        // calculate temperature average
        Double total = 0.0;
        for (Double d : results) {
            total += d;
        }
        return total / results.size();
    }

    /*
     * This method is called by CollectStatisticsAgent. When this instance isn't needed any
     * more.
     */
    public void bye() {
        destroy();
    }

}
```
That's all for the described of MakeStatisticsAgent. We will explain the class definition in the following part.

The difference of `MakeStatisticsAgent` compared with `WeatherAgent` is that it is inherited class of `MobileAgent` class, not `Agent` class. The agent that inherits the `MobileAgent` class can move between peers. The `MobileAgent` class inherits `Agent` class.

### Member variable collectorCid, init() Method and onArrival() Method

The member variable collectorCid is a variable for saving the AgentId of the CollectStatisticsAgent. This can be set by the init() method. Upon the creation of a MakeStatisticsAgent, the CollectStatisticsAgent immediately calls init() and sets its own AgentId.

`collectorCid` is used when the `MakeStatisticsAgent` calls a method of the `CollectStatisticsAgent`. The `MakeStatisticsAgent` operates as follows. The `MakeStatisticsAgent` moves to the peer specified by the peer having the `CollectStatisticsAgent`. When the movement is completed, the `onArrival()` method is automatically executed. `onArrival()` is a method inherited from the MobileAgent and is the method automatically called when the agent arrives at the target peer. The `arrived` method of `CollectStatitiscsAgent` is called via RPC inside `onArrival()` method. When `Stub` is generated, `collectorCid` is used.

The arrived() method of the CollectStatisticsAgent is called from among the onArrival() methods by means of callOneway(). At this time, collectorCid is used.


### doMake() Method

The `doMake()` method is called from the CollectStatisticsAgent. The `discoveryCall()` is executed on peers within a radius r of the location on the LL-Net where the peer having the MakeStatisticsAgent is. When you wan to tuse `discoveryCall()`, you should create Stub using ` getDCStub()` .


The definition of `getDCStub()` is below.

```
    public <S extends RPCIf> S getDCStub(String queryCond, Class<S> clz, RPCMode rcpMode);
```

The `queryCond` determines the overlay to be used. For a search on the LL-Net, the following strings are specified.

```
    $location in rect(x, y, w, h)
    $location in circle(x, y, r)
    // x is longitude. y is latitude. w is the width of longitude. h is the width of latitude.
    // r is radius of latitude and longitude
```

The interface the target agent implements is specified for `clz`. Whether the method is `oneway` or not (wait for response etc.) is specified for `rpcMode`. The method of target agent is called like `Stub.method()`.

`args` is specified by the argument of the method. The `getTemperature()` method does not have its argument, and , therefore, it is not specified.


### bye() Method

The `bye()` method is also called from the `CollectStatisticsAgent`. This is used to cancel the `MakeStatisticsAgent` that has become unnecessary.

That is all for the descriptions of the MakeStatisticsAgent. In the following part, we will create an interface for the `MakeStatisticsAgent`. The methods of the `MakeStatisticsAgent` are `init()`, `doMake()`, and `bye()`, which are all called externally. We will create `MakeStatisticsAgentIf.java` with following code.

```
package agents;

import org.piax.agent.AgentIf;
import org.piax.common.CalleeId;
import org.piax.gtrans.RemoteCallable;

public interface MakeStatisticsAgentIf extends AgentIf {
    @RemoteCallable
    public void init(CalleeId cid);

    @RemoteCallable
    public Double doMake(double r) throws Exception;

    @RemoteCallable
    public void bye();

    @RemoteCallable
     public CalleeId getCalleeId();
}
```
### Points
- PIAX agents that moves between peers inherit a `MoblieAgent` class.
- Create `Stub` specifying `CalleeId` for RPC calling.
- Create `CollectStatisticsAgent`

The following code is part of `CollectStatisticsAgent.java`. Since `CollectStatisticsAgent.java` is long, please refer to the source file for the entirety. The omitted portion is denoted by "...".

```
package agents;

import ...

public class CollectStatisticsAgent extends Agent implements
        CollectStatisticsAgentIf {

    /* RANGE enclosing Japan */
    public static final double BASE_LAT = 23.0;
    public static final double BASE_LON = 122.0;
    public static final double RANGE_LAT = 24.0;
    public static final double RANGE_LON = 27.0;
    ...

    private static final long collectionTimeout = 300;
    private ExecutorService executor = null;

    private HashMap<String, CalleeId> calleeIdMap = null;
    private HashMap<String, Future<Double>> resultList = null;

    ...

    private static Integer arrivedMarker = 0;
    private Double radius = 1.0;

    @Override
    public void onDestruction() {
        if (executor != null) {
            executor.shutdown();
            executor.shutdownNow();
        }
    }

    // Search WeatherAgents using LL-Net
    @Override
    public String[] searchWeatherAgent() {
        ...

        calleeIdMap = new HashMap<String, CalleeId>();
        ...

        String queryCond = "$location in rect(" + BASE_LON + ", " + BASE_LAT
                + ", " + RANGE_LON + ", " + RANGE_LAT + ")";

        List<CalleeId> results = null;
        try {
            results = getList(getDCStub(queryCond, WeatherAgentIf.class,
                    RPCMode.SYNC).getWeatherCalleeId());
        } catch (Exception e) {
            e.printStackTrace();
        }
        ...

        // Make WeatherAgent's list
        String[] cities = new String[results.size()];
        int i = 0;
        for (CalleeId cId : results) {
            WeatherAgentIf stub = getStub(WeatherAgentIf.class, cId);
            if (stub == null)
                continue;
            String city = stub.getCity();
            calleeIdMap.put(city, cId);
            cities[i++] = city;
        }

        return cities;
    }

    @Override
    public String doCollect(ArrayList<String> collectPoints, double r)
            throws Exception {
        String ret = "Radius is " + r + "\n";

        resetArrivedMarker();

        HashMap<String, CalleeId> makerList = new HashMap<String, CalleeId>();
        // create MakeStatisticsMaker and move it to the collecting points.
        int travelCount = 0;
        for (String city : collectPoints) {
            CalleeId wcId = calleeIdMap.get(city);
            ...

            /* Create MakeStatisticsAgent */
            AgentId aid = getHome().createAgent(MakeStatisticsAgent.class);
            /* Get its own(CollectStatisticsAgent) CalleeId for MakeStatistics onArrival */
            CalleeId mcId = getHome().getCalleeId(aid);
            /* Execute MakeStatistics init */
            getStub(MakeStatisticsAgentIf.class, mcId).init(
            /* my CalleeId for Arrived */
            getHome().getCalleeId(getId()));
            /* Get Endpoint(Peerid) from WeatherAgent CalleeId */
            PeerId pid = (PeerId) wcId.getPeerRef();
            ...

            /* Ignore the exception caused by the fact that the origin and ddestination of travel of travelAgent are same */
            if (!getHome().getPeerId().equals(pid)) {
                /* Move Weather Agent */
                getHome().travelAgent(aid, pid);
                travelCount++;
            }
            makerList.put(city, new CalleeId(aid, pid, null));
        }

        // wait until all MakeStatisticsAgent finished moving.
        long end = System.currentTimeMillis() + collectionTimeout;
        while (arrivedMarker < travelCount
                && (System.currentTimeMillis() < end)) {
            Thread.sleep(100);
        }

        ...

        // call makeStatistics, bye
        AgentPeer agentPeer = getAgentPeer();
        HashMap<Future<String>, String> futures = new HashMap<Future<String>, String>(
                makerList.size());
        for (String city : makerList.keySet()) {
            final double radius = r;
            final String cityName = city;
            final CalleeId cid = makerList.get(city);
            futures.put(agentPeer.submit(new Callable<String>() {
                @Override
                public String call() {
                    String ret = null;
                    MakeStatisticsAgentIf stub = getStub(
                            MakeStatisticsAgentIf.class, cid);
                    try {
                        double d = stub.doMake(radius);
                        ret = cityName + " and environs: " + Double.toString(d)
                                + "\n";
                    } catch (Exception e) {
                        ...

                    } finally {
                        stub.bye();
                    }
                    return ret;
                }
            }), city);
        }

        ...
    }

    @Override
    public void arrived() {
        logger.info("arrived");
        IncrementArrivedMarker();
    }

    private void IncrementArrivedMarker() {
        synchronized (arrivedMarker) {
            arrivedMarker = Integer.valueOf(arrivedMarker.intValue() + 1);
        }
    }

    private void resetArrivedMarker() {
        synchronized (arrivedMarker) {
            arrivedMarker = Integer.valueOf(0);
        }
    }

    ...

}
```
We will explain the class definition of `CollectStatisticsAgent` in following part.

Since the `CollectStatitiscsAgent` does not move between peers, it inherits an `Agent` class instead of the `MobileAgent` class. The interface file, CollectStatisticsIf.java is described below.

There are several member variables, and the variables are described in the descriptions of the methods.

### searchWeatherAgent() Method

This is a method for finding a `WeatherAgent` on the LL-Net and registering it with the member variable `peerInfoMap`. The information registered with the `peerInfoMap` is used by the `doCollect()` method.

The first argument of discoveryCall() is specified as follows for searching on the LL-Net.

```
"$location in rect(122.0, 23.0, 27.0, 24.0)"
```
These figures specified here are for the area that approximately cover the entire Japanese archipelago.

### arrived() Method

`arrived()` has already been described in the `MakeStatisticsAgent` description. This is a method to be called to notify the event that the `MakeStatisticsAgent` has moved to the `CollectStatisticsAgent`. Since there is a possibility of being called simultaneously by some `MakeStatisticsAgent`, this is surrounded by synchronized.


### doCollect() Method

The `collectPoints` of the arguments consist of a list of locations where statistical information is provided. The same number of `MakeStatisticsAgent` as that of the `collectPoints` are created, and `init()` method is called so as to set its own `AgentId`. After that, they are moved to a peer of the `collectPoints` by the `travelAgent()` method.

The arguments of the `travelAgent()` are the `AgentId` and the `PeerId`. The `PeerId` is gained from the `peerInfoMap` that has been collected by the `searchAgent()` method. After that, all the `MakeStatisticsAgent` are waited for until they reach the target peers.

When all the `MakeStatisticsAgent` reach the target peers, the `doMake()` method is called via RPC to acquire statistical information. This call is executed by task which is sent to other thread by `submit` of `AgentPeer`.`

This behavior aims to let each agent operate in multiple thread because we assume that it takes time to create statical information.(Since this is sample program, you will get the return quick. However, the program might wait for input from sensors.)

After that, `CollectStatisticsAgent` call the `awaitTermination` method of `AgentPeer` and wait for the all result of `doMake()` method. Note that there is no guarantee to return the result from all calls. It waits up to 300 seconds in this example.

That is all for the descriptions of the `CollectStatisticsAgent`. In the following, we will create an interface file of the `CollectStatisticsAgent`. Though the `CollectStatisticsAgent` has ten methods, the three out of ten methods: `init()`, `arrived()`, and `doCollect()`,  are called externally. We will create a file ,`CollectStatisticsAgentIf.java`, with the following code.


```
package agents;

import java.util.ArrayList;

import org.piax.agent.AgentIf;
import org.piax.common.CalleeId;
import org.piax.gtrans.RemoteCallable;
import org.piax.gtrans.RemoteCallable.Type;

public interface CollectStatisticsAgentIf extends AgentIf {
    @RemoteCallable
    public String[] searchWeatherAgent();

    @RemoteCallable
    public String doCollect(ArrayList<String> collectPoints, double r)
            throws Exception;

    @RemoteCallable(Type.ONEWAY)
    public void arrived();

    @RemoteCallable
    public CalleeId getCalleeId();

}
```

### Points
- Use the `submit` of `AgentPeer` to execute multiple RPC call parallelly.
- Not to expect all RPC call would be successful
- Do exclusive control the method that may call simultaneously

## Main Agent
We prepare CollectStatisticsAgent and the WeatherAgent and create a MainAgent class to execute them in the PIAX.

The following is part of MainAgent.java. The entirety of MainAgent.java is large, and therefore, refer to the source file for details. The omitted portions are denoted by "...".
```
...

public class MainAgent extends Agent implements MainAgentIf {
    ...

    @Override
    public String collection(String line) {
        ...

        String[] lines = line.split(":");
        ...

        try {
            radius = Double.parseDouble(lines[1]);
        } catch (NumberFormatException e) {
            ...
        }

        ArrayList<String> collectPoints = new ArrayList<String>();
        try {
            for (String s : lines[0].split(",")) {
                int p = Integer.parseInt(s);
                if (0 <= p && p <= weathers.length) {
                    collectPoints.add(weathers[p - 1]);
                } else {
                    logger.severe("That's Bad number. " + p);
                }
            }
        } catch (NumberFormatException e) {
            ...
        } ...

        try {
            ret = getStub(CollectStatisticsAgentIf.class, collectorCid)
                    .doCollect(collectPoints, radius);
        } catch (Exception e) {
            ...
        }
        return ret;
    }

    @Override
    public String cities() {
        String ret = "";
        if (collectorCid == null) {
            try {
                collectorCid = getHome().getCalleeId(
                        getHome().createAgent(
                                CollectStatisticsAgent.class.getName()));
            } catch (ClassNotFoundException e) {
                 ...
            }
        }
        weathers = getStub(CollectStatisticsAgentIf.class, collectorCid)
                .searchWeatherAgent();
        for (int i = 0; i < weathers.length; i++)
            ret += "[" + (i + 1) + "]" + weathers[i] + "\n";
        return ret;
    }

    @Override
    public String preparation(String agentNumString) {
        ...
        int agentNum = -1;
        ArrayList pdList = PointInfo.getPointData();
        ...
        try {
            agentNum = Integer.parseInt(agentNumString);
        } catch (NumberFormatException e) {
            ...
        }
        ...

        /* Discovery Call search for MainAgent within the ragne */
        String queryCond = "$location in rect("
                + CollectStatisticsAgent.BASE_LON + ", "
                + CollectStatisticsAgent.BASE_LAT + ", "
                + CollectStatisticsAgent.RANGE_LON + ", "
                + CollectStatisticsAgent.RANGE_LAT + ")";

        List<CalleeId> results = null;
        try {
            results = getList(getDCStub(queryCond, MainAgentIf.class,
                    RPCMode.SYNC).getCalleeId());
        } catch (Exception e) {
            e.printStackTrace();
        }
        ...

        ArrayList<CalleeId> mainList = new ArrayList<CalleeId>(results);
        ..

        /* Generate WeatherAgent */
        ArrayList<CalleeId> wl = new ArrayList<CalleeId>();
        for (int i = 0; i < agentNum; i++) {
            MainAgentIf stub = getStub(MainAgentIf.class, mainList.get(i));
            stub.setLocationFromPointData(pdList.get(i));
            wl.add(stub.createWeatherAgent(pdList.get(i)));
        }
        logger.info("weatherAgentContextList Length:" + wl.size());
        return "OK";
    }

    @Override
    public boolean setLocationFromPointData(PointData pd) {
        ...

        try {
            getHome().setAttrib("$location", pd.loc);
        } catch (IllegalArgumentException e) {
            ...
        }
        ...

        logger.info("setAttrib(\"$location\"" + getAttribValue("$location")
                + ")");
        return true;
    }

    @Override
    public CalleeId createWeatherAgent(PointData pd) {
        ...
        AgentId aid = null;
        CalleeId calleeId = null;
        try {
            aid = getHome().createAgent(WeatherAgent.class);
            ...
            calleeId = getHome().getCalleeId(aid);
            ...
            WeatherAgentIf stub = getStub(WeatherAgentIf.class, calleeId);
            stub.setCity(pd.name);
        } catch (AgentInstantiationException e) {
            ...
        }
        return calleeId;
    }

    public void init(TestbedInfo ti) throws Exception {
        ...
        try {
            // Plugin LLNET
            AgentPeer apeer = getAgentPeer();
            apeer.declareAttrib("$location", Location.class);
            apeer.bindOverlay("$location", "LLNET");

            // SetLocation for Agent
            Location loc = new Location(
                    (CollectStatisticsAgent.BASE_LON + INIT_RANGE),
                    (CollectStatisticsAgent.BASE_LAT + INIT_RANGE));
            apeer.setAttrib("$location", loc);
            ...
        } catch (Exception e) {
            ...
    }
}
```

In the PIAX testbed, when a *.jar file is loaded, an instance of the MainAgent is created at each peer. The main process can be kicked off by calling the method of the MainAgent. The MainAgent inevitably inherits the org.piax.agent.Agent class and belongs to the default package.

### init()
When the MainAgent is created, this method is called. It should be noted that the peer is not yet in an online state. In the Collection, the location is set to a agent.

### preparation()

In the preparation(), one `WeatherAgent` is created for each peer. Therefore, the `CalleeID`, is acquired from the instance of the existing `MainAgent`. The `getCalleeId()` method is used for acquiring `CalleeId` for each peer.  The `createWeatherAgent()` is called by using the acquired `CalleeId` so as to create a `WeatherAgent`.

#### Search for target peer

The `discoveryCall` is used for searching for the peer which call the `getCalleeId` of `MainAgent`. In order to do this, location information should be set during the process of `init()`.

#### Notes for Location configuration

It is necessary to load the LL-Net overlay to set a location. Before the setting of the location information using `setLocation()` and `setAttrib()`, load the overlay as follows. In the `Collection`, this is done by `init()`.


```
public void init(TestbedInfo ti) throws Exception {
    ...
    AgentPeer apeer = getAgentPeer();
    apeer.declareAttrib("$location", Location.class);
    apeer.bindOverlay("$location", "LLNET");
    ...
}
```
### cities()

`CollectStatisticsAgent$searchWeatherAgent()` is used to acquire a list of city names.

### collection()
Statistical information for each city specified by an argument is displayed.


## Run the sample program

We will run the sample program

### Procedure for Execution

1. [Add new package], and [Start] of the Collection.jar are carried out from the [Manage Package] screen so as to activate the Agent.
2. Select [Method call] tab in [Manage Agent]
3. Select Node and Peer from the [Method call] screen so as to call the preparation(). *1, *2
    - Select the number less than the number of assigned peer because the preparation creates one `WeatherAgent` in each peer.
4. Select Node and Peer from the [Method call] screen so as to call the cities(). *2
5. Select Node and Peer from the [Method call] screen so as to call the collection(). *2


The value of Args of collection(), the numbers of the city names resulting from the cities() are separated by commas, and the distance in the latitude and the longitude is designated after colons.


(example)
The results of `cities()`

```
[1]sapporo
[2]morioka
[3]osaka
[4]kyoto
[5]nagoya
[6]shizuoka
[7]yokohama
[8]tokyo
[9]akita
[10]sendai
```

Display of statistical information within a radius 1.0 for  [1]sapporo [2]morioka [3]osaka [4]kyoto
Argeumts : "1,2,3,4:1.0"
Output

```
Radius is 1.0
osaka and environs: 8.0
sapporo and environs: -4.0
morioka and environs: -1.0
kyoto and environs: 8.0
```

*1 Execute preparation() only once whenever the jar file is loaded.
*2 The same results can be gained when either Node or Peer is selected from the [Call Agent] screen.
