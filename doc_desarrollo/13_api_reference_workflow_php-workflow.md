# 13. Referencia API Completa de php-workflow

## 1. Objetivo

Proporcionar una referencia técnica completa de todas las clases, interfaces, métodos y parámetros principales de php-workflow, actuando como manual de consulta rápida para desarrolladores.

---

## 2. Contexto

Cuando necesitas recordar:
- **Exactamente qué parámetros** acepta cada método
- **Qué retorna** cada función
- **Qué excepciones** lanza
- **Restricciones** y comportamientos especiales
- **Contratos** implícitos

Este documento es tu referencia.

---

## 3. Workflow: Orquestador Principal

### 3.1 Constructor

```php
public function __construct(
    string $name,
    ...$middlewares
)
```

**Parámetros**:
- `$name`: Identificador único del workflow (ej: "checkout", "user-registration")
  - Usado en logs, reportes, debugging
  - No debe ser null o vacío
  - Recomendado: kebab-case (payment-process)

- `...$middlewares`: (Variadic) Middleware a aplicar a TODOS los steps
  - Cada uno debe ser callable: `function(WorkflowStep): WorkflowStep`
  - Se aplican en orden LIFO (último agregado, primero ejecutado)
  - Opcionales, pueden omitirse

**Ejemplos**:
```php
$w1 = new Workflow('payment');
$w2 = new Workflow('checkout', new ProfileStep());
$w3 = new Workflow('process', $profile, $logger, $validator);
```

**Retorna**: Instancia de `Workflow` (para encadenamiento)

### 3.2 Métodos de Encadenamiento de Etapas

Todos retornan `$this` para method chaining exceptuando `executeWorkflow()`.

#### prepare(WorkflowStep $step, ?bool $hard = null): Workflow

**Etapa**: **Prepare** - Preparación inicial

```php
$workflow->prepare(new LoadDataStep());
```

**Parámetros**:
- `$step`: Implementación de `WorkflowStep`
- `$hard`: (PHP8+ con Requires) Si `true`, se valida antes de ejecutar

**Comportamiento**:
- Se ejecuta primero, antes de validaciones
- No afecta flujo si falla (solo lo reporta)
- Acceso a container compartido

#### validate(WorkflowStep $step, bool $hard = false): Workflow

**Etapa**: **Validate** - Validación

```php
$workflow->validate(new EmailValidator(), true);  // hard validator
$workflow->validate(new PasswordValidator());     // soft validator
```

**Parámetros**:
- `$step`: Validador
- `$hard`: Si `true`, ejecuta primero (se valida antes que soft)
  - Hard validators siempre antes que soft
  - Aplicado en orden agregado

**Comportamiento**:
- Si falla → `WorkflowValidationException` con todos los errores
- Para hard validator: solo 1 fallo y detiene
- Para soft validator: recoge todos, reporta al final

#### before(WorkflowStep $step): Workflow

**Etapa**: **Before** - Antes del procesamiento

```php
$workflow->before(new AuthorizeStep())->before(new LogStartStep());
```

**Parámetros**:
- `$step`: Step a ejecutar

**Comportamiento**:
- Se ejecuta después de Pass/Validate, antes de Process
- Si falla → detiene workflow, va a OnError
- Punto de no retorno NO

#### process(WorkflowStep $step): Workflow

**Etapa**: **Process** - Procesamiento principal

```php
$workflow->process(new ChargePaymentStep())->process(new CreateOrderStep());
```

**Parámetros**:
- `$step`: Step lógica principal

**Comportamiento**:
- Múltiples process steps ejecutados en secuencia
- Si falla → va a OnError
- PUNTO DE NO RETORNO: si Process empieza, OnSuccess SIEMPRE ejecuta
- Si está en Process cuando falla, OnError y After sí ejecutan

#### onSuccess(WorkflowStep $step): Workflow

**Etapa**: **OnSuccess** - Ejecuta si NO hubo exception

