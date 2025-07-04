# Digdag Task Execution Sequence

このドキュメントは、Digdagにおけるタスクの実行シーケンスを詳細に説明し、スケジューラー、永続化、タスクの状態管理の関係を明らかにします。

## 概要

Digdagは、データベースを中心とした分散状態マシンとして動作します。複数のコンポーネントが協調してワークフローを実行し、タスクの状態をデータベースに永続化します。

## タスクの状態遷移

```
BLOCKED → READY → RUNNING → PLANNED → SUCCESS
                      ↓
                   ERROR → RETRY_WAITING → READY
```

## 主要コンポーネント

- **ScheduleExecutor**: スケジュール実行の起点
- **WorkflowExecutor**: ワークフロー実行エンジン
- **TaskControlStore**: タスク状態の永続化
- **Agent**: タスクの実際の実行
- **Database**: 状態管理の権威的データソース

## 実行シーケンス図

```mermaid
sequenceDiagram
    participant S as ScheduleExecutor
    participant WE as WorkflowExecutor
    participant DB as Database
    participant TC as TaskControlStore
    participant Q as TaskQueue
    participant A as Agent
    participant OM as OperatorManager

    Note over S: 1秒間隔でスケジュール実行
    S->>S: runScheduleOnce()
    S->>DB: lockReadySchedules(limit=1)
    DB-->>S: Schedule
    
    Note over S: スケジュールに基づいてワークフロー実行
    S->>WE: submitWorkflow(AttemptRequest)
    WE->>WE: compiler.compile(workflow)
    WE->>DB: putAndLockSession(session, attempt)
    DB-->>WE: SessionAttempt
    
    Note over WE: タスクをデータベースに保存
    WE->>DB: storeTasks(tasks)
    DB-->>WE: Task IDs
    
    Note over WE: メインループ開始
    loop WorkflowExecutor Main Loop
        WE->>WE: propagateBlockedChildrenToReady()
        WE->>DB: findBlockedTasks()
        DB-->>WE: BlockedTasks
        WE->>TC: setBlockedToReady()
        TC->>DB: UPDATE tasks SET state=READY
        
        WE->>WE: enqueueReadyTasks()
        WE->>DB: findAllReadyTaskIds()
        DB-->>WE: ReadyTaskIds
        
        loop For each ready task
            WE->>TC: lockTaskIfNotLocked(taskId)
            TC->>DB: SELECT FOR UPDATE
            DB-->>TC: TaskLock
            WE->>TC: setReadyToRunning()
            TC->>DB: UPDATE tasks SET state=RUNNING
            WE->>Q: dispatch(TaskQueueRequest)
            Q-->>WE: TaskQueueLock
        end
        
        WE->>WE: propagateAllPlannedToDone()
        WE->>DB: findAllPlannedTasks()
        DB-->>WE: PlannedTasks
        
        loop For each planned task
            WE->>DB: countChildrenByState()
            DB-->>WE: ChildrenCount
            alt All children succeeded
                WE->>TC: setPlannedToSuccess()
                TC->>DB: UPDATE tasks SET state=SUCCESS
            else Some children failed
                WE->>TC: setPlannedToGroupError()
                TC->>DB: UPDATE tasks SET state=GROUP_ERROR
            end
        end
    end
    
    Note over A: 別スレッドでタスク実行
    A->>Q: lockSharedAgentTasks()
    Q-->>A: TaskQueueLock
    A->>A: createTaskRequest()
    A->>OM: run(taskRequest)
    
    Note over OM: オペレータ実行
    OM->>OM: buildOperator()
    OM->>OM: operator.run()
    
    alt Task success
        OM->>WE: taskSucceeded(taskResult)
        WE->>TC: setRunningToPlannedSuccessful()
        TC->>DB: UPDATE tasks SET state=PLANNED
        WE->>DB: addSubtasks(generatedTasks)
    else Task failure
        OM->>WE: taskFailed(taskResult)
        WE->>TC: setRunningToShortCircuitError()
        TC->>DB: UPDATE tasks SET state=ERROR
        WE->>DB: addErrorTasks(errorTasks)
    end
    
    Note over WE: リトライ処理
    WE->>WE: retryRetryWaitingTasks()
    WE->>DB: findRetryWaitingTasks()
    DB-->>WE: RetryWaitingTasks
    WE->>TC: setRetryWaitingToReady()
    TC->>DB: UPDATE tasks SET state=READY
```

## 詳細な実行フロー

### 1. スケジューラーによるワークフロー起動

```java
// ScheduleExecutor.runScheduleOnce()
executor.scheduleWithFixedDelay(() -> runSchedules(), 1, 1, TimeUnit.SECONDS);
```

**実行手順：**
1. **スケジュールロック**: `lockReadySchedules(limit=1)` でデッドロック回避
2. **ワークフロー定義取得**: データベースからワークフロー定義を取得
3. **AttemptRequest作成**: スケジュール時刻とパラメータを設定
4. **ワークフロー実行**: `workflowExecutor.submitWorkflow()` を呼び出し

### 2. セッションと試行の作成

```java
// WorkflowExecutor.submitWorkflow()
Workflow workflow = compiler.compile(def.getName(), def.getConfig());
WorkflowTaskList tasks = workflow.getTasks();
return submitTasks(siteId, ar, tasks);
```

