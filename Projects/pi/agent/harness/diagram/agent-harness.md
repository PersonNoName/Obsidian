## Agent-Loop
+ 文件定位：agent-loop.ts
```mermaid
flowchart TD
	subgraph AgentLoop["AgentLoop"]
		stream["stream: 返回loop event"]
		runAgentLoopApi["调用runAgentLoop"]
	end
	
	subgraph AgentStream["createAgentStream"]
		EventStream["EventStream"]
	end
	
	subgraph runAgentLoop["runAgentLoop"]
		preEvent["preEvent<br/><br/>agent_start <br/>turn_start<br/>遍历prompts<br>[message_start<br/>  message_end]"]
		runLoopApi["调用runLoop"]
	end
	
	subgraph runLoop["runLoop"]
		loopInCircle["内层循环-steering<br/>终止条件：无steering message与无toolcall<br/>执行：与model通信获取下一步指令streamAssistantResponse --> 工具调用 executeToolCalls"]
		loopOutCircle["外层循环-followup<br/>终止条件：无followup message"]
	end
	
	subgraph streamAssistantResponse["streamAssistantResponse"]
		convertModelContext["转换成模型可理解context"]
		streamFunction["调用streamFunction获得模型response"]
		addMessage["动态将流式输出的response添加至消息结尾"]
	end
	
	subgraph executeToolCalls["executeToolCalls"]
		filterToolcalls["从上下文中筛选出toolcalls"]
		executeTool["根据tool执行方式采用parallel或sequence方式来执行tool"]
	end
	
	AgentStream --> stream
	runAgentLoop --> runAgentLoopApi
	runLoop --> runLoopApi
	streamAssistantResponse --> loopInCircle
	executeToolCalls --> loopInCircle
	
	convertModelContext --> streamFunction
	streamFunction --> addMessage
	
	filterToolcalls --> executeTool
```
## Agent Loop 执行流程

- 文件定位：`packages/agent/src/agent-loop.ts`

```mermaid
flowchart TD
    subgraph AgentLoop["agentLoop / agentLoopContinue"]
        createStream["createAgentStream(): 创建 EventStream"]
        callRunAgentLoop["调用 runAgentLoop / runAgentLoopContinue"]
        returnStream["返回 EventStream"]
    end

    subgraph RunAgentLoop["runAgentLoop"]
        initContext["合并 prompts 到 currentContext.messages"]
        emitStart["emit: agent_start / turn_start / prompt message_start+end"]
        callRunLoop["调用 runLoop"]
    end

    subgraph RunLoop["runLoop"]
        initSteering["读取 getSteeringMessages()"]
        outerLoop["外层循环：follow-up loop"]
        innerLoop["内层循环：steering / tool-result continuation"]
        assistantTurn["调用 streamAssistantResponse"]
        checkTools["检查 assistant message 中的 toolCall"]
        toolExec["调用 executeToolCalls"]
        appendToolResults["将 toolResult 追加到 currentContext 和 newMessages"]
        prepareNext["prepareNextTurn / shouldStopAfterTurn"]
        followUp["读取 getFollowUpMessages()"]
        agentEnd["emit: agent_end"]
    end

    subgraph StreamAssistantResponse["streamAssistantResponse"]
        transformContext["transformContext?"]
        convertToLlm["convertToLlm: AgentMessage[] -> LLM Message[]"]
        buildLlmContext["构造 LLM context: systemPrompt / messages / tools"]
        callStreamFn["调用 streamFn 或 streamSimple"]
        updatePartial["流式更新 partial assistant message"]
        finalAssistant["done/error 后写入 final assistant message"]
    end

    subgraph ExecuteToolCalls["executeToolCalls"]
        filterToolCalls["筛选 toolCall"]
        chooseMode["根据 config.toolExecution 或 tool.executionMode 选择 sequential / parallel"]
        prepareTool["prepareArguments / validateToolArguments / beforeToolCall"]
        runTool["执行 tool.execute"]
        afterTool["afterToolCall"]
        emitToolResult["生成并 emit toolResult message"]
    end

    createStream --> callRunAgentLoop
    callRunAgentLoop --> returnStream
    callRunAgentLoop --> initContext
    initContext --> emitStart
    emitStart --> callRunLoop
    callRunLoop --> initSteering

    initSteering --> outerLoop
    outerLoop --> innerLoop
    innerLoop --> assistantTurn
    assistantTurn --> transformContext
    transformContext --> convertToLlm
    convertToLlm --> buildLlmContext
    buildLlmContext --> callStreamFn
    callStreamFn --> updatePartial
    updatePartial --> finalAssistant

    finalAssistant --> checkTools
    checkTools --> toolExec
    toolExec --> filterToolCalls
    filterToolCalls --> chooseMode
    chooseMode --> prepareTool
    prepareTool --> runTool
    runTool --> afterTool
    afterTool --> emitToolResult
    emitToolResult --> appendToolResults

    appendToolResults --> prepareNext
    checkTools --> prepareNext
    prepareNext --> innerLoop
    prepareNext --> followUp
    followUp --> outerLoop
    followUp --> agentEnd
```

