
本文主要围绕 Flink 源码中 `flink-streaming-java` 模块。介绍下 StreamGraph 转成 JobGraph 的过程等。

<!-- more -->

StreamGraph 和 JobGraph 都是在 Client 端生成的，也就是说我们可以在 IDE 中通过断点调试观察 StreamGraph 和 JobGraph 的生成过程。

## 前置调用

从 StreamExecutionEnvironment 中的 execute() 方法一直往下跟：
```java
/**
 * Streaming 程序的提交入口
 */
public JobExecutionResult execute() throws Exception {
	return execute(DEFAULT_JOB_NAME);
}

/**
 * 生成 StreamGraph
 */
public JobExecutionResult execute(String jobName) throws Exception {
	Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");

	return execute(getStreamGraph(jobName));
}

/**
 * 生成 JobGraph ，提交任务，并响应 JobListeners
 */
@Internal
public JobExecutionResult execute(StreamGraph streamGraph) throws Exception {
	// 异步执行
	final JobClient jobClient = executeAsync(streamGraph);

	try {
		final JobExecutionResult jobExecutionResult;

		if (configuration.getBoolean(DeploymentOptions.ATTACHED)) {
			jobExecutionResult = jobClient.getJobExecutionResult(userClassloader).get();
		} else {
			jobExecutionResult = new DetachedJobExecutionResult(jobClient.getJobID());
		}

		jobListeners.forEach(jobListener -> jobListener.onJobExecuted(jobExecutionResult, null));

		return jobExecutionResult;
	} catch (Throwable t) {
		jobListeners.forEach(jobListener -> {
			jobListener.onJobExecuted(null, ExceptionUtils.stripExecutionException(t));
		});
		ExceptionUtils.rethrowException(t);

		// never reached, only make javac happy
		return null;
	}
}
```

下面我们详细看看 StreamExecutionEnvironment 中的 executeAsync 方法：
```java
/**
 * 根据 execution.target 配置反射得到 PipelineExecutorFactory，拿出工厂类对应的 PipelineExecutor，执行其 execute 方法
 * execute的主要工作是将 StreamGraph 转成了 JobGraph，并创建相应的 ClusterClient 完成提交任务的操作。
 */
@Internal
public JobClient executeAsync(StreamGraph streamGraph) throws Exception {
	checkNotNull(streamGraph, "StreamGraph cannot be null.");
	checkNotNull(configuration.get(DeploymentOptions.TARGET), "No execution.target specified in your configuration file.");

	// SPI机制
	// 根据flink Configuration中的"execution.target"加载 PipelineExecutorFactory
	// PipelineExecutorFactory 的实现类在flink-clients包或者flink-yarn包里，因此需要在pom.xml中添加此依赖
	final PipelineExecutorFactory executorFactory =
		executorServiceLoader.getExecutorFactory(configuration);

    // 反射出的 PipelineExecutorFactory 类不能为空
	checkNotNull(
		executorFactory,
		"Cannot find compatible factory for specified execution.target (=%s)",
		configuration.get(DeploymentOptions.TARGET));

	// 根据加载到的 PipelineExecutorFactory 工厂类，获取其对应的 PipelineExecutor，
	// 并执行 PipelineExecutor 的 execute() 方法，将 StreamGraph 转成 JobGraph
	CompletableFuture<JobClient> jobClientFuture = executorFactory
		.getExecutor(configuration)
		.execute(streamGraph, configuration);

	// 异步调用的返回结果
	try {
		JobClient jobClient = jobClientFuture.get();
		jobListeners.forEach(jobListener -> jobListener.onJobSubmitted(jobClient, null));
		return jobClient;
	} catch (Throwable t) {
		jobListeners.forEach(jobListener -> jobListener.onJobSubmitted(null, t));
		ExceptionUtils.rethrow(t);

		// make javac happy, this code path will not be reached
		return null;
	}
}
```
executeAsync 有涉及到 PipelineExecutorFactory 和 PipelineExecutor 。 
PipelineExecutorFactory 是通过 SPI ServiceLoader 加载的，我们看下 `flink-clients` 模块的 `META-INF.services` 文件：
![](./img_jobgraph/flink-clients模块的META-INF文件.png)

PipelineExecutorFactory 的实现子类，分别对应着 Flink 的不同部署模式，local、standalone、yarn、kubernets 等：
![](./img_jobgraph/PipelineExecutorFactory子类.png)

这里我们只看下 LocalExecutorFactory 的实现：
```java
@Internal
public class LocalExecutorFactory implements PipelineExecutorFactory {

	/**
	 * execution.target 配置项对应的值为 "local"
	 */
	@Override
	public boolean isCompatibleWith(final Configuration configuration) {
		return LocalExecutor.NAME.equalsIgnoreCase(configuration.get(DeploymentOptions.TARGET));
	}

	/**
	 * 直接 new 一个 LocalExecutor 返回
	 */
	@Override
	public PipelineExecutor getExecutor(final Configuration configuration) {
		return new LocalExecutor();
	}
}
```

