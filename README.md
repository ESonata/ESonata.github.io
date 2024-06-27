# volcano监控指标分析

## 作业和任务在调度器中的区别

volcano的相关监控指标值都是从volcano-scheduler获取，此处作业和任务的定义以volcano-scheduler中的设计为依据。<br/>
根据volcano-scheduler调度器中的设计，调度器中的Job作业信息指PodGroup，Task任务信息指与PodGroup有所关联的Pod.

## 关键指标项分析

#### 1. e2e_scheduling_latency_milliseconds
分析描述：
调度器启动后，会以1秒为间隔，周期性的执行runOnce方法,该方法中会首先设置scheduleStartTime变量为当前时间，然后获取所有Pending状态的Job，为这些Job执行所有配置的action和plugin，runOnce方法执行完成后，调度器统计从scheduleStartTime时间开始到目前时间为止的持续时间，这个持续时间就是e2e_scheduling_latency_milliseconds。

指标定义:
e2e_scheduling_latency_milliseconds为Histogram桶类型，统计了对所有Pending job完成一次全流程调度的时间分布。
``` 
e2eSchedulingLatency = promauto.NewHistogram(
		prometheus.HistogramOpts{
		Subsystem: VolcanoNamespace,
		Name:      "e2e_scheduling_latency_milliseconds",
		Help:      "E2e scheduling latency in milliseconds (scheduling algorithm + binding)",
		Buckets:   prometheus.ExponentialBuckets(5, 2, 10),
	},
)
```

#### 2. e2e_job_scheduling_latency_milliseconds
分析描述：
volcano job创建后，会有一个创建时间CreationTimestamp，调度器会根据Job创建唯一的PodGroup和相关的task pod，在每一个task pod与具体的Node节点绑定(此时还未更新Pod资源的NodeName)完成后，调度器会统计每个task pod从CreationTimestamp开始到绑定节点完成时的持续时间，这个持续时间就是e2e_job_scheduling_latency_milliseconds。

指标定义:
e2e_job_scheduling_latency_milliseconds为Histogram桶类型，统计了volcano job内的每一个task pod从voclano job创建时间开始，到完成节点绑定的持续时间分布。
``` 
e2eJobSchedulingLatency = promauto.NewHistogram(
	prometheus.HistogramOpts{
		Subsystem: VolcanoNamespace,
		Name:      "e2e_job_scheduling_latency_milliseconds",
		Help:      "E2e job scheduling latency in milliseconds",
		Buckets:   prometheus.ExponentialBuckets(32, 2, 10),
	},
)
```

#### 3. e2e_job_scheduling_duration
分析描述：
与e2e_job_scheduling_latency_milliseconds含义相同，在调度器中都是在同一时刻更新。不同的是e2e_job_scheduling_duration是GaugeVec类型，
有job_name(调度器中为PodGroup名称）,queue,job_namespace等标签。

指标定义:
e2e_job_scheduling_duration为GaugeVec类型，在添加了job_name,queue,job_namespace不同标签的情况下，
同样统计了volcano job内的每一个task pod，从voclano job创建时间开始，到完成节点绑定的持续时间分布。
``` 
e2eJobSchedulingDuration = promauto.NewGaugeVec(
	prometheus.GaugeOpts{
		Subsystem: VolcanoNamespace,
		Name:      "e2e_job_scheduling_duration",
		Help:      "E2E job scheduling duration",
	},
	[]string{"job_name", "queue", "job_namespace"},
)
```

#### 4. plugin_scheduling_latency_microseconds
分析描述：
更新plugin_scheduling_latency_microseconds值时有两种情况：<br>
1.每个调度周期开始后，即runOnce方法执行后，所有插件被遍历加载时，各个插件执行工作所用的时间会更新到plugin_scheduling_latency_microseconds. <br>
2.每个调度周期结束后，即runOnce方法执行结束后，各个插件执行后续清理工作所用的时间会更新到plugin_scheduling_latency_microseconds. <br>

指标定义:
plugin_scheduling_latency_microseconds为HistogramVec类型，在添加了plugin和OnSession标签的情况下，统计了各个插件执行工作所用的时延分布。
``` 
pluginSchedulingLatency = promauto.NewHistogramVec(
	prometheus.HistogramOpts{
		Subsystem: VolcanoNamespace,
		Name:      "plugin_scheduling_latency_microseconds",
		Help:      "Plugin scheduling latency in microseconds",
		Buckets:   prometheus.ExponentialBuckets(5, 2, 10),
	}, []string{"plugin", "OnSession"},
)
```

#### 5. action_scheduling_latency_microseconds
分析描述：
每个调度周期开始后，即runOnce方法执行后，配置的各个action执行工作所用的时间会更新到action_scheduling_latency_microseconds

指标定义:
action_scheduling_latency_microseconds为HistogramVec类型，在添加了action标签的情况下，统计了各个action执行工作所用的时延分布。

```
actionSchedulingLatency = promauto.NewHistogramVec(
	prometheus.HistogramOpts{
		Subsystem: VolcanoNamespace,
		Name:      "action_scheduling_latency_microseconds",
		Help:      "Action scheduling latency in microseconds",
		Buckets:   prometheus.ExponentialBuckets(5, 2, 10),
	}, []string{"action"},
)
```

#### 6. task_scheduling_latency_milliseconds
分析描述：
volcano job创建后，会有一个创建时间CreationTimestamp，调度器会根据Job创建唯一的PodGroup和相关的task pod，此时task pod会有一个创建时间，并且task pod还没有具体的NodeName，当调度器为pod绑定节点以及更新pod资源的NodeName后，会统计task pod从创建时间开始到更新NodeName完成后所用的时间，这个时间就是task_scheduling_latency_milliseconds。

指标定义:
task_scheduling_latency_milliseconds为Histogram类型，统计了所有volcano job所关联的每个task pod从创建时间到更新其NodeName字段后所用的持续时间。
``` 
taskSchedulingLatency = promauto.NewHistogram(
	prometheus.HistogramOpts{
		Subsystem: VolcanoNamespace,
		Name:      "task_scheduling_latency_milliseconds",
		Help:      "Task scheduling latency in milliseconds",
		Buckets:   prometheus.ExponentialBuckets(5, 2, 10),
	},
)
```



