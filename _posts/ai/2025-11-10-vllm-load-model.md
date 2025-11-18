---
title: "VLLM 源码 —— 单机单卡模型的加载"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - vllm
---
## 背景
我们通过vllm运行大模型的时候，从日志中可以看到经历了很多流程，包括物理设备检测、参数的校验、GPU显存检测、模型检测、模型加载、kv 管理、分布式推理等操作，最终启动了http server服务提供给我们去调用，在这个过程的背后是什么原理，我们尝试通过阅读源码的方式来学习下。

本次我们先看看大模型加载的流程。我们运行大模型的命令行是：
```
vllm serve /var/lib/cache/model_scope/deepseek-ai/DeepSeek-OCR --trust-remote-code --gpu-memory-utilization=0.3 --host 0.0.0.0 --port 40051 --served-model-name deepseek-ocr
```

使用的vllm版本是v0.11.1

## 入口函数
vllm的入口文件是vllm/entrypoints/cli/serve.py
```
class ServeSubcommand(CLISubcommand):
    """The `serve` subcommand for the vLLM CLI."""

    name = "serve"

    @staticmethod
    def cmd(args: argparse.Namespace) -> None:
        # If model is specified in CLI (as positional arg), it takes precedence
        if hasattr(args, "model_tag") and args.model_tag is not None:
            args.model = args.model_tag
				
        # 默认情况下这里的headless是False，api_server_count=1
        if args.headless or args.api_server_count < 1:
            run_headless(args)
        else:
            if args.api_server_count > 1:
                run_multi_api_server(args)
            else:
                # 最终走到了run_server中
                # Single API server (this process).
                uvloop.run(run_server(args))
``` 

run_server中根据参数创建socket，重点看下run_server_worker函数
```
async def run_server_worker(
    listen_address, sock, args, client_config=None, **uvicorn_kwargs
) -> None:
    ...
    # 大模型的加载就在build_async_engine_client中
    async with build_async_engine_client(
        args,
        client_config=client_config,
    ) as engine_client:
        maybe_register_tokenizer_info_endpoint(args)
        app = build_app(args)

        await init_app_state(engine_client, app.state, args)
				
        logger.info(
            "Starting vLLM API server %d on %s",
            engine_client.vllm_config.parallel_config._api_process_rank,
            listen_address,
        )
       # 到了这里模型已经加载完成，可以为外部提供api服务了
        shutdown_task = await serve_http(
            app,
            sock=sock,
            enable_ssl_refresh=args.enable_ssl_refresh,
            host=args.host,
            port=args.port,
            log_level=args.uvicorn_log_level,
            # NOTE: When the 'disable_uvicorn_access_log' value is True,
            # no access log will be output.
            access_log=not args.disable_uvicorn_access_log,
            timeout_keep_alive=envs.VLLM_HTTP_TIMEOUT_KEEP_ALIVE,
            ssl_keyfile=args.ssl_keyfile,
            ssl_certfile=args.ssl_certfile,
            ssl_ca_certs=args.ssl_ca_certs,
            ssl_cert_reqs=args.ssl_cert_reqs,
            h11_max_incomplete_event_size=args.h11_max_incomplete_event_size,
            h11_max_header_count=args.h11_max_header_count,
            **uvicorn_kwargs,
        )

    # NB: Await server shutdown only after the backend context is exited
    try:
        await shutdown_task
    finally:
        sock.close()
``` 