```php
$workflow->onSuccess(new SendEmailStep());
$workflow->onSuccess(new LogSuccessStep());
```

**Parámetros**:
- `$step`: Step a ejecutar si éxito

**Comportamiento**:
- Solo si el workflow fue exitoso hasta OnSuccess
- Si falla → se salta, va a OnError

#### onError(WorkflowStep $step): Workflow

**Etapa**: **OnError** - Ejecuta si hubo exception

```php
$workflow->onError(new RollbackStep());
$workflow->onError(new NotifyFailureStep());
```

**Parámetros**:
- `$step`: Step a ejecutar si error

**Comportamiento**:
- Solo si hubo excepción no capturada
- Multiple onError steps en secuencia
- Si OnError falla → su excepción reemplaza la original

#### after(WorkflowStep $step): Workflow

**Etapa**: **After** - Limpieza (siempre se ejecuta)

```php
$workflow->after(new CleanupStep())->after(new LogMetricsStep());
```

**Parámetros**:
- `$step`: Step de limpieza

**Comportamiento**:
- Siempre se ejecuta (success, error, o cualquier path)
- Si el workflow falló, After todavía ejecuta
- Excepciones en After no previenen ejecución de otros After steps

### 3.3 executeWorkflow()

```php
public function executeWorkflow(
    ?WorkflowContainer $container = null,
    bool $throwOnFailure = true,
    array|callable $middleware = []
): WorkflowResult
```

**Parámetros**:
- `$container`: (Opcional) Contenedor de datos compartidos
  - Si null → se crea uno vacío
  - Modificaciones dentro persisten en el resultado
  - Tipo: `WorkflowContainer`

- `$throwOnFailure`: (Default: true)
  - Si `true` → lanza `WorkflowException` si falla
  - Si `false` → retorna `WorkflowResult` con `success() === false`

- `$middleware`: (Default: [])
  - Array o callable individual
  - Se aplica SOLO a esta ejecución
  - Se combina con middleware del constructor

**Retorna**: `WorkflowResult`

**Lanza**:
- `WorkflowException` si `$throwOnFailure === true` y hay error
- Nunca lanza otras excepciones (las captura)

**Ejemplo**:
```php
// Modo excepción
try {
    $result = $workflow->executeWorkflow($container);
    // success guaranteed
} catch (WorkflowException $e) {
    $result = $e->getWorkflowResult();
}

// Modo no-lanzar
$result = $workflow->executeWorkflow($container, false);
if (!$result->success()) {
    // Handle error
}
```

---

## 4. WorkflowStep: Interfaz de Steps

### 4.1 Definición

```php
interface WorkflowStep extends Describable
{
    public function run(
        WorkflowControl $control,
        WorkflowContainer $container
    ): void;
}
```

### 4.2 run() - Método Principal

**Firma**:
```php
public function run(
    WorkflowControl $control,
    WorkflowContainer $container
): void
```

**Parámetros**:
- `$control`: Interfaz para controlar flujo del workflow
- `$container`: Datos compartidos entre steps

**Comportamiento**:
- Ejecutado por el workflow
- Retorna `void` (sin valor de retorno)
- Comunica estado via `$control` (skip, fail, etc.)
- Modifica estado via `$container->set()`
- Puede lanzar excepciones (serán capturadas)

**Contrato Implícito**:
- Debe ser **stateless** (sin estado interno que cambie)
- Debe ser **determinístico** (mismo input → mismo output)
- Debe estar **acoplado débilmente** (inyecta dependencias)
- Debe ser **rápido** (evita operaciones costosas)

### 4.3 getDescription() - Herencia de Describable

```php
public function getDescription(): string
```

**Retorna**: Descripción legible del step (ej: "Validate email format")

**Usado en**:
- Logs de ejecución
- Reportes
- Debugging

**Ejemplo**:
```php
class EmailValidator implements WorkflowStep {
    public function getDescription(): string {
        return "Validate email format";
    }
    
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // ...
    }
}
```

