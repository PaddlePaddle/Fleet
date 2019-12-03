#CPU分布式训练API
## FleetAPI
### Fleet是PaddlePaddle Fluid最新优化的多机API版本， 统一了多机API的实现，兼容Transpiler/Collective两种模式。 可以在MPI、K8S、PaddleCloud以及用户自定义环境下进行多机训练，以及自定义分布式训练配置。

#### API介绍

#### 使用说明
Fleet代码位于python/paddle/fluid/incubate/fleet下， 对外的实例名为fleet， 使用Transpiler模式， 请使用：
```
from paddle.fluid.incubate.fleet.parameter_server.distribute_transpiler import fleet
```

#### 运行流程及原理
1. fleet.init(role_maker=None)
fleet初始化，需要在使用fleet其他接口前先调用，用于定义多机的环境配置。

参数:
    role_maker(RoleMakerBase|None) — 多机环境配置，目前有MPISymetricRoleMaker(默认)和UserDefinedRoleMaker、PaddleCloudRoleMaker等多种。
    MPISymetricRoleMaker在MPI集群下使用，需要自定义环境变量可使用UserDefinedRoleMaker。

返回类型: None

代码示例：
```
exe = fluid.Executor(fluid.CPUPlace())
role = UserDefinedRoleMaker(current_id=0,
                 role=Role.WORKER,
                 worker_num=3,
                 server_endpoints=["127.0.0.1:6001","127.0.0.1:6002"])
fleet.init(role_maker=role)
```

2. fleet.distributed_optimizer(optimizer, strategy=None)

参数：
    optimizer (Optimizer) — 当前网络定义的优化器SGD/ADAM等。
    strategy(Any|None) — 多机策略配置，根据fleet的实现自行配置，DistributedTranspiler和Collective模式指定为DistributeTranspilerConfig。

返回类型：DistributedOptimizer

代码示例：
```
optimizer = fluid.optimizer.SGD(learning_rate=0.1)
config = DistributeTranspilerConfig()
config.sync_mode = False
optimizer = fleet.distributed_optimizer(optimizer, config)
optimizer.minimize(cost)
```

3. fleet.is_server()
判断当前节点是否是Server节点， 是则返回True，否则返回False。在CPU分布式训练下， 节点类型分为trainer和pserver两类。

参数： None
返回类型: bool

代码示例：
```
if fleet.is_server():
    fleet.run_server()
```

4. fleet.init_server(model_dir=None)
加载model_dir中保存的模型相关参数进行PServer的初始化

参数:
    model_dir (str|None) — 模型参数保存的目录。模型参数来自于fleet.save_persistable或者fluid.io.save_persistable保存下来的参数。

返回类型: None

```
if fleet.is_server():
    model_dir = "xxx"
    fleet.init_server(model_dir)
    fleet.run_server()
```

5. fleet.run_server()
启动PServer的进程， 此进程为常驻进程， 会一直监听来自trainer端的消息。 当前版本不会主动退出。

参数: None
返回类型: None

```
if fleet.is_server():
    fleet.init_server()
    fleet.run_server()
```

6. fleet.is_worker()
判断当前节点是否是Worker节点， trainer会启动训练。
参数: None
返回类型: None

7. fleet.init_worker()
如果是worker节点，则会根据当前启动的模式进行针对性的初始化。
参数: None
返回类型: None

8. fleet.save_inference_model
CPU分布式专用的模型保存接口，在trainer端调用，根据用户配置保存模型参数及模型文件， 具体用法参考 https://www.paddlepaddle.org.cn/documentation/docs/zh/api_cn/io_cn/save_inference_model_cn.html#save-inference-model。

10. fleet.save_persistable
CPU分布式专用的模型保存接口，在trainer端调用，根据网络保存完整的模型参数， 具体用法参考 https://www.paddlepaddle.org.cn/documentation/docs/zh/api_cn/io_cn/save_persistables_cn.html#save-persistables。


## TranspilerAPI
TranspilerAPI是老版本的分布式训练API，在设计和实现上有诸多不足之处，如果是新用户，请尽可能选择FleetAPI。Transpiler API可以把单机可以执行的程序快速转变成可以分布式执行的程序。在不同的服务器节点 上，通过传给 transpiler 对应的参数，以获取当前节点需要执行的 Program。
需要配置参数包括：
```table
参数     |  说明
role     | 区分作为pserver启动还是trainer启动，不传给transpile，也可以用其他的变量名或环境变量
trainer_id   |  如果是trainer进程，用于指定当前trainer在任务中的唯一id，从0开始，在一个任务中需保证不重复
pservers   |   当前任务所有pserver的ip:port列表字符串，形式比如：127.0.0.1:6170,127.0.0.1:6171
trainers  |   trainer节点的个数
sync_mode |  True为同步模式，False为异步模式，说明(目前分布式的其他模型需要配置此参数联合其他配置完成)
startup_program | 如果startup_program不是默认的fluid.default_startup_program()，需要传入此参数
current_endpoint | NCCL2模式需要传这个参数，且在分布式训练增量训练是需要指定此参数
```
一个例子，假设有两个节点，分别是 192.168.1.1 和 192.168.1.2 ，使用端口6170，启动4个trainer， 则代码可以写成：
```
role = "PSERVER"
trainer_id = 0  # get actual trainer id from cluster
pserver_endpoints = "192.168.1.1:6170,192.168.1.2:6170"
current_endpoint = "192.168.1.1:6170" # get actual current endpoint
trainers = 4
t = fluid.DistributeTranspiler()
t.transpile(trainer_id, pservers=pserver_endpoints, trainers=trainers)
if role == "PSERVER":
    pserver_prog = t.get_pserver_program(current_endpoint)
    pserver_startup = t.get_startup_program(current_endpoint,
                                            pserver_prog)
    exe.run(pserver_startup)
    exe.run(pserver_prog)
elif role == "TRAINER":
    train_loop(t.get_trainer_program())
```