build_async_engine_client函数中将外部参数封装成了engine_args类，我们重点看build_async_engine_client_from_engine_args
参数说明：
- usage_context=UsageContext.OPENAI_API_SERVER
- disable_frontend_multiprocessing=False
- client_config=None
```
async def build_async_engine_client_from_engine_args(
    engine_args: AsyncEngineArgs,
    *,
    usage_context: UsageContext = UsageContext.OPENAI_API_SERVER,
    disable_frontend_multiprocessing: bool = False,
    client_config: dict[str, Any] | None = None,
) -> AsyncIterator[EngineClient]:
    ...
    # 生成vllm配置
    vllm_config = engine_args.create_engine_config(usage_context=usage_context)

    # V1 AsyncLLM.
    assert envs.VLLM_USE_V1

    ...
    # 生成异步LLM，这里在生成llm的同时还加载了大模型权重文件
    try:
        async_llm = AsyncLLM.from_vllm_config(
            vllm_config=vllm_config,
            usage_context=usage_context,
            enable_log_requests=engine_args.enable_log_requests,
            aggregate_engine_logging=engine_args.aggregate_engine_logging,
            disable_log_stats=engine_args.disable_log_stats,
            client_addresses=client_config,
            client_count=client_count,
            client_index=client_index,
        )

        # Don't keep the dummy data in memory
        await async_llm.reset_mm_cache()

        yield async_llm
    finally:
        if async_llm:
            async_llm.shutdown()
``` 

from_vllm_config的内容很简单，但是要注意Executor.get_class(vllm_config)，这里用来获取模型加载器的函数
vllm中将加载大模型的模块叫executor。单机的情况下distributed-executor-backend参数是uni，分布式集群时可以使用ray，单机多卡时使用mp。
本次我们使用单机模式跑的
```
    def from_vllm_config(
        cls,
        vllm_config: VllmConfig,
        start_engine_loop: bool = True,
        usage_context: UsageContext = UsageContext.ENGINE_CONTEXT,
        stat_loggers: list[StatLoggerFactory] | None = None,
        enable_log_requests: bool = False,
        aggregate_engine_logging: bool = False,
        disable_log_stats: bool = False,
        client_addresses: dict[str, str] | None = None,
        client_count: int = 1,
        client_index: int = 0,
        disable_log_requests: bool = True,  # Deprecated, will be removed
    ) -> "AsyncLLM":
        ...
        # Executor.get_class(vllm_config)加载模型加载类，这里因为是单机运行，获取的模型加载器是UniProcExecutor
        return cls(
            vllm_config=vllm_config,
            executor_class=Executor.get_class(vllm_config),
            start_engine_loop=start_engine_loop,
            stat_loggers=stat_loggers,
            log_requests=enable_log_requests,
            log_stats=not disable_log_stats,
            aggregate_engine_logging=aggregate_engine_logging,
            usage_context=usage_context,
            client_addresses=client_addresses,
            client_count=client_count,
            client_index=client_index,
        )
``` 

AsyncLLM类初始化函数中会去加载模型文件，这里我们分别从分词器和模型加载器两个部分分别来看看具体的实现
```
class AsyncLLM(EngineClient):
    def __init__(
        ...
    ) -> None:
        ...
        self.model_config = vllm_config.model_config
        self.vllm_config = vllm_config
        self.observability_config = vllm_config.observability_config
        self.log_requests = log_requests

        custom_stat_loggers = list(stat_loggers or [])
        custom_stat_loggers.extend(load_stat_logger_plugin_factories())

        has_custom_loggers = bool(custom_stat_loggers)
        self.log_stats = log_stats or has_custom_loggers
        if not log_stats and has_custom_loggers:
            logger.info(
                "AsyncLLM created with log_stats=False, "
                "but custom stat loggers were found; "
                "enabling logging without default stat loggers."
            )
				
				# 这里默认skip_tokenizer_init是False
        if self.model_config.skip_tokenizer_init:
            tokenizer = None
        else:
				    # 加载分词器，下面讲
            tokenizer = init_tokenizer_from_configs(self.model_config)

        # 负责处理模型的输入数据，包括参数验证、文本预处理、多模态数据处理
        self.processor = Processor(self.vllm_config, tokenizer)
				# 这里默认没有指定io_processor，所以是空的
        self.io_processor = get_io_processor(
            self.vllm_config,
            self.model_config.io_processor_plugin,
        )
				# OutputProcessor (converts EngineCoreOutputs --> RequestOutput).
        self.output_processor = OutputProcessor(
            self.tokenizer, log_stats=self.log_stats
        )
        ...

        # 开始加载模型文件
        self.engine_core = EngineCoreClient.make_async_mp_client(
            vllm_config=vllm_config,
            executor_class=executor_class,
            log_stats=self.log_stats,
            client_addresses=client_addresses,
            client_count=client_count,
            client_index=client_index,
        )
        ...
        # output 处理器
        self.output_handler: asyncio.Task | None = None
        try:
            # Start output handler eagerly if we are in the asyncio eventloop.
            asyncio.get_running_loop()
            self._run_output_handler()
        except RuntimeError:
            pass
        ...
        else:
            self.profiler = None
``` 