### 4.4 Ejemplo: Implementación Mínima

```php
class SimpleStep implements WorkflowStep
{
    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        // Accede a datos
        $value = $container->get('input', null);
        
        // Valida
        if (!$value) {
            $control->failStep('Input required');
            return;
        }
        
        // Procesa
        $result = $this->process($value);
        
        // Guarda
        $container->set('output', $result);
        
        // Log
        $control->attachStepInfo("Processed successfully");
    }
    
    public function getDescription(): string
    {
        return "Simple processing step";
    }
    
    private function process($value) { /* ... */ }
}
```

---

## 5. WorkflowControl: Interfaz de Control

### 5.1 Métodos de Skip/Fail

#### skipStep(string $reason = ''): never

```php
$control->skipStep('User not premium');
```

**Parámetros**:
- `$reason`: Razón del skip (opcional)

**Comportamiento**:
- Lanza `SkipStepException`
- El paso actual se marca como "skipped"
- Workflow continúa al siguiente step
- Se registra en logs

#### failStep(string $reason = ''): never

```php
$control->failStep('Email validation failed');
```

**Parámetros**:
- `$reason`: Mensaje de error

**Comportamiento**:
- Lanza `FailStepException`
- El workflow falla
- Va a etapa `OnError`
- Se registra como punto de fallo

#### failWorkflow(string $reason = ''): never

```php
$control->failWorkflow('Critical error - aborting');
```

**Parámetros**:
- `$reason`: Mensaje de error

**Comportamiento**:
- Lanza `FailWorkflowException`
- ABORTA completamente el workflow
- Va directamente a `After` (no OnError)
- Útil para errores irrecuperables

#### skipWorkflow(string $reason = ''): never

```php
$control->skipWorkflow('Condition not met');
```

**Parámetros**:
- `$reason`: Razón del skip

**Comportamiento**:
- Lanza `SkipWorkflowException`
- Salta rest del workflow
- Va a `After`
- Se marca como exitoso

### 5.2 Métodos de Loop Control (En Loops únicamente)

#### continue(string $reason = ''): never

```php
if ($count < 5) {
    $control->continue('Continue to next iteration');
}
```

**Parámetros**:
- `$reason`: Razón (para logs)

**Comportamiento**:
- Lanza `ContinueException`
- Válido SOLO dentro de Loop
- Se continúa a siguiente iteración

#### break(string $reason = ''): never

```php
if ($count >= 5) {
    $control->break('Reached max iterations');
}
```

**Parámetros**:
- `$reason`: Razón

**Comportamiento**:
- Lanza `BreakException`
- Válido SOLO dentro de Loop
- Sale del loop, continúa después

### 5.3 Métodos de Información

#### attachStepInfo(string $info, array $context = []): void

```php
$control->attachStepInfo('User email verified', [
    'user_id' => 123,
    'verification_method' => 'code',
]);
```

**Parámetros**:
- `$info`: Mensaje informativo
- `$context`: (Opcional) Contexto adicional

**Comportamiento**:
- Se registra en logs
- NO afecta flujo
- Útil para debugging

#### warning(string $message, ?Exception $exception = null): void

```php
$control->warning('Retry attempt failed', new TimeoutException());
```

**Parámetros**:
- `$message`: Mensaje de warning
- `$exception`: (Opcional) Excepción asociada

**Comportamiento**:
- Se registra como warning (no error)
- Workflow continúa
- Se agrupa por etapa en resultado

---

## 6. WorkflowContainer: Almacén de Datos

### 6.1 Métodos Principales

#### get(string $key, mixed $default = null): mixed

```php
$value = $container->get('user');
$optional = $container->get('metadata', []);  // Con default
```

**Parámetros**:
- `$key`: Clave a buscar
- `$default`: Valor si no existe (default: null)

**Retorna**: Valor stored o default

#### set(string $key, mixed $value): self

```php
$container->set('user', $user)
    ->set('status', 'pending')
    ->set('metrics', $data);  // Chainable
```

