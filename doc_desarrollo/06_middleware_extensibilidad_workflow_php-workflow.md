# 6. Middleware y Extensibilidad en php-workflow

## 1. Objetivo

Documentar el sistema de middleware que permite interceptar, monitorear y extender la ejecución de cada step en un workflow, proporcionando un mecanismo flexible y poderoso para agregar comportamiento transversal sin modificar la lógica de los pasos.

---

## 2. Contexto

A menudo necesitas ejecutar lógica **alrededor de cada paso**: 

- Medir timing de ejecución
- Validar precondiciones
- Registrar acciones
- Aplicar transacciones de BD
- Implementar rate limiting
- Cachear resultados

php-workflow proporciona un **sistema de middleware** que permite aplicar esta lógica a TODOS los steps de un workflow de forma centralizada, sin tocar el código de los pasos.

El concepto es **interceptación en cadena**: cada middleware puede ejecutar lógica antes del step, después del step, y/o alrededor de la ejecución.

---

## 3. Concepto Fundamental: Middleware como Interceptor

### 3.1 El Patrón Básico

Un middleware es un **callable** que:

1. Recibe el siguiente en la cadena (`$next`)
2. Ejecuta lógica PRE-step si necesita
3. Llama `$next()` para pasar al siguiente
4. Ejecuta lógica POST-step si necesita
5. Retorna resultado

```php
// Estructura general de un middleware
$middleware = function(callable $next, ...$params) {
    // Lógica ANTES
    $result = $next();  // Ejecuta siguiente middleware o step
    // Lógica DESPUÉS
    return $result;
};
```

### 3.2 LIFO: Last In, First Out

Los middlewares se construyen en orden **LIFO** (inverso):

```
Definidos: [Middleware1, Middleware2, Middleware3]

Ejecución chain:
  Middleware1
    → Middleware2
      → Middleware3
        → Step.run()
        ← retorna
      ← retorna
    ← retorna
  ← retorna
```

**Orden de ejecución PRE**:
1. Middleware1 PRE
2. Middleware2 PRE
3. Middleware3 PRE
4. Step.run()

**Orden de ejecución POST**:
1. Middleware3 POST
2. Middleware2 POST
3. Middleware1 POST

---

## 4. Constructor de Workflow y Parámetros

Ubicación: [`src/Workflow.php`](../src/Workflow.php)

```php
class Workflow extends Stage
{
    private array $middleware;

    /**
     * @param string $name - Nombre del workflow
     * @param callable ...$middlewares - Middlewares a aplicar
     */
    public function __construct(string $name, callable ...$middlewares)
    {
        $this->name = $name;
        $this->middleware = $middlewares;  // Se almacenan
    }
}
```

### 4.1 Uso Basic

```php
// Sin middlewares
new Workflow('my-workflow');

// Con un middleware
new Workflow('my-workflow', new ProfileStep());

// Con múltiples middlewares
new Workflow('my-workflow', 
    new ProfileStep(),           // Primero a ejecutar POST
    new CustomValidation(),      // Segundo a ejecutar POST
    new LoggingMiddleware(),     // Tercero a ejecutar POST
);
```

### 4.2 Registro del Middleware

Durante ejecución, se pasa al WorkflowState:

```php
protected function runStage(WorkflowState $workflowState): ?Stage
{
    // ...
    $workflowState->setMiddlewares($this->middleware);  // Se registra aquí
    // ...
}
```

---

## 5. Construcción de la Cadena de Middleware

Ubicación: [`src/Step/StepExecutionTrait.php`](../src/Step/StepExecutionTrait.php)

```php
private function resolveMiddleware(WorkflowStep $step, WorkflowState $workflowState): callable
{
    // Comienza con el tip (final) - la ejecución del step
    $tip = fn () => $step->run(
        $workflowState->getWorkflowControl(), 
        $workflowState->getWorkflowContainer()
    );

    // Obtiene los middlewares registrados
    $middlewares = $workflowState->getMiddlewares();

    // En PHP 8+, agrega automáticamente validación de dependencias
    if (PHP_MAJOR_VERSION >= 8) {
        array_unshift($middlewares, new WorkflowStepDependencyCheck());
    }

    // Construye la cadena wrapping LIFO
    foreach ($middlewares as $middleware) {
        $tip = fn () => $middleware(
            $tip,                                           // $next
            $workflowState->getWorkflowControl(),          // $control
            $workflowState->getWorkflowContainer(),        // $container
            $step,                                          // $step
        );
    }

    return $tip;  // Retorna la cadena completa
}
```