分词器的初始化是init_tokenizer_from_configs函数中处理的
如果是从网络中下载分词器相关文件，则会排除权重文件的，这里我们直接读取本地模型目录。默认情况下我们使用AutoTokenizer.from_pretrained来加载tokenizer_config.json文件中的tokenizer_class属性
```
tokenizer = AutoTokenizer.from_pretrained(
                tokenizer_name,
                *args,
                trust_remote_code=trust_remote_code,
                revision=revision,
                **kwargs,
            )
``` 

下面我看看函数make_async_mp_client
参数data_parallel_size=1，因此最终返回的是AsyncMPClient
AsyncMPClient这个类的初始本身没有什么，我们需要注意的是父类中对executor_class的处理
```
    def make_async_mp_client(
        vllm_config: VllmConfig,
        executor_class: type[Executor],
        log_stats: bool,
        client_addresses: dict[str, str] | None = None,
        client_count: int = 1,
        client_index: int = 0,
    ) -> "MPClient":
        parallel_config = vllm_config.parallel_config
        client_args = (
            vllm_config,
            executor_class,
            log_stats,
            client_addresses,
            client_count,
            client_index,
        )
        if parallel_config.data_parallel_size > 1:
            if parallel_config.data_parallel_external_lb:
                # External load balancer - client per DP rank.
                return DPAsyncMPClient(*client_args)
            # Internal load balancer - client balances to all DP ranks.
            return DPLBAsyncMPClient(*client_args)
        return AsyncMPClient(*client_args)
``` 

MPClient的初始化中，参数
- asyncio_mode=True
- executor_class=<class 'vllm.v1.executor.uniproc_executor.UniProcExecutor'>
- log_stats=True
- client_addresses={}
```
class MPClient(EngineCoreClient):
		...
    def __init__(
        self,
        asyncio_mode: bool,
        vllm_config: VllmConfig,
        executor_class: type[Executor],
        log_stats: bool,
        client_addresses: dict[str, str] | None = None,
    ):
        self.vllm_config = vllm_config
        # Serialization setup.
        self.encoder = MsgpackEncoder()
        self.decoder = MsgpackDecoder(EngineCoreOutputs)

        # ZMQ setup.
        sync_ctx = zmq.Context(io_threads=2)
        self.ctx = zmq.asyncio.Context(sync_ctx) if asyncio_mode else sync_ctx

        # This will ensure resources created so far are closed
        # when the client is garbage collected, even if an
        # exception is raised mid-construction.
        self.resources = BackgroundResources(ctx=sync_ctx)
        self._finalizer = weakref.finalize(self, self.resources)
        success = False
        try:
            # State used for data parallel.
            self.engines_running = False

            self.stats_update_address: str | None = None
            if client_addresses:
               ...
            else:
                # Engines are managed by this client.
                with launch_core_engines(vllm_config, executor_class, log_stats) as (
                    engine_manager,
                    coordinator,
                    addresses,
                ):
                    self.resources.coordinator = coordinator
                    self.resources.engine_manager = engine_manager

                (input_address,) = addresses.inputs
                (output_address,) = addresses.outputs
                self.stats_update_address = addresses.frontend_stats_publish_address
                if coordinator is not None:
                    assert self.stats_update_address == (
                        coordinator.get_stats_publish_address()
                    )

            # Create input and output sockets.
            self.input_socket = self.resources.input_socket = make_zmq_socket(
                self.ctx, input_address, zmq.ROUTER, bind=True
            )
            self.resources.output_socket = make_zmq_socket(
                self.ctx, output_address, zmq.PULL
            )

            parallel_config = vllm_config.parallel_config
            dp_size = parallel_config.data_parallel_size
            dp_rank = parallel_config.data_parallel_rank
            dp_local_size = parallel_config.data_parallel_size_local
            offline_mode = parallel_config.data_parallel_rank_local is not None
            # Client manages local+remote EngineCores in pure internal LB case.
            # Client manages local EngineCores in hybrid and external LB case.
            local_engines_only = (
                parallel_config.data_parallel_hybrid_lb
                or parallel_config.data_parallel_external_lb
            )

            num_ranks = dp_local_size if local_engines_only else dp_size
            self.engine_ranks_managed = (
                [dp_rank] if offline_mode else list(range(dp_rank, dp_rank + num_ranks))
            )
            assert parallel_config.data_parallel_size_local <= len(
                self.engine_ranks_managed
            )

            # ZMQ identity of each engine that this client will talk to.
            self.core_engines: list[EngineIdentity] = [
                rank.to_bytes(2, "little") for rank in self.engine_ranks_managed
            ]

            # Wait for ready messages from each engine on the input socket.
            identities = set(self.core_engines)
            sync_input_socket = zmq.Socket.shadow(self.input_socket)
            while identities:
                if not sync_input_socket.poll(timeout=600_000):
                    raise TimeoutError(
                        "Timed out waiting for engines to send"
                        "initial message on input socket."
                    )
                identity, _ = sync_input_socket.recv_multipart()
                identities.remove(identity)

            self.core_engine: EngineIdentity = self.core_engines[0]
            self.utility_results: dict[int, AnyFuture] = {}

            # Request objects which may contain pytorch-allocated tensors
            # that we need to keep references to until zmq is done with the
            # underlying data.
            self.pending_messages = deque[tuple[zmq.MessageTracker, Any]]()

            # Start monitoring engine core processes for unexpected failures
            self.start_engine_core_monitor()

            success = True
        finally:
            if not success:
                self._finalizer()
``` 