**Parámetros**:
- `$key`: Clave
- `$value`: Cualquier valor de PHP (mixed)

**Retorna**: `$this` (para encadenamiento)

**Aceptados**: Tipos primitivos, objetos, arrays

#### has(string $key): bool

```php
if ($container->has('user')) {
    // Key existe
}
```

**Parámetros**:
- `$key`: Clave a verificar

**Retorna**: `true` si existe, `false` si no

#### unset(string $key): self

```php
$container->unset('temporary_data');
```

**Parámetros**:
- `$key`: Clave a remover

**Retorna**: `$this`

**Comportamiento**:
- Silenciosamente falla si key no existe (no lanza error)

#### keys(): array

```php
$allKeys = $container->keys();  // ['user', 'cart', 'status']
```

**Retorna**: Array de todas las claves

### 6.2 Restricciones

**Permitido**:
```php
$container->set('string', 'value');
$container->set('number', 42);
$container->set('array', [1, 2, 3]);
$container->set('object', new stdClass());
$container->set('null', null);
```

**NO permitido**:
```php
$container->set('', 'value');        // Key vacía
$container->set(null, 'value');      // Key null
```

### 6.3 NestedContainer (En NestedWorkflow)

Para nested workflows, usa `NestedContainer`:

```php
// Constructor en NestedWorkflow
$nested = new NestedWorkflow($innerWorkflow, $container);
// Automáticamente crea NestedContainer que hereda del externo
```

**Comportamiento**:
- `get()`: Busca en local primero, luego parent
- `set()`: Escribe en AMBOS (local y parent simultáneamente)
- `has()`: Verifica en ambos

---

## 7. WorkflowResult: Resultado de Ejecución

(Documentado completamente en Document 09, aquí sumario de referencia)

### 7.1 Métodos

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `success()` | bool | ¿Fue exitoso? |
| `getException()` | ?Exception | Excepción que ocurrió |
| `getContainer()` | WorkflowContainer | Estado final |
| `getLastStep()` | WorkflowStep | Último step ejecutado |
| `getWarnings()` | array | Warnings por etapa |
| `hasWarnings()` | bool | ¿Hay warnings? |
| `getWorkflowName()` | string | Nombre del workflow |
| `debug(?OutputFormat)` | mixed | Log formateado |

### 7.2 Ejemplo de Uso Completo

```php
$result = $workflow->executeWorkflow($container, false);

if ($result->success()) {
    $data = $result->getContainer()->get('result');
    return $data;
} else {
    $error = $result->getException();
    throw new ProcessingException($error->getMessage());
}
```

---

## 8. Excepciones: Jerarquía Completa

### 8.1 Jerarquía

```
Exception
├─ WorkflowException (base, wraps WorkflowResult)
├─ WorkflowValidationException
│   └─ getValidationErrors(): ValidationError[]
├─ WorkflowStepDependencyNotFulfilledException
├─ WorkflowControl\ControlException (base)
│   ├─ SkipStepException
│   ├─ FailStepException
│   ├─ FailWorkflowException
│   ├─ SkipWorkflowException
│   ├─ LoopControlException
│   │   ├─ ContinueException
│   │   └─ BreakException
│   └─ ... otros
```

### 8.2 Cuándo se Lanza Cada una

| Excepción | Lanzada por | Comportamiento |
|-----------|-------------|----------------|
| WorkflowException | Workflow si throwOnFailure=true | Wraps WorkflowResult |
| WorkflowValidationException | Validate stage | Validators fallidos |
| SkipStepException | control->skipStep() | Step skipped |
| FailStepException | control->failStep() | Step fallo |
| FailWorkflowException | control->failWorkflow() | Workflow abortado |

---

## 9. Fases de Ejecución: Tabla de Referencia

