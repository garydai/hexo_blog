---
date: 2019-6-15
layout: default

title: zookeeper

---

# zookeeper

创建唯一顺序id怎么做到的

高性能、高可用、写操作顺序性

## 常用命令

ls /

create /test 1

set /test 2

get /test

delete

rmr

## 节点
1. 持久节点

creste 

2. 临时节点

create -e

3. 持久顺序节点

create -s

4. 临时顺序节点

create -s -e


## 存储
存放在内存

树形存储结构

## 集群
leader

follower

learner

ack过半

zab协议

一个节点默认最大数据量是1M
## 端口
同步端口

选举端口

## 持久化


![](https://github.com/garydai/garydai.github.com/raw/master/_posts/pic/zookeeper.png)


![](https://github.com/garydai/garydai.github.com/raw/master/_posts/pic/zookeeper2.png)

## 服务端

### 入口

org.apache.zookeeper.server.quorum.QuorumPeerMain#main

org.apache.zookeeper.server.quorum.QuorumPeerMain#initializeAndRun

```java
protected void initializeAndRun(String[] args)
    throws ConfigException, IOException, AdminServerException
{
    QuorumPeerConfig config = new QuorumPeerConfig();
    if (args.length == 1) {
        config.parse(args[0]);
    }
		
  	// 清理线程
    // Start and schedule the the purge task
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
            .getDataDir(), config.getDataLogDir(), config
            .getSnapRetainCount(), config.getPurgeInterval());
    purgeMgr.start();

    if (args.length == 1 && config.isDistributed()) {
      	// 运行服务
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running "
                + " in standalone mode");
        // there is only server in the quorum -- run as standalone
        ZooKeeperServerMain.main(args);
    }
}
```

org.apache.zookeeper.server.quorum.QuorumPeerMain#runFromConfig

```java
public void runFromConfig(QuorumPeerConfig config)
        throws IOException, AdminServerException
{
  
  
    if (config.getClientPortAddress() != null) {
      // 实例化nio server factory，NIOServerCnxnFactory
      cnxnFactory = ServerCnxnFactory.createFactory();
      // 创建io相关线程
      cnxnFactory.configure(config.getClientPortAddress(),
                            config.getMaxClientCnxns(),
                            config.getClientPortListenBacklog(), false);      
    }
  
  
    // 启动io各个线程
    quorumPeer.start();
}
```

org.apache.zookeeper.server.NIOServerCnxnFactory#configure

```java
public void configure(InetSocketAddress addr, int maxcc, int backlog, boolean secure) throws IOException {
    if (secure) {
        throw new UnsupportedOperationException("SSL isn't supported in NIOServerCnxn");
    }
    configureSaslLogin();

    maxClientCnxns = maxcc;
    sessionlessCnxnTimeout = Integer.getInteger(
        ZOOKEEPER_NIO_SESSIONLESS_CNXN_TIMEOUT, 10000);
    // We also use the sessionlessCnxnTimeout as expiring interval for
    // cnxnExpiryQueue. These don't need to be the same, but the expiring
    // interval passed into the ExpiryQueue() constructor below should be
    // less than or equal to the timeout.
    cnxnExpiryQueue =
        new ExpiryQueue<NIOServerCnxn>(sessionlessCnxnTimeout);
    // 关闭过期的连接的线程
    expirerThread = new ConnectionExpirerThread();

    int numCores = Runtime.getRuntime().availableProcessors();
    // 32 cores sweet spot seems to be 4 selector threads
    // selector线程 sqrt(coreSize / 2)
    numSelectorThreads = Integer.getInteger(
        ZOOKEEPER_NIO_NUM_SELECTOR_THREADS,
        Math.max((int) Math.sqrt((float) numCores/2), 1));
    if (numSelectorThreads < 1) {
        throw new IOException("numSelectorThreads must be at least 1");
    }
		// 工作线程coreSize * 2
    numWorkerThreads = Integer.getInteger(
        ZOOKEEPER_NIO_NUM_WORKER_THREADS, 2 * numCores);
    workerShutdownTimeoutMS = Long.getLong(
        ZOOKEEPER_NIO_SHUTDOWN_TIMEOUT, 5000);

    LOG.info("Configuring NIO connection handler with "
             + (sessionlessCnxnTimeout/1000) + "s sessionless connection"
             + " timeout, " + numSelectorThreads + " selector thread(s), "
             + (numWorkerThreads > 0 ? numWorkerThreads : "no")
             + " worker threads, and "
             + (directBufferBytes == 0 ? "gathered writes." :
                ("" + (directBufferBytes/1024) + " kB direct buffers.")));
    for(int i=0; i<numSelectorThreads; ++i) {
        selectorThreads.add(new SelectorThread(i));
    }

    listenBacklog = backlog;
    this.ss = ServerSocketChannel.open();
    ss.socket().setReuseAddress(true);
    LOG.info("binding to port " + addr);
    if (listenBacklog == -1) {
      ss.socket().bind(addr);
    } else {
      ss.socket().bind(addr, listenBacklog);
    }
    ss.configureBlocking(false);
    acceptThread = new AcceptThread(ss, addr, selectorThreads);
}
```

```java
public void startup(ZooKeeperServer zks, boolean startServer)
        throws IOException, InterruptedException {
    start();
    setZooKeeperServer(zks);
    if (startServer) {
        zks.startdata();
        zks.startup();
    }
}
```

org.apache.zookeeper.server.quorum.QuorumPeer#QuorumPeer()

```java
public QuorumPeer() throws SaslException {
    super("QuorumPeer");
    quorumStats = new QuorumStats(this);
    jmxRemotePeerBean = new HashMap<Long, RemotePeerBean>();
    adminServer = AdminServerFactory.createAdminServer();
    x509Util = createX509Util();
    initialize();
}
```

org.apache.zookeeper.server.quorum.QuorumPeer#start

```java
public synchronized void start() {
    if (!getView().containsKey(myid)) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
     }
    // 加载快照和增量log到内存dataTree
    loadDataBase();
    // 启动io线程
    startServerCnxnFactory();
    try {
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
        System.out.println(e);
    }
    // 领导者选举准备工作，创建io线程
    startLeaderElection();
    startJvmPauseMonitor();
    // 开启选举
    super.start();
}
```

```java
private void startServerCnxnFactory() {
    if (cnxnFactory != null) {
        cnxnFactory.start();
    }
    if (secureCnxnFactory != null) {
        secureCnxnFactory.start();
    }
}
```

```java
public void start() {
    stopped = false;
    if (workerPool == null) {
        workerPool = new WorkerService(
            "NIOWorker", numWorkerThreads, false);
    }
    for(SelectorThread thread : selectorThreads) {
        if (thread.getState() == Thread.State.NEW) {
            thread.start();
        }
    }
    // ensure thread is started once and only once
    if (acceptThread.getState() == Thread.State.NEW) {
        acceptThread.start();
    }
    if (expirerThread.getState() == Thread.State.NEW) {
        expirerThread.start();
    }
}
```

### 选举

准备工作，开启io线程

org.apache.zookeeper.server.quorum.QuorumPeer#startLeaderElection

```java
synchronized public void startLeaderElection() {
    try {
        if (getPeerState() == ServerState.LOOKING) {
          	// 当前投自己
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
        }
    } catch(IOException e) {
        RuntimeException re = new RuntimeException(e.getMessage());
        re.setStackTrace(e.getStackTrace());
        throw re;
    }

    this.electionAlg = createElectionAlgorithm(electionType);
}
```

```java
protected Election createElectionAlgorithm(int electionAlgorithm){
    Election le=null;

    //TODO: use a factory rather than a switch
    switch (electionAlgorithm) {
    case 1:
        le = new AuthFastLeaderElection(this);
        break;
    case 2:
        le = new AuthFastLeaderElection(this, true);
        break;
    case 3:
        // 默认用FastLeaderElection
        QuorumCnxManager qcm = createCnxnManager();
        QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
        if (oldQcm != null) {
            LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
            oldQcm.halt();
        }
        QuorumCnxManager.Listener listener = qcm.listener;
        if(listener != null){
            listener.start();
            FastLeaderElection fle = new FastLeaderElection(this, qcm);
            fle.start();
            le = fle;
        } else {
            LOG.error("Null listener when initializing cnx manager");
        }
        break;
    default:
        assert false;
    }
    return le;
}
```

开启选举

org.apache.zookeeper.server.quorum.QuorumPeer#run

```java
public void run() {
    updateThreadName();

    LOG.debug("Starting quorum peer");
    try {
        jmxQuorumBean = new QuorumBean(this);
        MBeanRegistry.getInstance().register(jmxQuorumBean, null);
        for(QuorumServer s: getView().values()){
            ZKMBeanInfo p;
            if (getId() == s.id) {
                p = jmxLocalPeerBean = new LocalPeerBean(this);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                    jmxLocalPeerBean = null;
                }
            } else {
                RemotePeerBean rBean = new RemotePeerBean(this, s);
                try {
                    MBeanRegistry.getInstance().register(rBean, jmxQuorumBean);
                    jmxRemotePeerBean.put(s.id, rBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                }
            }
        }
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        jmxQuorumBean = null;
    }

    try {
        /*
         * Main loop
         */
        while (running) {
            switch (getPeerState()) {
            case LOOKING:
                LOG.info("LOOKING");
                ServerMetrics.getMetrics().LOOKING_COUNT.add(1);

                if (Boolean.getBoolean("readonlymode.enabled")) {
                    LOG.info("Attempting to start ReadOnlyZooKeeperServer");

                    // Create read-only server but don't start it immediately
                    final ReadOnlyZooKeeperServer roZk =
                        new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb);

                    // Instead of starting roZk immediately, wait some grace
                    // period before we decide we're partitioned.
                    //
                    // Thread is used here because otherwise it would require
                    // changes in each of election strategy classes which is
                    // unnecessary code coupling.
                    Thread roZkMgr = new Thread() {
                        public void run() {
                            try {
                                // lower-bound grace period to 2 secs
                                sleep(Math.max(2000, tickTime));
                                if (ServerState.LOOKING.equals(getPeerState())) {
                                    roZk.startup();
                                }
                            } catch (InterruptedException e) {
                                LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                            } catch (Exception e) {
                                LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                            }
                        }
                    };
                    try {
                        roZkMgr.start();
                        reconfigFlagClear();
                        if (shuttingDownLE) {
                            shuttingDownLE = false;
                            startLeaderElection();
                        }
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                        setPeerState(ServerState.LOOKING);
                    } finally {
                        // If the thread is in the the grace period, interrupt
                        // to come out of waiting.
                        roZkMgr.interrupt();
                        roZk.shutdown();
                    }
                } else {
                    try {
                       reconfigFlagClear();
                        if (shuttingDownLE) {
                           shuttingDownLE = false;
                           startLeaderElection();
                           }
                        // 开始选举
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                        setPeerState(ServerState.LOOKING);
                    }                        
                }
                break;
            case OBSERVING:
                try {
                    LOG.info("OBSERVING");
                    setObserver(makeObserver(logFactory));
                    observer.observeLeader();
                } catch (Exception e) {
                    LOG.warn("Unexpected exception",e );
                } finally {
                    observer.shutdown();
                    setObserver(null);
                    updateServerState();

                    // Add delay jitter before we switch to LOOKING
                    // state to reduce the load of ObserverMaster
                    if (isRunning()) {
                        Observer.waitForObserverElectionDelay();
                    }
                }
                break;
            case FOLLOWING:
                try {
                   LOG.info("FOLLOWING");
                    setFollower(makeFollower(logFactory));
                    follower.followLeader();
                } catch (Exception e) {
                   LOG.warn("Unexpected exception",e);
                } finally {
                   follower.shutdown();
                   setFollower(null);
                   updateServerState();
                }
                break;
            case LEADING:
                LOG.info("LEADING");
                try {
                    setLeader(makeLeader(logFactory));
                    leader.lead();
                    setLeader(null);
                } catch (Exception e) {
                    LOG.warn("Unexpected exception",e);
                } finally {
                    if (leader != null) {
                        leader.shutdown("Forcing shutdown");
                        setLeader(null);
                    }
                    updateServerState();
                }
                break;
            }
            start_fle = Time.currentElapsedTime();
        }
    } finally {
        LOG.warn("QuorumPeer main thread exited");
        MBeanRegistry instance = MBeanRegistry.getInstance();
        instance.unregister(jmxQuorumBean);
        instance.unregister(jmxLocalPeerBean);

        for (RemotePeerBean remotePeerBean : jmxRemotePeerBean.values()) {
            instance.unregister(remotePeerBean);
        }

        jmxQuorumBean = null;
        jmxLocalPeerBean = null;
        jmxRemotePeerBean = null;
    }
}
```

当当前服务的状态是LOOKING，则开启新一轮选举

org.apache.zookeeper.server.quorum.FastLeaderElection#lookForLeader

```java
public Vote lookForLeader() throws InterruptedException {
    try {
        self.jmxLeaderElectionBean = new LeaderElectionBean();
        MBeanRegistry.getInstance().register(
                self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        self.jmxLeaderElectionBean = null;
    }
    if (self.start_fle == 0) {
       self.start_fle = Time.currentElapsedTime();
    }
    try {
        Map<Long, Vote> recvset = new HashMap<Long, Vote>();

        Map<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = minNotificationInterval;

        synchronized(this){
            logicalclock.incrementAndGet();
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        LOG.info("New election. My id =  " + self.getId() +
                ", proposed zxid=0x" + Long.toHexString(proposedZxid));
      	// 发送选举自己的广播通知
        sendNotifications();

        SyncedLearnerTracker voteSet;

        /*
         * Loop in which we exchange notifications until we find a leader
         */

        while ((self.getPeerState() == ServerState.LOOKING) &&
                (!stop)){
            /*
             * Remove next notification from queue, times out after 2 times
             * the termination time
             */
          	// 接收选举通知
            Notification n = recvqueue.poll(notTimeout,
                    TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             */
            if(n == null){
                if(manager.haveDelivered()){
                    sendNotifications();
                } else {
                    manager.connectAll();
                }

                /*
                 * Exponential backoff
                 */
                int tmpTimeOut = notTimeout*2;
                notTimeout = (tmpTimeOut < maxNotificationInterval?
                        tmpTimeOut : maxNotificationInterval);
                LOG.info("Notification time out: " + notTimeout);
            }
            else if (validVoter(n.sid) && validVoter(n.leader)) {
                /*
                 * Only proceed if the vote comes from a replica in the current or next
                 * voting view for a replica in the current or next voting view.
                 */
                switch (n.state) {
                case LOOKING:
                    if (getInitLastLoggedZxid() == -1) {
                        LOG.debug("Ignoring notification as our zxid is -1");
                        break;
                    }
                    if (n.zxid == -1) {
                        LOG.debug("Ignoring notification from member with -1 zxid" + n.sid);
                        break;
                    }
                    // If notification > current, replace and send messages out
                    if (n.electionEpoch > logicalclock.get()) {
                        logicalclock.set(n.electionEpoch);
                        recvset.clear();
                        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else {
                            updateProposal(getInitId(),
                                    getInitLastLoggedZxid(),
                                    getPeerEpoch());
                        }
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) {
                        if(LOG.isDebugEnabled()){
                            LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                    + Long.toHexString(n.electionEpoch)
                                    + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                        }
                        break;
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                            proposedLeader, proposedZxid, proposedEpoch)) {
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        sendNotifications();
                    }

                    if(LOG.isDebugEnabled()){
                        LOG.debug("Adding vote: from=" + n.sid +
                                ", proposed leader=" + n.leader +
                                ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                                ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                    }

                    // don't care about the version if it's in LOOKING state
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                    voteSet = getVoteTracker(
                            recvset, new Vote(proposedLeader, proposedZxid,
                                    logicalclock.get(), proposedEpoch));

                    if (voteSet.hasAllQuorums()) {

                        // Verify if there is any change in the proposed leader
                        while((n = recvqueue.poll(finalizeWait,
                                TimeUnit.MILLISECONDS)) != null){
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    proposedLeader, proposedZxid, proposedEpoch)){
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                        if (n == null) {
                            setPeerState(proposedLeader, voteSet);
                            Vote endVote = new Vote(proposedLeader,
                                    proposedZxid, logicalclock.get(), 
                                    proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    LOG.debug("Notification from observer: " + n.sid);
                    break;
                case FOLLOWING:
                case LEADING:
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    if(n.electionEpoch == logicalclock.get()){
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        voteSet = getVoteTracker(recvset, new Vote(n.version, 
                                  n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        if (voteSet.hasAllQuorums() && 
                                checkLeader(outofelection, n.leader, n.electionEpoch)) {
                            setPeerState(n.leader, voteSet);
                            Vote endVote = new Vote(n.leader, 
                                    n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble, verify that
                     * a majority are following the same leader.
                     */
                    outofelection.put(n.sid, new Vote(n.version, n.leader,
                            n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                    voteSet = getVoteTracker(outofelection, new Vote(n.version, 
                            n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

                    if (voteSet.hasAllQuorums() &&
                            checkLeader(outofelection, n.leader, n.electionEpoch)) {
                        synchronized(this){
                            logicalclock.set(n.electionEpoch);
                            setPeerState(n.leader, voteSet);
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, 
                                n.electionEpoch, n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecoginized: " + n.state
                          + " (n.state), " + n.sid + " (n.sid)");
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
        try {
            if(self.jmxLeaderElectionBean != null){
                MBeanRegistry.getInstance().unregister(
                        self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
        LOG.debug("Number of connection processing threads: {}",
                manager.getConnectionThreadCount());
    }
}
```

### IO模型

AcceptThread

SelectorThread

workerThread

![image-20200121163533172](/Users/daitechang/Documents/hexo_blog/source/_posts/pic/image-20200121163533172.png)

## 处理请求

org.apache.zookeeper.server.NIOServerCnxnFactory.IOWorkRequest#doWork

```java
public void doWork() throws InterruptedException {
    if (!key.isValid()) {
        selectorThread.cleanupSelectionKey(key);
        return;
    }

    if (key.isReadable() || key.isWritable()) {
        cnxn.doIO(key);

        // Check if we shutdown or doIO() closed this connection
        if (stopped) {
            cnxn.close();
            return;
        }
        if (!key.isValid()) {
            selectorThread.cleanupSelectionKey(key);
            return;
        }
        touchCnxn(cnxn);
    }

    // Mark this connection as once again ready for selection
    cnxn.enableSelectable();
    // Push an update request on the queue to resume selecting
    // on the current set of interest ops, which may have changed
    // as a result of the I/O operations we just performed.
    if (!selectorThread.addInterestOpsUpdateRequest(key)) {
        cnxn.close();
    }
}
```

```java
void doIO(SelectionKey k) throws InterruptedException {
    try {
        if (isSocketOpen() == false) {
            LOG.warn("trying to do i/o on a null socket for session:0x"
                     + Long.toHexString(sessionId));

            return;
        }
        if (k.isReadable()) {
            int rc = sock.read(incomingBuffer);
            if (rc < 0) {
                ServerMetrics.getMetrics().CONNECTION_DROP_COUNT.add(1);
                throw new EndOfStreamException(
                        "Unable to read additional data from client sessionid 0x"
                        + Long.toHexString(sessionId)
                        + ", likely client has closed socket");
            }
            if (incomingBuffer.remaining() == 0) {
                boolean isPayload;
                if (incomingBuffer == lenBuffer) { // start of next request
                    incomingBuffer.flip();
                    isPayload = readLength(k);
                    incomingBuffer.clear();
                } else {
                    // continuation
                    isPayload = true;
                }
                if (isPayload) { // not the case for 4letterword
                    readPayload();
                }
                else {
                    // four letter words take care
                    // need not do anything else
                    return;
                }
            }
        }
        if (k.isWritable()) {
            handleWrite(k);

            if (!initialized && !getReadInterest() && !getWriteInterest()) {
                throw new CloseRequestException("responded to info probe");
            }
        }
    } catch (CancelledKeyException e) {
        ...
    }
}
```

org.apache.zookeeper.server.NIOServerCnxn#readPayload

org.apache.zookeeper.server.NIOServerCnxn#readRequest

org.apache.zookeeper.server.ZooKeeperServer#processPacket

```java
public void processPacket(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    // We have the request, now process and setup for next
    InputStream bais = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
    RequestHeader h = new RequestHeader();
    h.deserialize(bia, "header");

    // Need to increase the outstanding request count first, otherwise
    // there might be a race condition that it enabled recv after
    // processing request and then disabled when check throttling.
    //
    // Be aware that we're actually checking the global outstanding
    // request before this request.
    //
    // It's fine if the IOException thrown before we decrease the count
    // in cnxn, since it will close the cnxn anyway.
    cnxn.incrOutstandingAndCheckThrottle(h);

    // Through the magic of byte buffers, txn will not be
    // pointing
    // to the start of the txn
    incomingBuffer = incomingBuffer.slice();
    if (h.getType() == OpCode.auth) {
        LOG.info("got auth packet " + cnxn.getRemoteSocketAddress());
        AuthPacket authPacket = new AuthPacket();
        ByteBufferInputStream.byteBuffer2Record(incomingBuffer, authPacket);
        String scheme = authPacket.getScheme();
        ServerAuthenticationProvider ap = ProviderRegistry.getServerProvider(scheme);
        Code authReturn = KeeperException.Code.AUTHFAILED;
        if(ap != null) {
            try {
                // handleAuthentication may close the connection, to allow the client to choose
                // a different server to connect to.
                authReturn = ap.handleAuthentication(new ServerAuthenticationProvider.ServerObjs(this, cnxn), authPacket.getAuth());
            } catch(RuntimeException e) {
                LOG.warn("Caught runtime exception from AuthenticationProvider: " + scheme + " due to " + e);
                authReturn = KeeperException.Code.AUTHFAILED;
            }
        }
        if (authReturn == KeeperException.Code.OK) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("Authentication succeeded for scheme: " + scheme);
            }
            LOG.info("auth success " + cnxn.getRemoteSocketAddress());
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0,
                    KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh, null, null);
        } else {
            if (ap == null) {
                LOG.warn("No authentication provider for scheme: "
                        + scheme + " has "
                        + ProviderRegistry.listProviders());
            } else {
                LOG.warn("Authentication failed for scheme: " + scheme);
            }
            // send a response...
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0,
                    KeeperException.Code.AUTHFAILED.intValue());
            cnxn.sendResponse(rh, null, null);
            // ... and close connection
            cnxn.sendBuffer(ServerCnxnFactory.closeConn);
            cnxn.disableRecv();
        }
        return;
    } else if (h.getType() == OpCode.sasl) {
        Record rsp = processSasl(incomingBuffer,cnxn);
        ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.OK.intValue());
        cnxn.sendResponse(rh,rsp, "response"); // not sure about 3rd arg..what is it?
        return;
    } else {
        Request si = new Request(cnxn, cnxn.getSessionId(), h.getXid(),
          h.getType(), incomingBuffer, cnxn.getAuthInfo());
        si.setOwner(ServerCnxn.me);
        // Always treat packet from the client as a possible
        // local request.
        setLocalSessionFlag(si);
        submitRequest(si);
        return;
    }
}
```

链式处理请求

以commitProcessor为例

```java
public CommitProcessor(RequestProcessor nextProcessor, String id,
                       boolean matchSyncs, ZooKeeperServerListener listener) {
    super("CommitProcessor:" + id, listener);
    this.nextProcessor = nextProcessor;
    this.matchSyncs = matchSyncs;
}
```

org.apache.zookeeper.server.ZooKeeperServer#submitRequest

```java
public void submitRequest(Request si) {
    if (firstProcessor == null) {
        synchronized (this) {
            try {
                // Since all requests are passed to the request
                // processor it should wait for setting up the request
                // processor chain. The state will be updated to RUNNING
                // after the setup.
                while (state == State.INITIAL) {
                    wait(1000);
                }
            } catch (InterruptedException e) {
                LOG.warn("Unexpected interruption", e);
            }
            if (firstProcessor == null || state != State.RUNNING) {
                throw new RuntimeException("Not started");
            }
        }
    }
    try {
        touch(si.cnxn);
        boolean validpacket = Request.isValid(si.type);
        if (validpacket) {
            firstProcessor.processRequest(si);
            if (si.cnxn != null) {
                incInProcess();
            }
        } else {
            LOG.warn("Received packet at server of unknown type " + si.type);
            new UnimplementedRequestProcessor().processRequest(si);
        }
    } catch (MissingSessionException e) {
     ...
    }
}
```

```java
public interface RequestProcessor {
    @SuppressWarnings("serial")
    public static class RequestProcessorException extends Exception {
        public RequestProcessorException(String msg, Throwable t) {
            super(msg, t);
        }
    }

    void processRequest(Request request) throws RequestProcessorException;

    void shutdown();
}
```



1. 创建事务日志

2. 快照，database，文件
3. 更新内存，dataTree
4. 返回结果



## 客户端

使用nio连接服务器

开启两个线程
sendThread

eventThread

org.apache.zookeeper.ZooKeeper#ZooKeeper(java.lang.String, int, org.apache.zookeeper.Watcher, boolean, org.apache.zookeeper.client.HostProvider, org.apache.zookeeper.client.ZKClientConfig)

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
        boolean canBeReadOnly, HostProvider aHostProvider,
        ZKClientConfig clientConfig) throws IOException {
    LOG.info("Initiating client connection, connectString=" + connectString
            + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

    if (clientConfig == null) {
        clientConfig = new ZKClientConfig();
    }
    this.clientConfig = clientConfig;
    watchManager = defaultWatchManager();
    watchManager.defaultWatcher = watcher;
    ConnectStringParser connectStringParser = new ConnectStringParser(
            connectString);
    hostProvider = aHostProvider;

  	// 创建sendThread，eventThread
    cnxn = createConnection(connectStringParser.getChrootPath(),
            hostProvider, sessionTimeout, this, watchManager,
            getClientCnxnSocket(), canBeReadOnly);
    // 开启线程
    cnxn.start();
}
```

```java
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
        long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId;
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;

    connectTimeout = sessionTimeout / hostProvider.size();
    readTimeout = sessionTimeout * 2 / 3;
    readOnly = canBeReadOnly;

    sendThread = new SendThread(clientCnxnSocket);
    eventThread = new EventThread();
    this.clientConfig=zooKeeper.getClientConfig();
    initRequestTimeout();
}
```



客户端输入create等命令，zookeeper将命令放入outgoingQueue，sender线程不断从outgoingQueue里取命令，并发送socket给服务端，然后将socket放入pendingqueue里，等待请求返回；同时该线程还回取pendingqueue里的响应

## 数据结构

树形结构，和文件系统的目录结构相似

DataTree

## 角色

leader 读、写、选举

follower 读、选举

observer 读

## 事件监听器

允许用户在指定节点上注册一些watcher

## 消息广播

leader为follower服务器各自分配一个单独的队列，然后将需要广播的事务proposal依次放入队列中，并且根据fifo策略进行消息发送。

每个follower服务器在接收到这个事务之后，首先将其以事务日志的形式写入本地磁盘，成功写入之后发反馈给leader服务器一个ack响应。

当leader服务器收到超过半数follower的ack之后，广播一个commit消息给所有follower，同时自身也会完成事务提交。

follower收到commit之后，完成事务提价

## 崩溃恢复

## 选举策略

zxid事务id最大

zxid相等，则选serverid最大

## server工作状态

LOOKING：当前Server不知道leader是谁，正在搜寻
LEADING：当前Server即为选举出来的leader
FOLLOWING：leader已经选举出来，当前Server与之同步

 ```
Follower主要有四个功能：
1. 向Leader发送请求（PING消息、REQUEST消息、ACK消息、REVALIDATE消息）；
2 .接收Leader消息并进行处理；
3 .接收Client的请求，如果为写请求，发送给Leader进行投票；
4 .返回Client结果。
Follower的消息循环处理如下几种来自Leader的消息：
1 .PING消息： 心跳消息；
2 .PROPOSAL消息：Leader发起的提案，要求Follower投票；
3 .COMMIT消息：服务器端最新一次提案的信息；
4 .UPTODATE消息：表明同步完成；
5 .REVALIDATE消息：根据Leader的REVALIDATE结果，关闭待revalidate的session还是允许其接受消息；
6 .SYNC消息：返回SYNC结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新。
 ```

## redis异同点