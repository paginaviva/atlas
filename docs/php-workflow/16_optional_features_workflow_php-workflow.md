# 16. Características Opcionales y Extensiones de php-workflow

## 1. Objetivo

Explorar características avanzadas opcionales, patrones de extensión, e integraciones de terceros que potencian workflows en escenarios especializados.

---

## 2. Contexto

Además del core, existen características y patrones opcionales útiles para:
- **Persistencia**: Guardar/recuperar workflows
- **Event Streaming**: Reaccionar a eventos de workflow
- **Generator Patterns**: Memory-efficient processing
- **Integraciones**: Conectar con sistemas externos

---

## 3. Persistencia de Workflows

### 3.1 Save & Resume Pattern

Guarda el estado del workflow para continuar después:

```php
class PersistentWorkflow {
    private WorkflowContainer $state;
    private WorkflowResult $currentResult;
    
    public function start(WorkflowContainer $container): void {
        // Guarda estado inicial
        $this->state = $container;
        $this->saveState('started');
    }
    
    public function resume(string $stateId): void {
        // Carga estado guardado
        $this->state = $this->loadState($stateId);
        
        // Continúa workflow
        $this->currentResult = $this->workflow->executeWorkflow(
            $this->state,
            false
        );
        
        $this->saveState('resumed');
    }
    
    public function pause(string $stateId): void {
        // Pausa y guarda antes de un paso largo
        $this->saveState('paused');
    }
    
    private function saveState(string $event): void {
        // Store en DB, Redis, archivo, etc
        $serialized = serialize($this->state);
        $this->repository->save($serialized, $event);
    }
    
    private function loadState(string $stateId): WorkflowContainer {
        $serialized = $this->repository->load($stateId);
        return unserialize($serialized);
    }
}

// Uso
$persistent = new PersistentWorkflow($workflow);

// Sesión 1: Inicia
$persistent->start($container);

// ... usuario se va ...

// Sesión 2: Reanuda después de días
$persistent->resume('workflow_state_12345');
```

### 3.2 Checkpoint Pattern

Guarda checkpoints en etapas clave:

```php
class CheckpointStep implements WorkflowStep {
    private string $checkpointName;
    private string $storageBackend;  // 'redis', 'database', 'file'
    
    public function __construct(string $name, string $backend = 'redis') {
        $this->checkpointName = $name;
        $this->storageBackend = $backend;
    }
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        // Guarda checkpoint
        $checkpoint = [
            'name' => $this->checkpointName,
            'timestamp' => time(),
            'container' => $this->serialize($container),
            'state' => 'completed',
        ];
        
        $this->store($checkpoint);
        
        $control->attachStepInfo("Checkpoint created: {$this->checkpointName}");
    }
    
    public function getDescription(): string {
        return "Checkpoint: {$this->checkpointName}";
    }
    
    private function store(array $checkpoint): void {
        // Diferentes backends
        match($this->storageBackend) {
            'redis' => $this->redis->set("cp:{$this->checkpointName}", json_encode($checkpoint)),
            'database' => $this->db->insert('checkpoints', $checkpoint),
            'file' => file_put_contents("/tmp/cp_{$this->checkpointName}.json", json_encode($checkpoint)),
        };
    }
    
    private function serialize(WorkflowContainer $container): string {
        return json_encode(array_combine(
            $container->keys(),
            array_map(fn($k) => $container->get($k), $container->keys())
        ));
    }
}

// Uso
$workflow = new Workflow('long-process')
    ->process(new Step1())
    ->process(new CheckpointStep('phase1'))
    ->process(new Step2())
    ->process(new CheckpointStep('phase2'))
    ->process(new Step3());

// Si falla en Step3, puedes reanudar desde checkpoint 'phase2'
```

### 3.3 Snapshot and Recover