| Fase | Orden | Ejecuta Si | Si Falla |
|------|-------|-----------|---------|
| Prepare | 1 | Siempre | Info (no detiene) |
| Validate | 2 | Siempre | ValidationException |
| Before | 3 | Validó OK | Detiene, va OnError |
| Process | 4 | Antes OK | Detiene, va OnError |
| OnSuccess | 5 | No hubo fallo | Se salta |
| OnError | 6 | Hubo fallo | Exc. reemplaza |
| After | 7 | Siempre | Sin detener |

---

## 10. Ciclo de Vida Completo

```
┌─── Workflow Start
│
├─► Prepare [1..n]
│   └─ Excepción → Info, continúa
│
├─► Validate [Hard → Soft]
│   └─ Falla → WorkflowValidationException
│
├─► Before [1..n]
│   └─ Falla → OnError
│
├─► Process [1..n]  ◄─── PUNTO DE NO RETORNO
│   └─ Falla → OnError + (OnSuccess SALTA)
│
├─► OnSuccess [1..n] ◄─── Si éxito
│   └─ Falla → Reemplaza excepción
│
├─ OnError [1..n]  ◄─── Si fallo
│   └─ Falla → Nueva excepción
│
└─► After [1..n]
    └─ Siempre se ejecuta
    └─ Excepciones silenciadas
```

---

## 11. Convenciones y Best Practices

### 11.1 Naming Conventions

```php
// Workflows
new Workflow('checkout-process');  // kebab-case
new Workflow('user-registration');

// Steps
class ValidateEmailStep implements WorkflowStep { }
class CreateOrderStep implements WorkflowStep { }

// Methods
$workflow->prepare(...)
    ->validate(...)
    ->process(...);

// Keys en container
$container->set('user_id', $userId);
$container->set('order_items', $items);
```

### 11.2 Error Messages

```php
// Claro y específico
$control->failStep('Email already registered');     // ✅
$control->failStep('ERROR');                        // ❌

// Con contexto
$control->attachStepInfo('Processing', [
    'user_id' => $id,
    'status' => 'pending'
]);
```

---

## 12. Referencia Rápida: Métodos por Categoría

### Workflow
- `__construct(string, ...$middleware)`
- `prepare(WorkflowStep)`
- `validate(WorkflowStep, bool)`
- `before(WorkflowStep)`
- `process(WorkflowStep)`
- `onSuccess(WorkflowStep)`
- `onError(WorkflowStep)`
- `after(WorkflowStep)`
- `executeWorkflow(?Container, bool, array|callable)`

### WorkflowStep
- `run(WorkflowControl, WorkflowContainer): void`
- `getDescription(): string`

### WorkflowControl
- `skipStep(string)`
- `failStep(string)`
- `failWorkflow(string)`
- `skipWorkflow(string)`
- `continue(string)` (en loop)
- `break(string)` (en loop)
- `attachStepInfo(string, array)`
- `warning(string, ?Exception)`

### WorkflowContainer
- `get(string, mixed): mixed`
- `set(string, mixed): self`
- `has(string): bool`
- `unset(string): self`
- `keys(): array`

### WorkflowResult
- `success(): bool`
- `getException(): ?Exception`
- `getContainer(): WorkflowContainer`
- `getLastStep(): WorkflowStep`
- `getWarnings(): array`
- `hasWarnings(): bool`
- `getWorkflowName(): string`
- `debug(?OutputFormat): mixed`

---

## 13. Archivo Locations

Ubicaciones de clases principales en el repositorio:

- `Workflow`: [src/Workflow.php](../src/Workflow.php)
- `WorkflowStep`: [src/Step/WorkflowStep.php](../src/Step/WorkflowStep.php)
- `WorkflowControl`: [src/WorkflowControl.php](../src/WorkflowControl.php)
- `WorkflowContainer`: [src/State/WorkflowContainer.php](../src/State/WorkflowContainer.php)
- `WorkflowResult`: [src/State/WorkflowResult.php](../src/State/WorkflowResult.php)
- `Excepciones`: [src/Exception/](../src/Exception/)

