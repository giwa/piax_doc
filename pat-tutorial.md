このチュートリアルでは，SDK付属のCollectionを例にして，PIAX上で動作するエージェントを作ります．

Collectionは，LL-Net上のセンサ機能を持っているエージェントを検索し，その エージェントからデータ(気温データ)を集め，統計情報(平均気温)を作ります． ソースファイルはSDKのCollection/srcにあります．

チュートリアルを通して，以下を学べるようになっています．

エージェントの作り方
移動しないエージェントと，ピア間を移動できるエージェントを作ります．
LL-Net を使ったエージェントの検索方法
エージェントを移動させる方法
RPC 呼び出しの使い方
作るエージェントは 4つで，WeatherAgent，CollectStatisticsAgent， MakeStatisticsAgent，MainAgentです．

それぞれのエージェントの概要について説明します．

WeatherAgent

気象情報を収集したり，計測したりするエージェントです． ただし，このチュートリアルでは，予め用意しておいた気温データを使います．

MakeStatisticsAgent

ユーザが選択したピアに移動し，そのピアの周辺にあるWeatherAgentから気 温データを集め，平均値を計算します．

CollectStatisticsAgent

WeatherAgentを探し，MakeStatisticsAgentを生成して移動させ， MakeStatisticsAgentのメソッドを呼び出して気温の平均値を算出します．

MainAgent

PIAXではmain()関数が呼ばれません．そのため，main()関数の役割をする MainAgentクラスを作成し使います．

WeatherAgentを作る

以下の内容のWeatherAgent.javaというファイルを作ります．


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
これはWeatherAgentのクラス定義です． 以下ではクラス定義を説明します．

PIAXのエージェントは，Agentクラスを継承して作ります．

また，PIAXには，あるクラスについて外部から呼び出せるメソッドを明示的に示すためにインタフェースファイルを作成するという約束があります． インタフェースファイルWeatherAgentIf.javaは後述します．

getWeatherCalleeId()メソッドはCollectStatisticsAgentがWeatherAgentを探すときに呼び出すメソッドで，自分のCalleeIdを返します． CalleeIdクラスは他のピア上にあるオブジェクトを参照するためのクラスで， Serializableを実装しています。

このサンプルでは作成しませんが， エージェントのメソッドとしてユーザ独自のクラスを返すメソッドを作成する場合は， その独自クラスにSerializableを実装する必要があります．

getTemperature()はMakeStatisticsAgentが呼び出すメソッドです．

データは予め意しておいたものを使います． 予め用意しているデータはCollection/src/agents/PointInfo.javaにあります． 日本各地の緯度経度情報とある日の気温データです． 緯度経度から，地名を取得したり気温データをを取得したりするために使います．

WeatherAgentの説明は以上で終わりです． 次に，WeatherAgentIf.javaを作ります．

WeatherAgentが持っているメソッドはsetCity()， getCitry()，getTemperature()， および　getWeatherCalleeIdで， いずれも外部から呼び出されるメソッドです．

以下の内容のWeatherAgentIf.javaというファイルを作ります．


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
エージェントのインタフェースは，AgentIfを継承して作ります．

ポイント

PIAXエージェントはAgentクラスを継承する
PIAXエージェントはAgentIfを継承したインタフェースを実装する
エージェントのメソッドの戻り値はSerializableを実装する
MakeStatisticsAgentを作る

以下の内容のMakeStatisticsAgent.javaというファイルを作ります．


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
     * This method is called by CollectStatisticsAgent. I wasn't needed any
     * more.
     */
    public void bye() {
        destroy();
    }

}
MakeStatisticsAgentのクラス定義は以上です． 以下ではクラス定義を説明します．

WeatherAgentと異なる点は，継承するクラスがAgentクラスではなく， MobileAgentクラスであることです． MobileAgentクラスを継承したエージェントはピア間を移動できます． MobileAgentクラスはAgentクラスを継承しています．

メンバ変数collectorAid，init()メソッド，onArrival()メソッド

メンバ変数collectorCidはCollectStatisticsAgentのCalleeIdを保存しておくための変数です．init()メソッドで設定されます． CollectStatisticsAgentは，MakeStatisticsAgentを作ると，すぐにinit() を呼び出して，自分のCalleeIdを設定します．

collectorCidはMakeStatisticsAgentがCollectStatisticsAgentのメソッドを呼び出すときに使います． MakeStatisticsAgentは以下のように動作します． MakeStatisticsAgentはCollectStatisticsAgentのあるピアから指定されたピアに移動します． 移動が終了すると，onArrival()メソッドが自動的に実行されます． onArrival()はMobileAgentから継承したメソッドで， エージェントが目的ピアに到着したら自動的に呼び出されるメソッドです． onArrival()メソッドの中から， RPCで CollectStatisticsAgentのarrived()メソッドを呼び出します． RPCを行うには，getStub()を使ってRPC用のStubを作成します． Stubを作成するときに，collectorCidを使います．