**データベース操作：**
1. **セッション作成**: プロジェクトID、ワークフロー名、セッション時刻でSession作成
2. **試行作成**: リトライ名、パラメータ、タイムゾーンでSessionAttempt作成
3. **トランザクション**: `putAndLockSession()` でセッションと試行を原子的に作成
4. **タスク保存**: `storeTasks()` で全ワークフロータスクをデータベースに挿入

### 3. タスク状態管理と永続化

**TaskStateCode列挙型**による状態定義：
- **BLOCKED(0)**: 依存関係待ち
- **READY(1)**: 実行準備完了
- **RUNNING(4)**: 実行中
- **PLANNED(5)**: 完了、子タスク処理中
- **SUCCESS(7)**: 成功完了
- **ERROR(8)**: 失敗
- **GROUP_ERROR(6)**: 子タスク失敗によるグループ失敗
- **RETRY_WAITING(2)**: リトライ待機中
- **CANCELED(9)**: キャンセル

**状態遷移の永続化**：
```java
// TaskControlStore の原子的状態更新
setReadyToRunning()                  // READY → RUNNING
setRunningToPlannedSuccessful()      // RUNNING → PLANNED (子タスクあり)
setRunningToShortCircuitSuccess()    // RUNNING → SUCCESS (子タスクなし)
setRunningToShortCircuitError()      // RUNNING → ERROR
```

### 4. WorkflowExecutorメインループ

**1秒間隔でのポーリング実行**：
```java
workflowExecutor.runWhile(() -> !stop);
```

**実行サイクル**：
1. **`propagateBlockedChildrenToReady()`**: 依存関係が満たされたタスクをBLOCKED → READY
2. **`retryRetryWaitingTasks()`**: リトライ時間経過したタスクをRETRY_WAITING → READY
3. **`enqueueReadyTasks()`**: READYタスクをキューに送信
4. **`propagateAllPlannedToDone()`**: PLANNEDタスクの子タスク完了チェック、SUCCESS/ERRORに遷移
5. **`propagateSessionArchive()`**: 完了セッションのアーカイブ

### 5. タスクキューとエージェントの相互作用

**タスクエンキュー処理**：
1. **`enqueueReadyTasks()`**: `findAllReadyTaskIds()` でREADYタスクを検索
2. **`enqueueTask()`**: 個別タスクをロックし：
   - タスクがまだREADY状態か確認
   - タスクIDベースの一意名で`TaskQueueRequest`作成
   - `dispatcher.dispatch()` でキューに送信
   - 原子的にタスクをRUNNING状態に遷移

**エージェント実行**：
1. **`MultiThreadAgent`**: 別スレッドで実行、タスクをポーリング
2. **`lockSharedAgentTasks()`**: ロック保持でキューからタスク取得
3. **`OperatorManager.run()`**: 実際のタスクロジック実行
4. **タスク完了**: `WorkflowExecutor.taskSucceeded()` または `taskFailed()` にコールバック

### 6. データベース相互作用とトランザクション管理

**TransactionManager**による一貫性確保：
- 全状態変更はデータベーストランザクション内で実行
- **楽観的ロック**: 同時状態変更を防止
- **行レベルロック**: タスク状態一貫性を保証

**主要データベース操作**：
- **SessionStoreManager**: セッション、試行、タスクの管理
- **TaskControlStore**: 原子的タスク状態遷移
- **タスク関係**: 依存関係追跡のため別途保存
- **パラメータ保存**: exportパラメータとstoreパラメータをタスクごとに追跡

### 7. タスク依存関係の解決

**依存関係管理**：
1. **タスクコンパイル**: `WorkflowCompiler` が親/子および上流関係でタスクDAGを構築
2. **依存関係追跡**: データベースにタスク関係として保存
3. **準備完了チェック**: 上流の全依存関係が `canRunDownstreamStates()` (SUCCESS) の時タスクがREADY
4. **並列実行**: `_parallel` 設定と依存関係解決により制御

### 8. エラーハンドリングと回復

**タスク失敗処理**：
1. **エラータスク生成**: 失敗タスクが `_error` 子タスクをトリガー
2. **リトライロジック**: `RetryControl` がリトライ間隔と試行回数を管理
3. **グループエラー伝播**: 子タスク失敗時に親タスクがGROUP_ERRORに遷移
4. **状態回復**: 失敗タスクを最後の成功チェックポイントから再開可能

## 主要なデータフロー

```
Scheduler → WorkflowExecutor → TaskQueue → Agent → Database
    ↓           ↓                 ↓         ↓        ↓
Schedule    Session/Attempt    Task      Operator   State
Trigger  →  Creation      → Enqueue  →  Execution → Persistence
```

## 重要な設計原則

1. **分散状態マシン**: データベースがタスク状態の権威的ソース
2. **ポーリングループ**: 状態変化を検出して伝播
3. **楽観的並行性**: 複数のエグゼキューターが安全に動作
4. **エージェントスレッド**: 独立してタスクを実行し、結果を報告
5. **水平スケーラビリティ**: 複数のWorkflowExecutorとAgentインスタンスが並行動作

この設計により、Digdagは高可用性と拡張性を持つワークフロー実行エンジンを実現しています。