```php
class SnapshotRecovery {
    public function createSnapshot(WorkflowResult $result): string {
        $snapshot = [
            'workflow' => $result->getWorkflowName(),
            'status' => $result->success() ? 'success' : 'failed',
            'timestamp' => time(),
            'container' => $this->containerToArray($result->getContainer()),
            'exception' => $result->getException() ? get_class($result->getException()) : null,
            'metadata' => [
                'last_step' => $result->getLastStep()->getDescription(),
                'duration' => $result->getDuration(),
            ],
        ];
        
        $snapshotId = md5(json_encode($snapshot) . time());
        $this->storage->save("snapshot:$snapshotId", $snapshot);
        
        return $snapshotId;
    }
    
    public function recoverFromSnapshot(string $snapshotId): WorkflowContainer {
        $snapshot = $this->storage->get("snapshot:$snapshotId");
        
        $container = new WorkflowContainer();
        foreach ($snapshot['container'] as $key => $value) {
            $container->set($key, $value);
        }
        
        return $container;
    }
    
    private function containerToArray(WorkflowContainer $container): array {
        $array = [];
        foreach ($container->keys() as $key) {
            $array[$key] = $container->get($key);
        }
        return $array;
    }
}
```

---

## 4. Event Streaming y Reactions

### 4.1 Workflow Events

Emite eventos en momentos clave:

```php
interface WorkflowEventListener {
    public function onWorkflowStart(string $workflowName): void;
    public function onStepStart(WorkflowStep $step): void;
    public function onStepComplete(WorkflowStep $step): void;
    public function onStepFail(WorkflowStep $step, Exception $e): void;
    public function onWorkflowComplete(WorkflowResult $result): void;
}

class EventEmitterStep implements WorkflowStep {
    private array $listeners = [];
    
    public function subscribe(WorkflowEventListener $listener): void {
        $this->listeners[] = $listener;
    }
    
    public function emitEvent(string $event, ...$args): void {
        foreach ($this->listeners as $listener) {
            $method = "on" . ucfirst($event);
            if (method_exists($listener, $method)) {
                $listener->$method(...$args);
            }
        }
    }
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $this->emitEvent('stepStart', $this);
        
        try {
            // Process...
            $this->emitEvent('stepComplete', $this);
        } catch (Exception $e) {
            $this->emitEvent('stepFail', $this, $e);
        }
    }
    
    public function getDescription(): string { return "Event emitter"; }
}

// Implementación de listener
class LogStreamListener implements WorkflowEventListener {
    public function onStepStart(WorkflowStep $step): void {
        error_log("→ Starting: " . $step->getDescription());
    }
    
    public function onStepComplete(WorkflowStep $step): void {
        error_log("✓ Completed: " . $step->getDescription());
    }
    
    public function onStepFail(WorkflowStep $step, Exception $e): void {
        error_log("✗ Failed: " . $step->getDescription() . " - " . $e->getMessage());
    }
    
    public function onWorkflowComplete(WorkflowResult $result): void {
        error_log("Workflow finished: " . ($result->success() ? 'SUCCESS' : 'FAILURE'));
    }
    
    public function onWorkflowStart(string $workflowName): void {
        error_log("Starting workflow: $workflowName");
    }
}

// Uso
$listener = new LogStreamListener();
$emitter = new EventEmitterStep();
$emitter->subscribe($listener);
```

### 4.2 Pub/Sub Pattern

Reacciona a eventos vía pub/sub:

```php
class PubSubStep implements WorkflowStep {
    private $pubsub;  // Redis, RabbitMQ, etc
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $event = [
            'type' => 'order.created',
            'data' => $container->get('order'),
            'timestamp' => time(),
        ];
        
        // Publica evento
        $this->pubsub->publish('orders', json_encode($event));
        
        // Suscriptores externos reaccionan
        // - Email service: envía confirmación
        // - Analytics: registra compra
        // - Inventory: actualiza stock
    }
    
    public function getDescription(): string { return "Publish event"; }
}

// Suscriptor externo
class OrderEmailSubscriber {
    public function __construct($pubsub) {
        $pubsub->subscribe('orders', [$this, 'handleOrderEvent']);
    }
    
    public function handleOrderEvent(string $message): void {
        $event = json_decode($message);
        
        if ($event->type === 'order.created') {
            // Envía email
            mail($event->data->customer_email, 'Order Confirmation', ...);
        }
    }
}
```