### 5.1 Lógica de Construcción

```
1. Inicial:
   $tip = function() => $step->run(...)

2. Primera iteración (Middleware1):
   $tip = function() => Middleware1($tip, $control, $container, $step)

3. Segunda iteración (Middleware2):
   $tip = function() => Middleware2($tip, $control, $container, $step)
   // Nota: $tip ahora es Middleware1 wrapping

4. Tercera iteración (Middleware3):
   $tip = function() => Middleware3($tip, $control, $container, $step)
   // Nota: $tip ahora es Middleware2 > Middleware1 wrapping

5. Final (aplicar cadena):
   $tip()  // Ejecuta Middleware3 > Middleware2 > Middleware1 > step.run()
```

### 5.2 Parámetros que Recibe Cada Middleware

```php
$middleware = function(
    callable $next,
    WorkflowControl $control,
    WorkflowContainer $container,
    WorkflowStep $step,
) {
    // $next - Callable para el siguiente middleware o step
    // $control - Control para failStep, warning, etc.
    // $container - Contenedor de datos
    // $step - El step siendo ejecutado
};
```

---

## 6. Middleware Builtin: ProfileStep

Ubicación: [`src/Middleware/ProfileStep.php`](../src/Middleware/ProfileStep.php)

**Propósito**: Medir el tiempo de ejecución de cada step.

### 6.1 Implementación

```php
class ProfileStep
{
    public function __invoke(callable $next, WorkflowControl $control)
    {
        // Inicia temporizador
        $start = microtime(true);
        
        // Función para adjuntar timing
        $profile = fn () => $control->attachStepInfo(
            "Step execution time: " . number_format(1000 * (microtime(true) - $start), 5) . 'ms',
        );

        try {
            // Ejecuta el siguiente (step u otro middleware)
            $result = $next();
            
            // Adjunta timing en caso de éxito
            $profile();
        } catch (Exception $exception) {
            // Adjunta timing INCLUSO en caso de excepción
            $profile();
            
            // Re-lanza la excepción
            throw $exception;
        }

        return $result;
    }
}
```

### 6.2 Comportamiento

- **Mide**: Tiempo transcurrido entre `$start` y fin de ejecución
- **Adjunta**: Como `StepInfo` en el log
- **Captura**: Excepciones y sigue midiendo
- **Impacto**: Cada step muestra "Step execution time: XXX.XXXXXms"

### 6.3 Uso

```php
$workflow = new Workflow('my-process', new ProfileStep());

// Todas los steps tendrán timing medido
$workflow->process(new MyStep());
$workflow->after(new CleanupStep());

$result = $workflow->executeWorkflow();
$debug = $result->debug();
// Verás en el debug:
// MyStep: ok
//   - Step execution time: 12.34567ms
// CleanupStep: ok
//   - Step execution time: 2.34567ms
```

---

## 7. Middleware Builtin: WorkflowStepDependencyCheck (PHP 8+)

Ubicación: [`src/Middleware/WorkflowStepDependencyCheck.php`](../src/Middleware/WorkflowStepDependencyCheck.php)

**Propósito** (solo PHP 8+): Validar que el contenedor tiene todas las dependencias requeridas antes de ejecutar un step.

### 7.1 Concepto de Dependencias

Los steps pueden declarar dependencias en sus parámetros usando atributos:

```php
// Step que REQUIERE 'user_id' en el contenedor
class ProcessUserStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('user_id', 'int')]  // Requiere user_id (tipo int)
        WorkflowContainer $container
    ): void {
        // Garantizado que $container->get('user_id') existe y es int
        $userId = $container->get('user_id');
        // ... procesa ...
    }
}
```

### 7.2 Implementación

