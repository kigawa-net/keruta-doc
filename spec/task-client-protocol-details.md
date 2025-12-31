## 通信プロトコル詳細

### WebSocket 接続

#### 接続確立
```
ws://keruta-api:8080/ws/ktcp
wss://keruta-api:8080/ws/ktcp (推奨)
```

#### 認証メッセージ
```json
{
  "type": "authenticate",
  "token": "<jwt-token>",
  "clientType": "provider",
  "clientVersion": "1.0.0",
  "capabilities": ["kubernetes", "docker", "local"]
}
```

#### ハートビート
```json
{
  "type": "heartbeat",
  "timestamp": "2024-01-01T10:00:00Z",
  "status": "healthy"
}
```

### メッセージタイプ

#### 1. タスク実行要求
```json
{
  "type": "task_execute",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "name": "Build Project",
    "timeout": 3600
  },
  "timestamp": "2024-01-01T10:00:00Z"
}
```

#### 2. タスク状態更新
```json
{
  "type": "task_status_update",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "PROCESSING",
    "progress": 50,
    "message": "ビルド進行中",
    "startedAt": "2024-01-01T10:05:00Z"
  },
  "timestamp": "2024-01-01T10:05:00Z"
}
```

#### 3. ログストリーミング
```json
{
  "type": "task_log",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "level": "INFO",
    "message": "ビルドを開始します",
    "timestamp": "2024-01-01T10:05:30Z",
    "metadata": {
      "phase": "build",
      "step": "npm install"
    }
  },
  "timestamp": "2024-01-01T10:05:30Z"
}
```

#### 4. タスク完了通知
```json
{
  "type": "task_completed",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "COMPLETED",
    "exitCode": 0,
    "completedAt": "2024-01-01T10:10:00Z",
    "resourceUsage": {
      "cpuTime": 45.2,
      "memoryPeak": "256MB",
      "executionTime": 300
    }
  },
  "timestamp": "2024-01-01T10:10:00Z"
}
```

#### 5. エラー通知
```json
{
  "type": "task_error",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "FAILED",
    "errorCode": "BUILD_FAILED",
    "errorMessage": "npm install に失敗しました",
    "failedAt": "2024-01-01T10:08:00Z"
  },
  "timestamp": "2024-01-01T10:08:00Z"
}
```

##### 汎用的なエラー通知例
```json
{
  "type": "generic_error",
  "data": {
    "errorCode": "GENERIC_ERROR",
    "errorMessage": "システムで予期しないエラーが発生しました",
    "exitCode": -1,
    "retryable": false,
    "failedAt": "2024-01-01T10:08:00Z"
  },
  "timestamp": "2024-01-01T10:08:00Z"
}
```

#### 6. タスクキャンセル要求
```json
{
  "type": "task_cancel",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "reason": "ユーザーによるキャンセル",
    "force": false
  },
  "timestamp": "2024-01-01T10:07:00Z"
}
```

#### 7. タスク作成要求 (Create)
```json
{
  "type": "task_create",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "name": "Build Project",
    "description": "プロジェクトのビルド実行",
    "timeout": 3600,
    "tags": ["build", "production"]
  },
  "timestamp": "2024-01-01T10:00:00Z"
}
```

#### 8. タスク作成応答
```json
{
  "type": "task_create_response",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174000",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "CREATED",
    "createdAt": "2024-01-01T10:00:00Z",
    "estimatedStartTime": "2024-01-01T10:05:00Z"
  },
  "timestamp": "2024-01-01T10:00:01Z"
}
```

#### 9. タスク情報取得要求 (Read)
```json
{
  "type": "task_read",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174001",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "includeLogs": true,
    "includeMetadata": true
  },
  "timestamp": "2024-01-01T10:01:00Z"
}
```

#### 10. タスク情報取得応答
```json
{
  "type": "task_read_response",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174001",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "name": "Build Project",
    "description": "プロジェクトのビルド実行",
    "status": "PENDING",
    "timeout": 3600,
    "tags": ["build", "production"],
    "createdAt": "2024-01-01T10:00:00Z",
    "updatedAt": "2024-01-01T10:00:00Z",
    "createdBy": "user123",
    "assignedProvider": null,
    "logs": [],
    "metadata": {
      "estimatedDuration": 300,
      "resourceRequirements": {
        "cpu": "1",
        "memory": "512MB"
      }
    }
  },
  "timestamp": "2024-01-01T10:01:01Z"
}
```

#### 11. タスク更新要求 (Update)
```json
{
  "type": "task_update",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174002",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "name": "Updated Build Project",
    "description": "更新されたプロジェクトのビルド実行",
    "timeout": 1800,
    "tags": ["build", "staging", "urgent"]
  },
  "timestamp": "2024-01-01T10:02:00Z"
}
```

#### 12. タスク更新応答
```json
{
  "type": "task_update_response",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174002",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "UPDATED",
    "updatedAt": "2024-01-01T10:02:00Z",
    "changes": {
      "timeout": {"old": 3600, "new": 1800}
    }
  },
  "timestamp": "2024-01-01T10:02:01Z"
}
```

#### 13. タスク削除要求 (Delete)
```json
{
  "type": "task_delete",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174003",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "reason": "不要になったタスク",
    "force": false
  },
  "timestamp": "2024-01-01T10:03:00Z"
}
```

#### 14. タスク削除応答
```json
{
  "type": "task_delete_response",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174003",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "DELETED",
    "deletedAt": "2024-01-01T10:03:00Z",
    "cleanupStatus": "COMPLETED"
  },
  "timestamp": "2024-01-01T10:03:01Z"
}
```

#### 15. タスク一覧取得要求
```json
{
  "type": "task_list",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174004",
  "data": {
    "status": ["PENDING", "PROCESSING"],
    "tags": ["build"],
    "createdBy": "user123",
    "limit": 50,
    "offset": 0,
    "sortBy": "createdAt",
    "sortOrder": "DESC"
  },
  "timestamp": "2024-01-01T10:04:00Z"
}
```

#### 16. タスク一覧取得応答
```json
{
  "type": "task_list_response",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174004",
  "data": {
    "tasks": [
      {
        "taskId": "123e4567-e89b-12d3-a456-426614174000",
        "name": "Build Project",
        "status": "PENDING",
        "createdAt": "2024-01-01T10:00:00Z",
        "tags": ["build", "production"]
      }
    ],
    "totalCount": 1,
    "hasMore": false
  },
  "timestamp": "2024-01-01T10:04:01Z"
}
```