doMake()メソッド

doMake()メソッドはCollectStatisticsAgentから呼び出されます． MakeStatisticsAgentがいるピアのLL-Net上の場所(Location)から半径 r度内にあるピアに対しディスカバリーコールを行います． ディスカバリーコールを行う場合は， getDCStub()を使ってディスカバリーコール用のStubを作成します．

getDCStub()の定義は以下のとおりです．


    public <S extends RPCIf> S getDCStub(String queryCond, Class<S> clz, RPCMode rcpMode);
  
queryCondによって使われるオーバーレイが決まります． LL-Net で検索するには，以下のような文字列を指定します．

    $location in rect(x, y, w, h)
    $location in circle(x, y, r)
    ここで，x は経度，y は緯度，wは経度の幅，hは緯度の幅，rは経緯度の半径です．
clzには，対象エージェントが実装しているインタフェースを指定します． rpcModeには，Onewayかどうかを指定します． Stub.method()のように対象エージェントのメソッドを呼び出します．

argsはメソッドの引数に合わせて指定します． getTemperature()メソッドには引数がないので指定しません．

bye()メソッド

bye()メソッドもCollectStatisticsAgentから呼び出されます． 不要になったMakeStatisticsAgentを破棄するために使います．

MakeStatisticsAgentの説明はこれで終わりです． 以下ではMakeStatisticsAgentのインタフェースファイルを作ります． MakeStatisticsAgentが持っているメソッドはinit()，doMake()，bye()，getCalleeIdで， いずれも外部から呼び出されるメソッドです． 以下の内容のMakeStatisticsAgentIf.javaというファイルを作ります．


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
ポイント

ピア間を移動する PIAXエージェントはMobileAgentクラスを継承する
RCP呼び出しを行うには，CalleeIdを指定してStubを作成する
CollectStatisticsAgentを作る

以下は，CollectStatisticsAgent.javaの一部です． CollectStatisticsAgent.javaは長いので，全体はソースファイルを参照してくださnnxsい．省略部分は "..." で示します．


package agents;

import ...