由于我们是单机单卡场景，这里的data_parallel_size和data_parallel_size_local都是1
看下来核心代码就在CoreEngineProcManager中了，我们继续往下看
```
def launch_core_engines(
    vllm_config: VllmConfig,
    executor_class: type[Executor],
    log_stats: bool,
    num_api_servers: int = 1,
) -> Iterator[
    tuple[
        CoreEngineProcManager | CoreEngineActorManager | None,
        DPCoordinator | None,
        EngineZmqAddresses,
    ]
]:
    ...

    # local_start_index=0因此offline_mode=True
    offline_mode = local_start_index is not None

    # client_local_only = True
    client_local_only = (
        offline_mode or local_engines_only or (local_engine_count == dp_size)
    )

    # 初始化input/output zmq
    addresses = EngineZmqAddresses(
        inputs=[
            get_engine_client_zmq_addr(client_local_only, host)
            for _ in range(num_api_servers)
        ],
        outputs=[
            get_engine_client_zmq_addr(client_local_only, host)
            for _ in range(num_api_servers)
        ],
    )

    # 这里由于是单机单卡，run_coordinator=False  data_parallel_backend=uni
    ...
   
    if offline_mode:
        assert local_engine_count == 1
        engines_to_handshake = [CoreEngine(index=dp_rank, local=True)]
    ...
		
    # handshake_local_only = True
    handshake_local_only = offline_mode or local_engine_count == dp_size
    handshake_address = get_engine_client_zmq_addr(
        handshake_local_only, host, parallel_config.data_parallel_rpc_port
    )

    if local_engines_only and dp_rank > 0:
        assert not handshake_local_only
        local_handshake_address = get_open_zmq_ipc_path()
        client_handshake_address = local_handshake_address
    else:
        local_handshake_address = handshake_address
        client_handshake_address = None

    with zmq_socket_ctx(
        local_handshake_address, zmq.ROUTER, bind=True
    ) as handshake_socket:
        from vllm.v1.engine.core import EngineCoreProc

        # 启动本地模型
        if local_engine_count:
            local_engine_manager = CoreEngineProcManager(
                EngineCoreProc.run_engine_core,
                vllm_config=vllm_config,
                executor_class=executor_class,
                log_stats=log_stats,
                handshake_address=handshake_address,
                client_handshake_address=client_handshake_address,
                local_client=True,
                local_engine_count=local_engine_count,
                start_index=dp_rank,
                local_start_index=local_start_index or 0,
            )
        else:
            local_engine_manager = None

        yield local_engine_manager, coordinator, addresses

        # 等待本地模型起来
        wait_for_engine_startup(
            handshake_socket,
            addresses,
            engines_to_handshake,
            parallel_config,
            vllm_config.cache_config,
            local_engine_manager,
            coordinator.proc if coordinator else None,
        )
``` 