PipelineExecutor 的实现子类与 PipelineExecutorFactory 与工厂类一一对应，负责将 StreamGraph 转成 JobGraph，并生成 ClusterClient 执行任务的提交：
![](./img_jobgraph/PipelineExecutor子类.png)

LocalExecutorFactory 对应的 LocalExecutor 实现如下：
```java
@Internal
public class LocalExecutor implements PipelineExecutor {

	public static final String NAME = "local";

	@Override
	public CompletableFuture<JobClient> execute(Pipeline pipeline, Configuration configuration) throws Exception {
		checkNotNull(pipeline);
		checkNotNull(configuration);

		// we only support attached execution with the local executor.
		checkState(configuration.getBoolean(DeploymentOptions.ATTACHED));

		// StreamGraph 转成 JobGraph
		final JobGraph jobGraph = getJobGraph(pipeline, configuration);

		// local 模式，本地启动一个 Mini Cluster
		final MiniCluster miniCluster = startMiniCluster(jobGraph, configuration);
		// 创建 MiniClusterClient ，准备提交任务
		final MiniClusterClient clusterClient = new MiniClusterClient(configuration, miniCluster);
        // 提交任务
		CompletableFuture<JobID> jobIdFuture = clusterClient.submitJob(jobGraph);

		jobIdFuture
				.thenCompose(clusterClient::requestJobResult)
				.thenAccept((jobResult) -> clusterClient.shutDownCluster());

		return jobIdFuture.thenApply(jobID ->
				new ClusterClientJobClientAdapter<>(() -> clusterClient, jobID));
	}

	private JobGraph getJobGraph(Pipeline pipeline, Configuration configuration) {
		...

		// 这里调用 FlinkPipelineTranslationUtil 的 getJobGraph() 方法
		return FlinkPipelineTranslationUtil.getJobGraph(pipeline, configuration, 1);
	}
}
```


回归主题，我们看下 FlinkPipelineTranslationUtil 的 getJobGraph() 方法：
```java
public static JobGraph getJobGraph(
		Pipeline pipeline,
		Configuration optimizerConfiguration,
		int defaultParallelism) {

	// 通过反射得到 FlinkPipelineTranslator 
	FlinkPipelineTranslator pipelineTranslator = getPipelineTranslator(pipeline);

	return pipelineTranslator.translateToJobGraph(pipeline,
			optimizerConfiguration,
			defaultParallelism);
}

private static FlinkPipelineTranslator getPipelineTranslator(Pipeline pipeline) {
	PlanTranslator planToJobGraphTransmogrifier = new PlanTranslator();

	if (planToJobGraphTransmogrifier.canTranslate(pipeline)) {
		return planToJobGraphTransmogrifier;
	}

	FlinkPipelineTranslator streamGraphTranslator = reflectStreamGraphTranslator();

	// 其实就是判断当前的 Pipeline 实例是不是 StreamGraph
	if (!streamGraphTranslator.canTranslate(pipeline)) {
		throw new RuntimeException("Translator " + streamGraphTranslator + " cannot translate "
				+ "the given pipeline " + pipeline + ".");
	}
	return streamGraphTranslator;
}

private static FlinkPipelineTranslator reflectStreamGraphTranslator() {
		
	Class<?> streamGraphTranslatorClass;
	try {
		streamGraphTranslatorClass = Class.forName(
				// 因为这个类在 flink-streaming-java 模块中，FlinkPipelineTranslationUtil 在 flink-clients 模块中，
			    // flink-clients 模块没有引入 flink-streaming-java 模块，所以只能通过反射拿到
				"org.apache.flink.streaming.api.graph.StreamGraphTranslator",
				true,
				FlinkPipelineTranslationUtil.class.getClassLoader());
	} catch (ClassNotFoundException e) {
		throw new RuntimeException("Could not load StreamGraphTranslator.", e);
	}

	FlinkPipelineTranslator streamGraphTranslator;
	try {
		streamGraphTranslator =
				(FlinkPipelineTranslator) streamGraphTranslatorClass.newInstance();
	} catch (InstantiationException | IllegalAccessException e) {
		throw new RuntimeException("Could not instantiate StreamGraphTranslator.", e);
	}
	return streamGraphTranslator;
}
```