```php
class WorkflowStepDependencyCheck
{
    public function __invoke(
        callable $next,
        WorkflowControl $control,
        WorkflowContainer $container,
        WorkflowStep $step,
    ) {
        // Obtiene el método 'run' del step
        $containerParameter = (new ReflectionMethod($step, 'run'))->getParameters()[1] ?? null;

        if ($containerParameter) {
            // Busca atributos de dependencia en el parámetro $container
            foreach ($containerParameter->getAttributes(
                    StepDependencyInterface::class,
                    ReflectionAttribute::IS_INSTANCEOF,
                ) as $dependencyAttribute
            ) {
                // Instancia el atributo
                /** @var StepDependencyInterface $dependency */
                $dependency = $dependencyAttribute->newInstance();
                
                // Ejecuta validación
                $dependency->check($container);
                // Si falla, lanza WorkflowStepDependencyNotFulfilledException
            }
        }

        // Si todas las dependencias están OK, ejecuta el step
        return $next();
    }
}
```

### 7.3 Funcionamiento Automático

En `resolveMiddleware`, si PHP 8+:

```php
if (PHP_MAJOR_VERSION >= 8) {
    array_unshift($middlewares, new WorkflowStepDependencyCheck());
}
```

**Resultado**: Se agrega automáticamente como **primer middleware** en la cadena. Valida ANTES de cualquier otro middleware o del step.

---

## 8. Sistema de Dependencias

### 8.1 Interfaz: StepDependencyInterface

Ubicación: [`src/Step/Dependency/StepDependencyInterface.php`](../src/Step/Dependency/StepDependencyInterface.php)

```php
interface StepDependencyInterface
{
    /**
     * @throws WorkflowStepDependencyNotFulfilledException
     */
    public function check(WorkflowContainer $container): void;
}
```

Contrato simple: valida o lanza excepción.

### 8.2 Implementación Builtin: Requires

Ubicación: [`src/Step/Dependency/Requires.php`](../src/Step/Dependency/Requires.php)

**Propósito**: Validar que una clave existe en el contenedor y tiene el tipo correcto.

```php
#[Attribute(Attribute::TARGET_PARAMETER | Attribute::IS_REPEATABLE)]
class Requires implements StepDependencyInterface
{
    public function __construct(
        private string $key,           // Clave en contenedor
        private ?string $type = null   // Tipo esperado (opcional)
    ) {}

    public function check(WorkflowContainer $container): void
    {
        // 1. Verifica que la clave existe
        if (!$container->has($this->key)) {
            throw new WorkflowStepDependencyNotFulfilledException(
                "Missing '$this->key' in container"
            );
        }

        // Si no se especificó tipo, la validación termina aquí
        if ($this->type === null) {
            return;
        }

        $value = $container->get($this->key);

        // 2. Permite null si el tipo es opcional (?type)
        if (str_starts_with($this->type, '?') && $value === null) {
            return;
        }

        // 3. Valida tipo primitivo
        $type = str_replace('?', '', $this->type);
        
        if (preg_match('/^(string|bool|int|float|object|array|iterable|scalar)$/', $type)) {
            $checkMethod = 'is_' . $type;
            if ($checkMethod($value)) {
                return;
            }
        }
        // 4. Valida clase
        elseif (class_exists($type) && ($value instanceof $type)) {
            return;
        }

        // No pasó validación
        throw new WorkflowStepDependencyNotFulfilledException(
            sprintf(
                "Value for '%s' has invalid type. Expected %s, got %s",
                $this->key,
                $this->type,
                gettype($value) . (is_object($value) ? " ({$value::class})" : ''),
            ),
        );
    }
}
```

### 8.3 Uso de Requires

```php
class ProcessOrderStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('order_id', 'int')]        // Requiere order_id int
        #[Requires('user_id', '?int')]        // user_id int o null
        #[Requires('payment_method', 'string')] // Requiere payment_method string
        #[Requires('processor', 'PaymentProcessor')] // Instancia de PaymentProcessor
        WorkflowContainer $container
    ): void {
        // Todos garantizados de existir y tener tipos correctos
        $orderId = $container->get('order_id');    // int
        $userId = $container->get('user_id');      // int|null
        $paymentMethod = $container->get('payment_method');  // string
        $processor = $container->get('processor');  // PaymentProcessor
    }
}
```

### 8.4 Validación de Tipos Soportados

**Primitivos**: `string`, `bool`, `int`, `float`, `object`, `array`, `iterable`, `scalar`

**Clases**: Cualquier nombre de clase existente

**Opcionales**: Prefijo `?` permite null