public class CollectStatisticsAgent extends Agent implements
        CollectStatisticsAgentIf {

    /* 日本を内包するRANGE */
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

            /* MakeStatisticsAgent 作成 */
            AgentId aid = getHome().createAgent(MakeStatisticsAgent.class);
            /* MakeStatistics onArrival 用に自身(CollectStatisticsAgen)のCalleeIdを取得 */
            CalleeId mcId = getHome().getCalleeId(aid);
            /* MakeStatistics init を実行 */
            getStub(MakeStatisticsAgentIf.class, mcId).init(
            /* my CalleeId for Arrived */
            getHome().getCalleeId(getId()));
            /* WeatherAgent CalleeId から Endpoint(PeerId) の情報を取得 */
            PeerId pid = (PeerId) wcId.getPeerRef();
            ...

            /* travelAgent で移動元と移動先が同じ場合例外が発生するが無視する。 */
            if (!getHome().getPeerId().equals(pid)) {
                /* Weather Agent の移動 */
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
以下ではCollectStatisticsAgentのクラス定義を説明します．

CollectStatitiscsAgentは移動しないので，MobileAgentクラスではなく， Agentクラスを継承します． インタフェースファイルCollectStatisticsIf.javaについては後述します．

メンバ変数がいくつかありますが，変数の説明はメソッドの説明の中で行います．

searchWeatherAgent()メソッド

LL-Net上のWeatherAgentを探し， メンバ変数calleeIdMapに登録するメソッドです． calleeIdMapに登録された情報は， doCollect()メソッドが使います．

LL-Net上で検索するためにgetDCStub()の第1引数を次のように指定しています．


"$location in rect(122.0, 23.0, 27.0, 24.0)"
ここで指定した数字は，およそ日本列島全体を囲む領域になっています．

arrived()メソッド

arrived()は，MakeStatisticsAgentの説明で出てきました． MakeStatisticsAgentが移動したことをCollectStatisticsAgentに通知するために呼び出すメソッドです． 複数のMakeStatisticsAgentから同時に呼び出される可能性があるので， synchronizedで囲んでいます．

doCollect()メソッド

引数のcollectPointsは統計情報を作る場所のリストです． collectPointsの数だけMakeStatisticsAgentを作り， init()メソッドを呼び出して自分のAgentIdを設定した後， travelAgent()でcollectPointsのピアまで移動させます．

travelAgent()の引数はAgentIdとPeerIdです． PeerIdはsearchAgent() メソッドで収集したcalleeIdMapから得ます． その後，全部のMakeStatisticsAgentが目的のピアに到着するのを待ちます．

全部のMakeStatisticsAgentが目的ピアに到着したら， RPCで doMake()メソッドを呼び出して， 統計情報を取得します． この呼び出しは，AgentPeerのsubmitにより別スレッドに送信されたタスクで行われます．

これは，統計情報の作成に時間がかかると想定し， 複数のスレッドで各エージェントに同時に作業させることを目的としています． (これはサンプルなのですぐに戻ってきますが，実際にはセンサからの入力を待つかもしれません）

その後，AgentPeerのawaitTermination()メソッドを呼び出して， doMake()の結果が揃うのを待ちます． ただし，すべての呼び出しが必ず結果を返す保証はないため， このサンプルでは最大300秒待つようにしています．

CollectStatisticsAgentの説明は以上で終わりです． 以下でCollectStatisticsAgentのインタフェースファイルを作ります． CollectStatisticsAgentにはメソッドが 10個ありますが， 10個のうち外部から呼び出されるのは， doCollect()とarrived()とgetCalleeId()の 3個です． 以下の内容のCollectStatisticsAgentIf.javaというファイルを作ります．


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
ポイント

複数の RPC 呼び出しを並列実行するには AgentPeer の submit を使う
RPC 呼び出しがすべて成功することを期待してはならない
同時に呼び出される可能性のあるメソッドは排他制御を行う
MainAgentを作る

CollectStatisticsAgentとWeatherAgentを用意し， PIAXで実行するためにMainAgentクラスを作ります．

以下は，MainAgent.javaの一部です． MainAgent.java全体は大きいので，詳細はソースファイルを参照してください． 省略部分は "..." で示します．


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
      
        /* ディスカバリーコール 範囲内の MainAgentを探す */
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

        /* WeatherAgent を生成する */
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
            // LLNETをプラグイン
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
PIAXでは，*.jarファイルをloadすると，各ピアでMainAgentのインスタンスを作成します． メインの処理は，MainAgentのメソッドをcallすることで，キックすることができます． MainAgentは，必ずorg.piax.agent.Agentクラスを継承し，デフォルトパッケージに所属するようにします．

init()

MainAgentを作成すると，このメソッドが呼ばれます． まだピアがオンライン状態ではないことに注意してください． Collectionでは，ロケーションの設定を行なっています．
preparation()

preparation()では，ピア毎に，WeatherAgentを 1つ作成します． そのため，現存しているMainAgentのインスタンスのCalleeIDを取得します． ピアごとのCalleeIdの取得には，getCalleeId()メソッドを使用します． 取得したCalleeIdを使用してcreateWeatherAgent()を呼びWeatherAgentを作成します．
対象となるピアの検索

MainAgentのgetCalleeId()を呼ぶピアの検索には， ディスカバリーコールを使用します． そのため，init()の処理でローケンションを設定します．
ロケーション設定時の注意点

ロケーションの設定にはLL-Netオーバーレイのロードが必要です． setLocation()やsetAttrib()でロケーション情報を設定するまえに， 以下のようにオーバレイをロードしてください． Collectionでは，init()で行っています．

public void init(TestbedInfo ti) throws Exception {
    ...
    AgentPeer apeer = getAgentPeer();
    apeer.declareAttrib("$location", Location.class);
    apeer.bindOverlay("$location", "LLNET");
    ...
}
cities()

CollectStatisticsAgent$searchWeatherAgent()を使用して都市名のリストを取得します．
collection()

引数で指定した．都市毎の統計情報を表示します．
使ってみる

作ったサンプルを動かします．

実行手順

「パッケージ操作」ページから，Collection.jarの[登録]，[起動]を行い， パッケージのステータスを[動作中]にする．
「エージェント操作」ページの「メソッド呼出」タブを選択する．
ノード，ピアを選択し，メソッドにpreparationを指定して， 「呼出実行」ボタンを押す．*1, *2
preparationは各ピア上にWeatherAgentを1つ作成するので， 引数には現在割り当てられているピアの数以下の値を指定する．
ノード，ピアを選択し，メソッドにcitiesを指定して， 「呼出実行」ボタンを押す．*2
ノード，ピアを選択し，メソッドにcollection指定して， 「呼出実行」ボタンを押す． *2
引数には，citiesの結果の都市名の番号をカンマ区切りにしコロンの後に緯度経度の距離を指定します．
(例)
citiesの結果

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
[1]sapporo [2]morioka [3]osaka [4]kyoto の半径1.0の中の統計情報の表示
引数 : "1,2,3,4:1.0"
出力結果
Radius is 1.0
osaka and environs: 8.0
sapporo and environs: -4.0
morioka and environs: -1.0
kyoto and environs: 8.0

*1 preparationは， パッケージの起動毎に一回だけ実行してください．
*2 ノード，ピアはどれを選択しても同様の結果になります．