CoreEngineProcManager的初始化流程中根据本地卡的数量创建对应多的进程，进程运行的函数是target_fn
这里target_fn参数传的是EngineCoreProc.run_engine_core，继续看run_engine_core函数
```
class CoreEngineProcManager:
    def __init__(
        self,
        target_fn: Callable,
        local_engine_count: int,
        start_index: int,
        local_start_index: int,
        vllm_config: VllmConfig,
        local_client: bool,
        handshake_address: str,
        executor_class: type[Executor],
        log_stats: bool,
        client_handshake_address: str | None = None,
    ):
        context = get_mp_context()
        common_kwargs = {
            "vllm_config": vllm_config,
            "local_client": local_client,
            "handshake_address": handshake_address,
            "executor_class": executor_class,
            "log_stats": log_stats,
        }

        if client_handshake_address:
            common_kwargs["client_handshake_address"] = client_handshake_address

        self.processes: list[BaseProcess] = []
        local_dp_ranks = []
        # 这里是单机单卡，所以local_engine_count=1
        for index in range(local_engine_count):
            local_index = local_start_index + index
            global_index = start_index + index

            # Start EngineCore in background process.
            local_dp_ranks.append(local_index)
            # 这里添加了一个独立运行target_fn函数的进程，注意这里将executor_class传进去了
            self.processes.append(
                context.Process(
                    target=target_fn,
                    name=f"EngineCore_DP{global_index}",
                    kwargs=common_kwargs
                    | {
                        "dp_rank": global_index,
                        "local_dp_rank": local_index,
                    },
                )
            )

        self._finalizer = weakref.finalize(self, shutdown, self.processes)

        data_parallel = vllm_config.parallel_config.data_parallel_size > 1
        try:
            for proc, local_dp_rank in zip(self.processes, local_dp_ranks):
                # Adjust device control in DP for non-CUDA platforms
                # For CUDA platforms, setting same device id for different DP
                # processes affects NCCL init performance.
                with (
                    set_device_control_env_var(vllm_config, local_dp_rank)
                    if (data_parallel and not current_platform.is_cuda_alike())
                    else contextlib.nullcontext()
                ):
                    # 启动进程
                    proc.start()
        finally:
            # Kill other procs if not all are running.
            if self.finished_procs():
                self.close()
``` 

run_engine_core中核心的函数是EngineCoreProc
```
    def run_engine_core(*args, dp_rank: int = 0, local_dp_rank: int = 0, **kwargs):
				...
        engine_core: EngineCoreProc | None = None
        try:
            parallel_config: ParallelConfig = kwargs["vllm_config"].parallel_config
            if parallel_config.data_parallel_size > 1 or dp_rank > 0:
                ...
            else:
                set_process_title("EngineCore")
                decorate_logs()
                engine_core = EngineCoreProc(*args, **kwargs)

            engine_core.run_busy_loop()
				...
``` 