## Agent 状态封装与事件处理流程

- 文件定位：`packages/agent/src/agent.ts`

```mermaid
flowchart TD
    subgraph AgentConstructor["Agent constructor"]
        initState["createMutableAgentState(initialState)<br/>初始化 systemPrompt / model / tools / messages"]
        initConvert["设置 convertToLlm<br/>默认 defaultConvertToLlm"]
        initStream["设置 streamFn<br/>默认 streamSimple"]
        initHooks["设置 hooks<br/>getApiKey / beforeToolCall / afterToolCall / prepareNextTurn"]
        initQueues["创建 steeringQueue / followUpQueue"]
        initRuntime["设置 sessionId / thinkingBudgets / transport / toolExecution"]
    end

    subgraph PublicAPI["Agent public API"]
        promptApi["prompt(input, images?)"]
        continueApi["continue()"]
        steerApi["steer(message)"]
        followUpApi["followUp(message)"]
        abortApi["abort()"]
        waitApi["waitForIdle()"]
        subscribeApi["subscribe(listener)"]
        resetApi["reset()"]
    end

    subgraph Queues["PendingMessageQueue"]
        enqueueSteering["steeringQueue.enqueue"]
        enqueueFollowUp["followUpQueue.enqueue"]
        drainSteering["steeringQueue.drain"]
        drainFollowUp["followUpQueue.drain"]
        queueMode["drain 模式<br/>all / one-at-a-time"]
    end

    subgraph PromptFlow["prompt() flow"]
        activeCheckPrompt["检查 activeRun<br/>如果正在运行则抛错"]
        normalizeInput["normalizePromptInput<br/>string -> user message<br/>AgentMessage -> array"]
        runPromptMessages["runPromptMessages(messages)"]
    end

    subgraph ContinueFlow["continue() flow"]
        activeCheckContinue["检查 activeRun"]
        readLastMessage["读取最后一条 state.messages"]
        lastAssistant{"最后一条是 assistant?"}
        useQueuedSteering["drain steeringQueue<br/>作为新 prompt 运行<br/>skipInitialSteeringPoll = true"]
        useQueuedFollowUp["drain followUpQueue<br/>作为新 prompt 运行"]
        runContinuation["runContinuation()"]
        continueError["无可继续消息或 assistant 后无队列<br/>抛错"]
    end

    subgraph Lifecycle["runWithLifecycle"]
        createAbort["创建 AbortController"]
        setActiveRun["设置 activeRun<br/>promise / resolve / abortController"]
        markStreaming["state.isStreaming = true<br/>清空 streamingMessage / errorMessage"]
        executeLoop["执行 executor(signal)"]
        handleFailure["handleRunFailure<br/>生成 error / aborted assistant message"]
        finishRun["finishRun<br/>isStreaming=false<br/>清空 streamingMessage / pendingToolCalls<br/>resolve activeRun.promise<br/>activeRun=undefined"]
    end

    subgraph LoopBridge["Agent -> agent-loop.ts bridge"]
        createContext["createContextSnapshot<br/>复制 systemPrompt / messages / tools"]
        createConfig["createLoopConfig<br/>组装模型参数 / hooks / 队列读取函数"]
        callRunAgentLoop["runAgentLoop(...)"]
        callRunAgentLoopContinue["runAgentLoopContinue(...)"]
        processEventsCallback["event => processEvents(event)"]
    end

    subgraph LoopConfig["createLoopConfig"]
        configModel["model / reasoning / sessionId"]
        configStream["onPayload / onResponse / transport / thinkingBudgets"]
        configTools["toolExecution / beforeToolCall / afterToolCall"]
        configPrepare["prepareNextTurn"]
        configContext["convertToLlm / transformContext / getApiKey"]
        configQueues["getSteeringMessages / getFollowUpMessages"]
    end

    subgraph ProcessEvents["processEvents(event)"]
        reduceState["根据 event.type 更新 _state"]
        awaitListeners["按订阅顺序 await listeners"]
        signalCheck["从 activeRun 读取 signal<br/>没有 activeRun 则抛错"]
    end

    initState --> initConvert
    initConvert --> initStream
    initStream --> initHooks
    initHooks --> initQueues
    initQueues --> initRuntime

    promptApi --> activeCheckPrompt
    activeCheckPrompt --> normalizeInput
    normalizeInput --> runPromptMessages

    continueApi --> activeCheckContinue
    activeCheckContinue --> readLastMessage
    readLastMessage --> lastAssistant
    lastAssistant -->|是| drainSteering
    drainSteering --> useQueuedSteering
    drainSteering -->|无 steering| drainFollowUp
    drainFollowUp --> useQueuedFollowUp
    drainFollowUp -->|无 follow-up| continueError
    lastAssistant -->|否| runContinuation

    steerApi --> enqueueSteering
    followUpApi --> enqueueFollowUp
    enqueueSteering --> queueMode
    enqueueFollowUp --> queueMode

    runPromptMessages --> createAbort
    runContinuation --> createAbort
    createAbort --> setActiveRun
    setActiveRun --> markStreaming
    markStreaming --> executeLoop
    executeLoop --> createContext
    executeLoop --> createConfig

    createConfig --> configModel
    createConfig --> configStream
    createConfig --> configTools
    createConfig --> configPrepare
    createConfig --> configContext
    createConfig --> configQueues

    configQueues --> drainSteering
    configQueues --> drainFollowUp

    createContext --> callRunAgentLoop
    createConfig --> callRunAgentLoop
    processEventsCallback --> callRunAgentLoop

    createContext --> callRunAgentLoopContinue
    createConfig --> callRunAgentLoopContinue
    processEventsCallback --> callRunAgentLoopContinue

    runPromptMessages --> callRunAgentLoop
    runContinuation --> callRunAgentLoopContinue

    callRunAgentLoop --> processEventsCallback
    callRunAgentLoopContinue --> processEventsCallback
    processEventsCallback --> reduceState
    reduceState --> signalCheck
    signalCheck --> awaitListeners

    executeLoop -->|throw| handleFailure
    handleFailure --> reduceState
    executeLoop --> finishRun
    handleFailure --> finishRun

    abortApi --> createAbort
    waitApi --> setActiveRun
    subscribeApi --> awaitListeners
    resetApi --> initState
```

