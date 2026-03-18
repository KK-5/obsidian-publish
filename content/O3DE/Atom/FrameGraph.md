# FrameGraph
一张包含了所有Scope和所需资源的有向无环图，属于数据，有拓扑排序方法
# Builder
构建FrameGraph，具体是提供FrameGraphInterface和FrameAttachmentDatabaseInterface给外部，让它们可以向FrameGraph中添加Scope和资源。程序
# Compiler
编译FrameGraph，梳理有向无环图，根据不同的Command队列进行区分，创建临时的资源并设解析其生命周期，为资源创建view。程序
# Executer
执行FrameGraph，具体为录制每个Scope的命令。几乎都是由后端API实现的。程序




使用一个Component描述FrameGraph，然后各个阶段来处理它。

Scope 对应 Entity

GraphCom
GraphNode
	Scope* 指向实际的scope，应该可以消除
	vector<Scope*> 两个，记录前驱后后继节点，用index替代
	uint16_t 记录未排序的前驱节点个数
	ScopeGroupId 所属group id
GraphEdge
	 uint32_t 两个，两端连接GraphNode

Attachment
所用的资源，一堆span引用，或者vector\<index\>

Func
对应的FrameGraph 3 个阶段的函数