怎么还没有开始初始化这个executor_class，心累，直接看到底在哪里初始化的吧
```
class EngineCore:
    """Inner loop of vLLM's Engine."""

    def __init__(
        self,
        vllm_config: VllmConfig,
        executor_class: type[Executor],
        log_stats: bool,
        executor_fail_callback: Callable | None = None,
    ):
        ...
        self.vllm_config = vllm_config
        if vllm_config.parallel_config.data_parallel_rank == 0:
            logger.info(
                "Initializing a V1 LLM engine (v%s) with config: %s",
                VLLM_VERSION,
                vllm_config,
            )

        self.log_stats = log_stats

        # 来了来了，初始化了。这里executor_class=UniProcExecutor
        self.model_executor = executor_class(vllm_config)
        if executor_fail_callback is not None:
            self.model_executor.register_failure_callback(executor_fail_callback)

        self.available_gpu_memory_for_kv_cache = -1

        # Setup KV Caches and update CacheConfig after profiling.
        num_gpu_blocks, num_cpu_blocks, kv_cache_config = self._initialize_kv_caches(
            vllm_config
        )

        vllm_config.cache_config.num_gpu_blocks = num_gpu_blocks
        vllm_config.cache_config.num_cpu_blocks = num_cpu_blocks
        self.collective_rpc("initialize_cache", args=(num_gpu_blocks, num_cpu_blocks))

        self.structured_output_manager = StructuredOutputManager(vllm_config)

        # 这里的scheduler_cls=vllm.v1.core.sched.scheduler.Scheduler
        if isinstance(vllm_config.scheduler_config.scheduler_cls, str):
            Scheduler = resolve_obj_by_qualname(
                vllm_config.scheduler_config.scheduler_cls
            )
        else:
            Scheduler = vllm_config.scheduler_config.scheduler_cls

        ...
        if len(kv_cache_config.kv_cache_groups) == 0:
            # Encoder models without KV cache don't support
            # chunked prefill. But do SSM models?
            logger.info("Disabling chunked prefill for model without KVCache")
            vllm_config.scheduler_config.chunked_prefill_enabled = False

        scheduler_block_size = (
            vllm_config.cache_config.block_size
            * vllm_config.parallel_config.decode_context_parallel_size
        )

        # 初始化Scheduler
        self.scheduler: SchedulerInterface = Scheduler(
            vllm_config=vllm_config,
            kv_cache_config=kv_cache_config,
            structured_output_manager=self.structured_output_manager,
            include_finished_set=vllm_config.parallel_config.data_parallel_size > 1,
            log_stats=self.log_stats,
            block_size=scheduler_block_size,
        )
        ...
``` 

开始加载UniProcExecutor了，重点就是末尾的三个调用
```
class UniProcExecutor(Executor):
    def _init_executor(self) -> None:
        """Initialize the worker and load the model."""
        self.driver_worker = WorkerWrapperBase(vllm_config=self.vllm_config, rpc_rank=0)
        distributed_init_method, rank, local_rank = self._distributed_args()
        kwargs = dict(
            vllm_config=self.vllm_config,
            local_rank=local_rank,
            rank=rank,
            distributed_init_method=distributed_init_method,
            is_driver_worker=True,
            shared_worker_lock=Lock(),
        )

        self.async_output_thread: ThreadPoolExecutor | None = None
        if self.max_concurrent_batches > 1:
            self.async_output_thread = ThreadPoolExecutor(
                max_workers=1, thread_name_prefix="WorkerAsyncOutput"
            )
				
				# 加载worker=vllm.v1.worker.gpu_worker.Worker
        self.driver_worker.init_worker(all_kwargs=[kwargs])
				
				# 调用vllm.v1.worker.gpu_worker.Worker的init_device函数
				# 最主要的是初始化了model_runner
        self.driver_worker.init_device()
				
				# 内部调用了self.model_runner.load_model
        self.driver_worker.load_model()
``` 

下面看看load_model函数
```vllm\v1\worker\gpu_model_runner.py
    def load_model(self, eep_scale_up: bool = False) -> None:
        ...
				# 当前场景eep_scale_up=False
        if eep_scale_up:
            ...
        else:
            global_expert_load = None
            old_global_expert_indices = None
            rank_mapping = None

        with DeviceMemoryProfiler() as m:
            time_before_load = time.perf_counter()
						
						# 默认使用 "auto": DefaultModelLoader
            model_loader = get_model_loader(self.load_config)
            self.model = model_loader.load_model(
                vllm_config=self.vllm_config, model_config=self.model_config
            )
            ...
        # wrap the model with full cudagraph wrapper if needed.
        if (
            self.compilation_config.cudagraph_mode.has_full_cudagraphs()
            and not self.parallel_config.enable_dbo
        ):
           # 最终就封装成了self.model对象
            self.model = CUDAGraphWrapper(
                self.model, self.vllm_config, runtime_mode=CUDAGraphMode.FULL
            )
        elif self.parallel_config.enable_dbo:
            ...
``` 