```php
#[Requires('optional_field', '?string')]
#[Requires('amount', '?float')]
```

---

## 9. Crear Middleware Personalizado

### 9.1 Patrón: Middleware Simple

```php
class MyMiddleware
{
    public function __invoke(
        callable $next,
        WorkflowControl $control,
        WorkflowContainer $container,
        WorkflowStep $step,
    ) {
        // Lógica PRE-step
        echo "Ejecutando: " . $step->getDescription() . "\n";
        
        // Ejecuta el siguiente
        $result = $next();
        
        // Lógica POST-step
        echo "Terminado: " . $step->getDescription() . "\n";
        
        return $result;
    }
}

// Uso:
$workflow = new Workflow('test', new MyMiddleware());
```

### 9.2 Ejemplo: Transacciones DB

```php
class DatabaseTransaction
{
    public function __construct(private PDO $pdo) {}

    public function __invoke(
        callable $next,
        WorkflowControl $control,
        WorkflowContainer $container,
        WorkflowStep $step,
    ) {
        // Inicia transacción
        $this->pdo->beginTransaction();

        try {
            $result = $next();
            
            // Commit si éxito
            $this->pdo->commit();
            
            return $result;
        } catch (Exception $e) {
            // Rollback si error
            $this->pdo->rollBack();
            throw $e;
        }
    }
}

// Uso:
$workflow = new Workflow('payment-process', new DatabaseTransaction($pdo));
```

### 9.3 Ejemplo: Logging Custom

```php
class LoggingMiddleware
{
    public function __construct(private Logger $logger) {}

    public function __invoke(
        callable $next,
        WorkflowControl $control,
        WorkflowContainer $container,
        WorkflowStep $step,
    ) {
        $stepName = $step->getDescription();
        
        $this->logger->info("Step starting: $stepName");
        $start = microtime(true);

        try {
            $result = $next();
            
            $elapsed = microtime(true) - $start;
            $this->logger->info("Step completed: $stepName in {$elapsed}s");
            
            return $result;
        } catch (Exception $e) {
            $this->logger->error("Step failed: $stepName - " . $e->getMessage());
            throw $e;
        }
    }
}
```

### 9.4 Ejemplo: Rate Limiting

```php
class RateLimiter
{
    private array $callCounts = [];
    private int $maxCalls;
    private int $windowSeconds;

    public function __construct(int $maxCalls = 10, int $windowSeconds = 60)
    {
        $this->maxCalls = $maxCalls;
        $this->windowSeconds = $windowSeconds;
    }

    public function __invoke(
        callable $next,
        WorkflowControl $control,
        WorkflowContainer $container,
        WorkflowStep $step,
    ) {
        $stepName = $step->getDescription();
        $now = time();

        // Limpia llamadas antiguas del contador
        if (!isset($this->callCounts[$stepName])) {
            $this->callCounts[$stepName] = [];
        }

        $this->callCounts[$stepName] = array_filter(
            $this->callCounts[$stepName],
            fn($time) => ($now - $time) < $this->windowSeconds
        );

        // Verifica límite
        if (count($this->callCounts[$stepName]) >= $this->maxCalls) {
            $control->failStep("Rate limit exceeded for $stepName");
        }

        // Registra esta llamada
        $this->callCounts[$stepName][] = $now;

        return $next();
    }
}

// Uso:
$workflow = new Workflow(
    'api-calls',
    new RateLimiter(maxCalls: 100, windowSeconds: 60)  // 100 calls per minute
);
```

### 9.5 Ejemplo: Caching

```php
class CachingMiddleware
{
    private array $cache = [];

    public function __invoke(
        callable $next,
        WorkflowControl $control,
        WorkflowContainer $container,
        WorkflowStep $step,
    ) {
        $cacheKey = $step->getDescription() . ':' . md5(json_encode($container));

        // Verifica cache
        if (isset($this->cache[$cacheKey])) {
            $control->attachStepInfo("Result from cache");
            return $this->cache[$cacheKey];
        }

        // Ejecuta step
        $result = $next();

        // Guarda en cache
        $this->cache[$cacheKey] = $result;

        return $result;
    }
}
```

---

## 10. Middleware en Loops

Los middlewares se aplican a **cada iteración** del loop.

### 10.1 Comportamiento