## processEvents 状态归约流程

```mermaid
flowchart TD
    event["AgentEvent"] --> switchEvent{"event.type"}

    switchEvent -->|message_start| msgStart["state.streamingMessage = event.message"]
    switchEvent -->|message_update| msgUpdate["state.streamingMessage = event.message"]
    switchEvent -->|message_end| msgEnd["state.streamingMessage = undefined<br/>state.messages.push(event.message)"]

    switchEvent -->|tool_execution_start| toolStart["复制 pendingToolCalls<br/>add(event.toolCallId)<br/>写回 state.pendingToolCalls"]
    switchEvent -->|tool_execution_end| toolEnd["复制 pendingToolCalls<br/>delete(event.toolCallId)<br/>写回 state.pendingToolCalls"]

    switchEvent -->|turn_end| turnEnd{"assistant message<br/>存在 errorMessage?"}
    turnEnd -->|是| setError["state.errorMessage = event.message.errorMessage"]
    turnEnd -->|否| noError["不更新 errorMessage"]

    switchEvent -->|agent_end| agentEnd["state.streamingMessage = undefined"]

    msgStart --> getSignal["读取 activeRun.abortController.signal"]
    msgUpdate --> getSignal
    msgEnd --> getSignal
    toolStart --> getSignal
    toolEnd --> getSignal
    setError --> getSignal
    noError --> getSignal
    agentEnd --> getSignal

    getSignal --> hasSignal{"signal 存在?"}
    hasSignal -->|否| signalError["抛错：Agent listener invoked outside active run"]
    hasSignal -->|是| notifyListeners["按订阅顺序 await listener(event, signal)"]
```

