# Kubernetes構成・デプロイ手順

[目次に戻る](./taskQueueSystemDesign.md)

- [システム概要・アーキテクチャ](./systemOverview.md)
- [データモデル](./dataModel.md)
- [API仕様](./apiSpec.md)
- [Kotlin実装例](./kotlinExamples.md)
- [エラー発生時の自動修正タスク追加仕様](./autoFixTask.md)
- [監視・ヘルスチェック](./monitoring.md)
- [まとめ・関連リンク](./summaryAndLinks.md)

---

## Kubernetes構成例
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-api
# ...（省略）...
```

## データベース
```yaml
# MongoDB
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
# ...（省略）...
```

## 設定ファイル

### application.yml
```yaml
spring:
  data:
    mongodb:
      host: mongodb
      port: 27017
      database: taskqueue
      username: admin
      password: password
      authentication-database: admin

server:
  port: 8080

logging:
  level:
    com.example.taskqueue: INFO
```

### build.gradle.kts
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation ("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
    implementation("org.mongodb:mongodb-driver-sync")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:mongodb")
}
```

## デプロイ手順

### ビルド
```bash
# アプリケーションビルド
./gradlew build

# Dockerイメージ作成
docker build -t task-api:latest .
docker build -t task-worker:latest .
```

### デプロイ
```bash
# Kubernetesデプロイ
kubectl apply -f kubernetes/
kubectl get pods
kubectl logs -f deployment/task-api
``` 
