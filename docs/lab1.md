## 任务

实现分布式 MapReduce，包含 coordinator 和 worker。worker 和 coordinator 通过 RPC 通信，每一个 worker 进程会向 coordinator 请求一个任务，从一个或多个文件中读取任务输入，执行任务，将任务输出写到一个或多个文件中。如果 worker 执行任务超时（10s），coordinator 会将任务分配给其他 worker。

你的实现应放在 mr/coordinator.go，mr/worker.go 和 mr/rpc.go。

## 规则

1. Map 阶段应将中间键值对分成 nReduce 份
2. X'th reduce 任务输出到 mr-out-X
3. 输出格式需要正确
4. 除了 mr/coordinator.go，mr/worker.go 和 mr/rpc.go 外，其余文件保持原始状态
5. Worker 应将中间文件放在当前目录下
6. Done() 函数是 MapReduce 作业结束的位置
7. 作业结束后 woker 进程应该退出，简单实现是使用 call() 的返回值，coordinator 失联后可以退出，或者 coordinator 向 worker 发送一个 “退出” 的伪任务。

## 提示

1. [Guidance page](https://pdos.csail.mit.edu/6.824/labs/guidance.html)
2. 一种入手方式是首先修改 worker.go 实现向 coordinator 请求任务的 RPC，然后修改 coordinator 响应尚未开始 map 任务的文件名，然后修改 worker 读取文件并调用 Map 函数
3. Map 和 Reduce 函数是使用 Go plugin 在运行时加载的
4. 修改  mr 目录下文件后需要重新打包 so 文件
5. Worker 在不同机器上时需要类似 GFS 的全局文件系统
6. 中间文件的命名可以是 mr-X-Y，X 是 map number，Y 是 reduce number
7. 中间键值对的读写可以使用 Go 的 encoding/json 包
8. 使用 ihash(key) (in worker.go) 函数对一个给定键选择 reduce 任务
9. 参考 mrsequential.go 的代码读取 map 输入文件，排序中间键值对，存储 reduce 输出文件
10. coordinator 作为一个 RPC server 是并发的，需要给共享数据加锁
11. 根据 test-mr.sh 的注释使用 Go 的 race detector
12. Worker 有时需要等待，例如 reduce 任务直到最后一个 map 完成才会开始，一个可能的方案是 worker 周期性的向 coordinator 询问，另一个是使用 RPC handler
13. Coordinator 并不能可靠的区分 worker 的状态，因此超时后一律看作 worker 失效。
14. 备份任务（3.6 节）应该在相对长时间（10s）后安排
15. mrapps/crash.go 可以用于测试故障恢复
16. 使用临时文件写入并在完成写入后自动重命名，使用 ioutil.TempFile 创建临时文件并使用 os.Rename 自动重命名
17. test-mr.sh 在子目录 mr-tmp 下运行
18. test-mr-many.sh 运行 test-mr.sh 多次以发现低概率错误
19. Go RPC 仅发送名称以大写字母开头的结构字段，子结构也必须有大写的字段名称
20. 调用 RPC call()函数时，回复结构应包含所有默认值

## 数据结构设计

### 1. Coordinator

```Go
type MapFile struct {
    MapFileName string    // map file name
    StartTime   time.Time // start time
    HasAssigned bool
    HasFinished bool
}

type ReduceTask struct {
    ReduceTaskId int       // reduce task id
    StartTime    time.Time // start time
    HasAssigned  bool
    HasFinished  bool
}

type Coordinator struct {
    // Your definitions here.
    WorkerId    int          // worker identity
    WorkerNum   int          // worker number in active
    Lock        sync.Mutex   // lock shared data
    Phase       int          // 0 map 1 reduce 2 exit
    NMap        int          // map number
    MapFiles    []MapFile    // map file list
    NReduce     int          // reduce number
    ReduceTasks []ReduceTask // reduce task list
    JobDone     bool         // job done
}
```

### 2. Worker

```Go
// Map functions return a slice of KeyValue.
type KeyValue struct {
    Key   string
    Value string
}

// for sorting by key.
type ByKey []KeyValue

// for sorting by key.
func (a ByKey) Len() int           { return len(a) }
func (a ByKey) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByKey) Less(i, j int) bool { return a[i].Key < a[j].Key }
```

## 接口设计

### 1. Register

```Go
// register

type RegisterArgs struct {
    // nothing
}

type RegisterReply struct {
    WorkerId int
}
```

### 2. GetTask

```Go
// get task

type GetTaskArgs struct {
    WorkerId int
}

type GetTaskReply struct {
    HasTask      bool   // 0 no 1 yes
    TaskType     int    // 0 map 1 reduce 2 exit
    MapFileName  string // map file name
    MapTaskId    int    // map task id, -1 in reduce phase
    NMap         int    // map number
    NReduce      int    // reduce number
    ReduceTaskId int    // reduce task id, -1 in map phase
}
```

### 3. FinishTask

```Go
// finish task

type FinishTaskArgs struct {
    WorkerId     int    // worker identity
    TaskType     int    // 0 map 1 reduce
    MapFileName  string // map file name
    MapTaskId    int    // map task id, -1 in reduce phase
    ReduceTaskId int    // reduce task id, -1 in map phase
}

type FinishTaskReply struct {
    // nothing
}
```