## AgentHarness 执行流程

- 文件定位：`packages/agent/src/harness/agent-harness.ts`

```mermaid
flowchart TD
    subgraph PublicAPI["AgentHarness public API"]
        promptApi["prompt(text, images?)"]
        skillApi["skill(name, ...)"]
        templateApi["promptFromTemplate(name, args)"]
        steerApi["steer(text)"]
        followUpApi["followUp(text)"]
        nextTurnApi["nextTurn(text)"]
        compactApi["compact(...)"]
        navigateApi["navigateTree(targetId, ...)"]
        abortApi["abort()"]
        subscribeApi["subscribe(listener)"]
        onApi["on(type, handler)"]
    end

    subgraph PhaseGuard["phase 守卫"]
        checkIdle["phase !== 'idle' 抛 busy"]
        setTurn["phase = 'turn'"]
        startRun["startRunPromise(): 创建 runPromise"]
    end

    subgraph TurnSetup["executeTurn 前置"]
        createTurnState["createTurnState<br/>session.buildContext<br/>解析 systemPrompt / activeTools / resources"]
        buildUserMsg["createUserMessage(text, images)"]
        drainNextTurn["合并 nextTurnQueue"]
        beforeHook["emitHook: before_agent_start<br/>可注入 messages / systemPrompt"]
    end

    subgraph LoopBridge["executeTurn -> runAgentLoop"]
        createContext["createContext<br/>systemPrompt / messages / activeTools"]
        createLoopConfig["createLoopConfig"]
        createStreamFn["createStreamFn"]
        handleEventCb["event => handleAgentEvent(event, signal)"]
        callRunAgentLoop["runAgentLoop(...)"]
        pickAssistant["从 newMessages 取最后一条 assistant"]
        runFailure["emitRunFailure<br/>构造 error/aborted 消息"]
    end

    subgraph LoopConfig["createLoopConfig 注入的 hook"]
        cfgConvert["convertToLlm"]
        cfgContext["transformContext -> emitHook context"]
        cfgBefore["beforeToolCall -> emitHook tool_call"]
        cfgAfter["afterToolCall -> emitHook tool_result"]
        cfgPrepare["prepareNextTurn<br/>flushPendingSessionWrites<br/>createTurnState + setTurnState"]
        cfgSteer["getSteeringMessages -> drainQueuedMessages(steerQueue)"]
        cfgFollowUp["getFollowUpMessages -> drainQueuedMessages(followUpQueue)"]
    end

    subgraph StreamFn["createStreamFn"]
        getAuth["getApiKeyAndHeaders(model)"]
        mergeHeaders["mergeHeaders(turnState + auth)"]
        beforeReq["emitBeforeProviderRequest -> before_provider_request hook"]
        callStreamSimple["streamSimple(model, context, options)"]
        onPayload["onPayload -> emitBeforeProviderPayload"]
        onResponse["onResponse -> emitOwn after_provider_response"]
    end

    subgraph EventHandling["handleAgentEvent(event)"]
        evMessageEnd["message_end<br/>session.appendMessage + emitAny"]
        evTurnEnd["turn_end<br/>emitAny + flushPendingSessionWrites + emitOwn save_point"]
        evAgentEnd["agent_end<br/>flushPendingSessionWrites<br/>phase='idle' + emitAny + emitOwn settled"]
        evOther["其他事件 -> emitAny"]
    end

    subgraph EventDispatch["事件分发"]
        emitOwn["emitOwn / emitAny<br/>广播给 subscribe('*') 监听器"]
        emitHook["emitHook<br/>调用 on(type) 注册的处理器<br/>返回值可修改流程"]
    end

    subgraph Queues["队列与状态"]
        steerQueue["steerQueue"]
        followUpQueue["followUpQueue"]
        nextTurnQueue["nextTurnQueue"]
        pendingWrites["pendingSessionWrites"]
        emitQueueUpdate["emitQueueUpdate -> queue_update"]
    end

    promptApi --> checkIdle
    skillApi --> checkIdle
    templateApi --> checkIdle
    checkIdle --> setTurn
    setTurn --> startRun
    startRun --> createTurnState

    createTurnState --> buildUserMsg
    buildUserMsg --> drainNextTurn
    drainNextTurn --> beforeHook
    beforeHook --> createContext

    createContext --> callRunAgentLoop
    createLoopConfig --> callRunAgentLoop
    createStreamFn --> callRunAgentLoop
    handleEventCb --> callRunAgentLoop

    createLoopConfig --> cfgConvert
    createLoopConfig --> cfgContext
    createLoopConfig --> cfgBefore
    createLoopConfig --> cfgAfter
    createLoopConfig --> cfgPrepare
    createLoopConfig --> cfgSteer
    createLoopConfig --> cfgFollowUp

    cfgContext --> emitHook
    cfgBefore --> emitHook
    cfgAfter --> emitHook
    cfgPrepare --> pendingWrites
    cfgSteer --> steerQueue
    cfgFollowUp --> followUpQueue

    createStreamFn --> getAuth
    getAuth --> mergeHeaders
    mergeHeaders --> beforeReq
    beforeReq --> callStreamSimple
    callStreamSimple --> onPayload
    callStreamSimple --> onResponse
    onPayload --> emitHook
    onResponse --> emitOwn
    beforeReq --> emitHook

    callRunAgentLoop --> handleEventCb
    handleEventCb --> evMessageEnd
    handleEventCb --> evTurnEnd
    handleEventCb --> evAgentEnd
    handleEventCb --> evOther

    evMessageEnd --> emitOwn
    evTurnEnd --> emitOwn
    evTurnEnd --> pendingWrites
    evAgentEnd --> pendingWrites
    evAgentEnd --> emitOwn
    evOther --> emitOwn

    callRunAgentLoop --> pickAssistant
    callRunAgentLoop -->|throw| runFailure
    runFailure --> handleEventCb

    steerApi --> steerQueue
    followUpApi --> followUpQueue
    nextTurnApi --> nextTurnQueue
    steerQueue --> emitQueueUpdate
    followUpQueue --> emitQueueUpdate
    nextTurnQueue --> emitQueueUpdate

    compactApi --> emitHook
    navigateApi --> emitHook
    abortApi --> emitOwn
    subscribeApi --> emitOwn
    onApi --> emitHook
```
## handleAgentEvent 事件分流

```mermaid
flowchart TD
    event["AgentEvent"] --> switchEvent{"event.type"}

    switchEvent -->|message_end| msgEnd["session.appendMessage(message)"]
    msgEnd --> msgEndEmit["emitAny(event)"]

    switchEvent -->|turn_end| turnEmit["emitAny(event)<br/>捕获错误暂存 eventError"]
    turnEmit --> turnFlush["flushPendingSessionWrites"]
    turnFlush --> turnRethrow{"eventError 存在?"}
    turnRethrow -->|是| turnThrow["抛出 eventError"]
    turnRethrow -->|否| turnSave["emitOwn save_point<br/>带 hadPendingMutations"]

    switchEvent -->|agent_end| agentFlush["flushPendingSessionWrites"]
    agentFlush --> agentIdle["phase = 'idle'"]
    agentIdle --> agentEmit["emitAny(event)"]
    agentEmit --> agentSettled["emitOwn settled<br/>带 nextTurnCount"]

    switchEvent -->|其他| otherEmit["emitAny(event)"]
```