---

## 5. Generator Pattern (Memory Efficient)

### 5.1 Lazy Evaluation

Procesa streams grandes sin cargar en memoria:

```php
class GeneratorStreamStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $dataSource = $container->get('data_source');
        
        // Procesa items via generator (memory efficient)
        foreach ($this->generateItems($dataSource) as $item) {
            $processed = $this->process($item);
            $container->set("item_{$item['id']}", $processed);
            
            $control->attachStepInfo("Processed item", ['id' => $item['id']]);
        }
    }
    
    private function generateItems($source) {
        // Yield items one by one (no cargan todos en memoria)
        foreach (file($source) as $line) {
            yield json_decode($line, true);
        }
    }
    
    public function process($item) { /* ... */ }
    
    public function getDescription(): string { return "Generator-based streaming"; }
}

// Uso
// Em lugar de: $items = file_get_contents(...); // Carga todo
// Usamos: foreach ($this->generateItems($file)) // Carga línea por línea
```

### 5.2 Pipelined Processing

Procesa datos en etapas tipo Unix pipe:

```php
class PipelineProcessor {
    private array $stages = [];
    
    public function pipe(callable $transform) {
        $this->stages[] = $transform;
        return $this;
    }
    
    public function processStream($items) {
        $data = $items;
        
        foreach ($this->stages as $stage) {
            $data = $stage($data);
        }
        
        return $data;
    }
}

// Uso
$pipeline = new PipelineProcessor()
    ->pipe(fn($items) => array_filter($items, fn($i) => $i['active']))
    ->pipe(fn($items) => array_map(fn($i) => array_merge($i, ['total' => $i['qty'] * $i['price']]), $items))
    ->pipe(fn($items) => array_reduce($items, fn($carry, $i) => $carry + $i['total'], 0));

$result = $pipeline->processStream($items);
```

---

## 6. Integraciones de Terceros

### 6.1 Integration con DataFrame (Pandas-like)

```php
class DataFrameProcessingStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $csvFile = $container->get('csv_file');
        
        // Load CSV como pseudo-DataFrame
        $rows = array_map('str_getcsv', file($csvFile));
        
        // Transformaciones tipo pandas
        $processed = array_filter($rows, fn($r) => $r['status'] === 'active');
        $aggregated = array_reduce($processed, fn($acc, $r) => [
            ...$acc,
            'total' => ($acc['total'] ?? 0) + $r['amount'],
            'count' => ($acc['count'] ?? 0) + 1,
        ]);
        
        $container->set('result', $aggregated);
        $control->attachStepInfo("Processed CSV data", $aggregated);
    }
    
    public function getDescription(): string { return "DataFrame-like processing"; }
}
```

### 6.2 Integration con GraphQL

```php
class GraphQLQueryStep implements WorkflowStep {
    private $graphqlClient;
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $query = $container->get('graphql_query');
        $variables = $container->get('graphql_vars', []);
        
        try {
            $response = $this->graphqlClient->query($query, $variables);
            
            if (isset($response['errors'])) {
                $control->failStep("GraphQL errors: " . json_encode($response['errors']));
                return;
            }
            
            $container->set('graphql_result', $response['data']);
        } catch (Exception $e) {
            $control->failStep("GraphQL query failed: " . $e->getMessage());
        }
    }
    
    public function getDescription(): string { return "GraphQL query execution"; }
}
```

### 6.3 Integration con Apache Kafka

```php
class KafkaProducerStep implements WorkflowStep {
    private $kafkaProducer;
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $topic = $container->get('kafka_topic');
        $message = $container->get('kafka_message');
        
        try {
            $this->kafkaProducer->produce(
                $topic,
                json_encode($message)
            );
            
            $control->attachStepInfo("Message published to Kafka", [
                'topic' => $topic,
                'keys' => array_keys((array)$message),
            ]);
        } catch (Exception $e) {
            $control->warning("Kafka publish failed", $e);
        }
    }
    
    public function getDescription(): string { return "Kafka message producer"; }
}

class KafkaConsumerStep implements WorkflowStep {
    private $kafkaConsumer;
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $topic = $container->get('kafka_topic');
        
        // Consume messages con timeout
        $messages = [];
        $timeout = 5000;  // 5 segundos
        
        while ($message = $this->kafkaConsumer->consume($topic, $timeout)) {
            $messages[] = json_decode($message->payload, true);
        }
        
        $container->set('kafka_messages', $messages);
        $control->attachStepInfo("Consumed messages from Kafka", ['count' => count($messages)]);
    }
    
    public function getDescription(): string { return "Kafka message consumer"; }
}
```