```php
$loop = new Loop($control, continueOnError: false);
$loop->addStep($myStep);

$workflow = new Workflow('test', new ProfileStep());
$workflow->process($loop);
```

**Ejecución**:

```
Loop::run()
  ├─ Iteración 1:
  │   └─ ProfileStep PRE
  │       → $myStep.run()
  │       ← ProfileStep POST
  ├─ Iteración 2:
  │   └─ ProfileStep PRE
  │       → $myStep.run()
  │       ← ProfileStep POST
  └─ ...
```

**Cada iteración** construye la cadena nuevamente.

### 10.2 Performance Consideration

Multiple iteraciones = múltiples invocaciones de middleware.

```php
// BIEN - ✓ Middleware ligero
new Workflow('name', new ProfileStep());

// CUIDADO - ⚠️ Si el middleware es pesado
$heavyMiddleware = new ExpensiveValidation();
new Workflow('name', $heavyMiddleware);

// En un loop de 1000 iteraciones, se invoca 1000 veces
```

---

## 11. Middleware en Workflows Anidados

Los middlewares del workflow padre **NO** se aplican al workflow anidado.

### 11.1 Comportamiento

```php
$inner = new Workflow('inner', new ProfileStep());  // ProfileStep aplicado a inner

$outer = new Workflow('outer', new LoggingMiddleware());

$outer->process(
    new NestedWorkflow($inner),
);

// Ejecución de outer:
//   LoggingMiddleware PRE
//     → NestedWorkflow.run()
//       → $inner.executeWorkflow()  // ProfileStep aplicado a inner steps, NO LoggingMiddleware
//     ← LoggingMiddleware POST
```

Cada workflow tiene su propio stack de middlewares.

---

## 12. Tabla de Referencia: Aplicación de Middleware

| Escenario | Middleware Aplicado |
|-----------|-------------------|
| Stage simple | Sí (a cada step) |
| MultiStepStage | Sí (a cada step) |
| Loop (iteración) | Sí (a cada step de cada iteración) |
| NestedWorkflow | No (usa middlewares del nested) |
| Excepciones | Sí (incluso si step falla) |

---

## 13. Orden de Ejecución Completo

Cuando se ejecuta un step en Workflow A con Middleware [M1, M2]:

```
1. resolveMiddleware() construye cadena:
   $tip = M1(M2(step.run, ...), ...)

2. $tip() se ejecuta:
   - M1 PRE-logic
   - M2 PRE-logic
   - step.run()
   - M2 POST-logic
   - M1 POST-logic

3. Si hay excepción en step:
   - M2 POST-logic aún ejecuta (si maneja en try-catch)
   - M1 POST-logic aún ejecuta
   - Excepción propaga hacia arriba

4. Resultado final:
   - Adjunta StepInfo (logging)
   - Retorna o propaga
```

---

## 14. Hallazgos Clave

### 14.1 Middleware Automático en PHP 8+

`WorkflowStepDependencyCheck` se agrega automáticamente en PHP 8+.

**Implicación**: Validación de dependencias sin código extra.

### 14.2 LIFO = Orden Inverso

Middlewares se agregan al final de la cadena → ejecución PRE en orden inverso.

```php
new Workflow('test', M1, M2, M3)

PRE execution order: M3 → M2 → M1 → step
POST execution order: M1 → M2 → M3
```

### 14.3 Excepciones Capturadas

Los middlewares pueden capturar excepciones del siguiente.

```php
try {
    $result = $next();
} catch (SpecificException $e) {
    // Maneja la excepción
    // Puede negar que se propague
}
```

### 14.4 Aplicación Universal

Middlewares se aplican a:
- Todos los steps de todas las etapas
- Cada iteración de loops
- NO se heredan a workflows anidados

### 14.5 Parámetros Siempre Disponibles

Cada middleware recibe:
- `$next`: Callable para siguiente
- `$control`: Para failStep, warning, attachStepInfo
- `$container`: Para acceder/modificar datos
- `$step`: Referencia al step siendo ejecutado

---

## 15. Antipatrones

### ❌ Antipatrón 1: Middleware que Modifica Container Globalmente

```php
// MALO - ✗
class BadMiddleware {
    public function __invoke($next, $control, $container, $step) {
        $container->set('debug', true);  // Afecta todos los steps posteriores
        return $next();
    }
}
```