还得继续看load_model函数
```vllm\model_executor\model_loader\base_loader.py
    def load_model(
        self, vllm_config: VllmConfig, model_config: ModelConfig
    ) -> nn.Module:
        """Load a model with the given configurations."""
        device_config = vllm_config.device_config
        load_config = vllm_config.load_config
        load_device = (
            device_config.device if load_config.device is None else load_config.device
        )
        target_device = torch.device(load_device)
        with set_default_torch_dtype(model_config.dtype):
            with target_device:
						    # 加载模型目录中config.json文件的architectures属性
                model = initialize_model(
                    vllm_config=vllm_config, model_config=model_config
                )

            logger.debug("Loading weights on %s ...", load_device)
            # Quantization does not happen in `load_weights` but after it
            self.load_weights(model, model_config)
            process_weights_after_loading(model, model_config, target_device)
        return model.eval()
``` 

下面看看如何加载模型权重文件，重点如下
```vllm\model_executor\model_loader\default_loader.py
    def load_weights(self, model: nn.Module, model_config: ModelConfig) -> None:
        ...
        # 获取模型中所有需要加载的权重，在实际加载后会对比看看是否有没有加载到的
        weights_to_load = {name for name, _ in model.named_parameters()}

        ...
        if model_config.quantization is None:
            # model is not quantized
            loaded_weights = model.load_weights(
                self.get_all_weights(model_config, model)
            )
        ...
``` 

## 疑问
- 多个worker加载模型时，如何分片加载的
  接着上面的代码继续看，model.load_weights会调用模型对应的load_weights函数
```
    def load_weights(self, weights: Iterable[tuple[str, torch.Tensor]]) -> set[str]:
        stacked_params_mapping = [
            # (param_name, shard_name, shard_id)
            ("gate_up_proj", "gate_proj", 0),
            ("gate_up_proj", "up_proj", 1),
            ("fused_qkv_a_proj", "q_a_proj", 0),
            ("fused_qkv_a_proj", "kv_a_proj_with_mqa", 1),
        ]

        ...
        # 这里获取了当前模型维护的named_parameters参数，后面会用来做判断
        params_dict = dict(self.named_parameters())
        loaded_params: set[str] = set()
        for name, loaded_weight in weights:
            if "rotary_emb.inv_freq" in name:
                continue

            spec_layer = get_spec_layer_idx_from_weight_name(self.config, name)
            if spec_layer is not None:
                continue  # skip spec decode layers for main model

            is_fuse_shared_experts_layer = (
                is_rocm_aiter_fusion_shared_expert_enabled()
                and ("mlp.shared_experts" in name)
            )

            for param_name, weight_name, shard_id in stacked_params_mapping:
                ...
                name_mapped = name.replace(weight_name, param_name)

                # QKV fusion is optional, fall back to normal
                # weight loading if it's not enabled
                # if go with fusion option, then update name
                if (
                    param_name == "fused_qkv_a_proj"
                ) and name_mapped not in params_dict:
                    # 如果权重的name不在当前模型的params_dict中，直接不加载
                    continue
                else:
                    name = name_mapped
                # Skip loading extra bias for GPTQ models.
                if name.endswith(".bias") and name not in params_dict:
                    continue

                if is_pp_missing_parameter(name, self):
                    continue

                param = params_dict[name]
                weight_loader = param.weight_loader
                weight_loader(param, loaded_weight, shard_id)
                ...

``` 

## 总结
这里我们只看到了模型的加载逻辑，实际还有其他复杂的流程，比如kv cache管理/调度相关/input和output的处理/tokenizer等等，还要继续学习