接着走到 StreamGraphTranslator 的 translateToJobGraph 方法：
```java
public class StreamGraphTranslator implements FlinkPipelineTranslator {

	/**
	 * 其实就是调用 StreamGraph 自己的 getJobGraph 方法生成 JobGraph
	 */
	@Override
	public JobGraph translateToJobGraph(
			Pipeline pipeline,
			Configuration optimizerConfiguration,
			int defaultParallelism) {
		checkArgument(pipeline instanceof StreamGraph,
				"Given pipeline is not a DataStream StreamGraph.");

		StreamGraph streamGraph = (StreamGraph) pipeline;
		return streamGraph.getJobGraph(null);
	}

	@Override
	public String translateToJSONExecutionPlan(Pipeline pipeline) {
		checkArgument(pipeline instanceof StreamGraph,
				"Given pipeline is not a DataStream StreamGraph.");

		StreamGraph streamGraph = (StreamGraph) pipeline;

		return streamGraph.getStreamingPlanAsJSON();
	}

	@Override
	public boolean canTranslate(Pipeline pipeline) {
		return pipeline instanceof StreamGraph;
	}
}
```

## StreamGraph 到 JobGraph 的转换

接着走到 StreamGraph 中的 getJobGraph() 方法：
```java
public JobGraph getJobGraph(@Nullable JobID jobID) {
	return StreamingJobGraphGenerator.createJobGraph(this, jobID);
}
```

接着走到 StreamingJobGraphGenerator 的 createJobGraph() 方法：
```java
/**
 * 传入 StreamGraph，生成 JobGraph
 */
public static JobGraph createJobGraph(StreamGraph streamGraph) {
	return createJobGraph(streamGraph, null);
}

public static JobGraph createJobGraph(StreamGraph streamGraph, @Nullable JobID jobID) {
	return new StreamingJobGraphGenerator(streamGraph, jobID).createJobGraph();
}

/**
 * 核心方法
 * StreamGraph 转 JobGraph 的整体流程
 */
private JobGraph createJobGraph() {
	preValidate();

	// make sure that all vertices start immediately
	// 设置调度模式，streaming 模式下，调度模式是所有节点一起启动
	jobGraph.setScheduleMode(streamGraph.getScheduleMode());

	// 1. 广度优先遍历 StreamGraph 并且为每个 SteamNode 生成一个唯一确定的 hash id
	// Generate deterministic hashes for the nodes in order to identify them across
	// submission iff they didn't change.
	// 保证如果提交的拓扑没有改变，则每次生成的 hash id 都是一样的，这里只要保证 source 的顺序是确定的，就可以保证最后生产的 hash id 不变
	// 它是利用 input 节点的 hash 值及该节点在 map 中位置（实际上是 map.size 算的）来计算确定的
	Map<Integer, byte[]> hashes = defaultStreamGraphHasher.traverseStreamGraphAndGenerateHashes(streamGraph);

	// Generate legacy version hashes for backwards compatibility
	// 这个设置主要是为了防止 hash 机制变化时出现不兼容的情况
	List<Map<Integer, byte[]>> legacyHashes = new ArrayList<>(legacyStreamGraphHashers.size());
	for (StreamGraphHasher hasher : legacyStreamGraphHashers) {
		legacyHashes.add(hasher.traverseStreamGraphAndGenerateHashes(streamGraph));
	}

	Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes = new HashMap<>();

	// 2. 最重要的函数，生成 JobVertex/JobEdge/IntermediateDataSet 等，并尽可能地将多个 StreamNode 节点 chain 在一起
	setChaining(hashes, legacyHashes, chainedOperatorHashes);

	// 3. 将每个 JobVertex 的入边集合也序列化到该 JobVertex 的 StreamConfig 中 (出边集合已经在 setChaining 的时候写入了)
	setPhysicalEdges();

	// 4. 根据 group name，为每个 JobVertex 指定所属的 SlotSharingGroup 以及设置 CoLocationGroup
	setSlotSharingAndCoLocation();

	// 5. 其他设置
	// 设置 ManagedMemory 因子
	setManagedMemoryFraction(
		Collections.unmodifiableMap(jobVertices),
		Collections.unmodifiableMap(vertexConfigs),
		Collections.unmodifiableMap(chainedConfigs),
		id -> streamGraph.getStreamNode(id).getMinResources(),
		id -> streamGraph.getStreamNode(id).getManagedMemoryWeight());

	// checkpoint相关的配置
	configureCheckpointing();

	// savepoint相关的配置
	jobGraph.setSavepointRestoreSettings(streamGraph.getSavepointRestoreSettings());

	// 用户的第三方依赖包就是在这里（cacheFile）传给 JobGraph
	JobGraphGenerator.addUserArtifactEntries(streamGraph.getUserArtifacts(), jobGraph);

	// set the ExecutionConfig last when it has been finalized
	try {
		// 将 StreamGraph 的 ExecutionConfig 序列化到 JobGraph 的配置中
		jobGraph.setExecutionConfig(streamGraph.getExecutionConfig());
	}
	catch (IOException e) {
		throw new IllegalConfigurationException("Could not serialize the ExecutionConfig." +
				"This indicates that non-serializable types (like custom serializers) were registered");
	}

	return jobGraph;
}
```