### 6.4 Integration con Elasticsearch

```php
class ElasticsearchStep implements WorkflowStep {
    private $elasticsearchClient;
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $index = $container->get('es_index', 'workflows');
        $query = $container->get('es_query', ['match_all' => []]);
        
        try {
            $results = $this->elasticsearchClient->search([
                'index' => $index,
                'body' => $query,
            ]);
            
            $hits = array_map(fn($h) => $h['_source'], $results['hits']['hits']);
            $container->set('search_results', $hits);
            
            $control->attachStepInfo("Elasticsearch query", [
                'total_hits' => $results['hits']['total']['value'],
                'returned' => count($hits),
            ]);
        } catch (Exception $e) {
            $control->failStep("Elasticsearch error: " . $e->getMessage());
        }
    }
    
    public function getDescription(): string { return "Elasticsearch query"; }
}
```

---

## 7. Machine Learning Integration

### 7.1 ML Pipeline

```php
class MLPredictionStep implements WorkflowStep {
    private $mlModel;  // Loaded model, scikit-learn via Python API, etc
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $features = $container->get('features');  // Numerical array
        
        try {
            // Invoke ML model
            $prediction = $this->mlModel->predict($features);
            $confidence = $this->mlModel->predictProba($features);
            
            $container->set('prediction', $prediction);
            $container->set('confidence', $confidence);
            
            $control->attachStepInfo("ML prediction", [
                'predicted_class' => $prediction,
                'confidence_score' => $confidence,
            ]);
        } catch (Exception $e) {
            $control->failStep("ML prediction failed: " . $e->getMessage());
        }
    }
    
    public function getDescription(): string { return "ML model prediction"; }
}
```

### 7.2 Feature Engineering

```php
class FeatureEngineeringStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $rawData = $container->get('raw_data');
        
        // Feature extraction
        $features = [
            'numeric_field_normalized' => ($rawData['numeric'] - 100) / 50,
            'categorical_encoded' => $this->encodeCategory($rawData['category']),
            'text_embedding' => $this->embedText($rawData['text']),
            'time_features' => $this->extractTimeFeatures($rawData['timestamp']),
        ];
        
        $container->set('features', $features);
        $control->attachStepInfo("Features engineered", ['feature_count' => count($features)]);
    }
    
    private function encodeCategory($category) { /* ... */ }
    private function embedText($text) { /* ... */ }
    private function extractTimeFeatures($timestamp) { /* ... */ }
    
    pub function getDescription(): string { return "Feature engineering"; }
}
```

---

## 8. Resumen: Características Opcionales

| Característica | Caso de Uso | Complejidad |
|----------------|-----------|-----------|
| Persistencia | Long-running workflows | Media |
| Checkpoints | Fault recovery | Media |
| Event Streaming | Real-time reactions | Alta |
| Generators | Large data processing | Media |
| Kafka/Queue | Distributed systems | Alta |
| Elasticsearch | Search & analytics | Media |
| ML Models | Predictions & scoring | Alta |
| GraphQL | API querying | Baja |

---

## 9. Próximos Pasos

Para maximizar php-workflow:
1. **Persiste**: Implementa save/resume para workflows críticos
2. **Fluye**: Agrega event listeners para monitoreo real-time
3. **Escala**: Usar generators para datos masivos
4. **Integra**: Conecta con sistemas externos (Kafka, ES, etc)
5. **Inteligencia**: Agrega ML predictions en steps
6. **Monitorea**: Importa custom event handlers y tracking