### ❌ Antipatrón 2: Middleware que No Re-lanza Excepciones

```php
// MALO - ✗
class BadMiddleware {
    public function __invoke($next, $control, $container, $step) {
        try {
            return $next();
        } catch (Exception $e) {
            // Silencia la excepción ✗
        }
    }
}
```

### ❌ Antipatrón 3: Middleware Pesado en Loops

```php
// CUIDADO - ⚠️ En loop de 1000 iteraciones
$expensive = new ExpensiveDatabaseValidation();
new Workflow('name', $expensive);

// Se ejecuta 1000 veces, puede ser lento
```

---

## 16. Ejemplos Prácticos Completos

### 16.1 Workflow con Multiple Middleware

```php
$workflow = new Workflow(
    'payment-processing',
    new ProfileStep(),                    // Mide tiempo
    new DatabaseTransaction($pdo),        // Transacciones
    new LoggingMiddleware($logger),       // Registra
    new RateLimiter(maxCalls: 100),      // Rate limiting
);

$workflow
    ->prepare(new ValidatePaymentData())
    ->process(new ProcessPayment())
    ->onSuccess(new SendReceipt())
    ->onError(new LogError());

$result = $workflow->executeWorkflow($container);
```

Log resultante (imaginario):
```
[LOG] Step starting: Validate Payment Data
[TIMING] Step execution time: 5.23ms
[LOG] Step completed: Validate Payment Data in 0.005s
[LOG] Step starting: Process Payment
[TIMING] Step execution time: 234.56ms
[LOG] Step completed: Process Payment in 0.234s
```

### 16.2 Workflow con Dependencias (PHP 8+)

```php
class ChargeUserStep implements WorkflowStep {
    public function run(
        WorkflowControl $control,
        #[Requires('user_id', 'int')]
        #[Requires('amount', 'float')]
        #[Requires('payment_method', 'string')]
        WorkflowContainer $container
    ): void {
        $userId = $container->get('user_id');
        $amount = $container->get('amount');
        $method = $container->get('payment_method');
        
        // Garantizado que existen y tienen tipos correctos
        // WorkflowStepDependencyCheck validó automaticamente
    }
}

$workflow = new Workflow('charge-user');
$workflow->process(new ChargeUserStep());

// Si falta user_id o no es int:
// → WorkflowStepDependencyNotFulfilledException automáticamente
```

---

## 17. Conclusiones

El sistema de middleware en php-workflow es **flexible y poderoso**:

### 17.1 Fortalezas

- **Reutilizable**: Middleware se comparte entre workflows
- **Separación de concerns**: Lógica transversal separada de pasos
- **Composable**: Múltiples middlewares pueden combinarse
- **Automático en PHP 8+**: Validación de dependencias sin código
- **Tipo-seguro**: Sistema de dependencias con tipos

### 17.2 Patrones Emergentes

- **Transacciones**: DB Transaction middleware
- **Auditoría**: Logging de todas las acciones
- **Performance**: ProfileStep integrado
- **Validación**: Dependencias con atributos
- **Rate limiting**: Protección contra abuso

### 17.3 Responsabilidades del Usuario

- Entender el orden LIFO de ejecución
- Capturar/propagar excepciones correctamente
- Considerar performance en loops
- Documentar middlewares personalizados

### 17.4 Integración Perfecta

- Constructor de Workflow acepta middlewares
- Se aplican a todos los steps automáticamente
- Parámetros siempre disponibles
- Compatible con todos los tipos de steps

---

## 18. Referencias de Código

### Ubicaciones Clave

- Interfaces: `src/Middleware/*.php` - Implementaciones
- Resolución: `src/Step/StepExecutionTrait.php` - Construcción de cadena
- Workflow: `src/Workflow.php` - Parámetros
- Dependencias: `src/Step/Dependency/*.php` - Sistema de validación

### Tests Recomendados

- `tests/WorkflowTest.php::testProfilingMiddleware()` - ProfileStep en acción

---

## 19. Próximas Extensiones

Para mayor funcionalidad:

- **Grupo 6: Dependencias** - Más sobre sistema de validación
- **Grupo 8: Best Practices** - Patrones avanzados
- **Grupo 12: Debugging** - Middleware para debugging

