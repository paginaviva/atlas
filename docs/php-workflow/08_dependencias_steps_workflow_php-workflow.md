# 8. Dependencias Entre Steps en php-workflow

## 1. Objetivo

Documentar el sistema de dependencias de steps que permite declarar y validar precondiciones en los pasos de un workflow, asegurando que los datos necesarios estén disponibles con los tipos correctos antes de que se ejecute cada step.

---

## 2. Contexto

**Disponible solo en PHP 8.0+** (usa Attributes/Decoradores de PHP 8)

Frecuentemente necesitas garantizar que un step reciba datos en estado válido:

- Requiere que ciertos datos existan en el contenedor
- Requiere que ciertos datos sean del tipo correcto
- Requiere validación de precondiciones
- Requiere composición de múltiples dependencias en un step

php-workflow proporciona un **sistema de validación declarativa** que:
- Usa atributos PHP 8 (`#[Requires()]`) en parámetros del método `run()`
- Se valida **automáticamente ANTES de ejecutar el step**
- Proporciona messages claros cuando faltan dependencias
- Integra con middleware de validación
- Falla el step claramente sin ejecutar lógica

---

## 3. Requisito: PHP 8.0+

Ubicación: [`src/Middleware/WorkflowStepDependencyCheck.php`](../src/Middleware/WorkflowStepDependencyCheck.php)

```php
if (PHP_MAJOR_VERSION >= 8) {
    array_unshift($middlewares, new WorkflowStepDependencyCheck());
}
```

**Implicación**: Sistema de dependencias **funciona automáticamente en PHP 8.0+**, no requiere código especial.

---

## 4. Interfaz: StepDependencyInterface

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

Contrato simple:
- **check()** - Valida dependencia contra contenedor
- **Lanza excepción** si no se cumple

---

## 5. Implementación Builtin: Requires

Ubicación: [`src/Step/Dependency/Requires.php`](../src/Step/Dependency/Requires.php)

### 5.1 Declaración

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
        // Valida...
    }
}
```

**Características**:
- `#[Attribute]` - Atributo PHP 8
- `TARGET_PARAMETER` - Se aplica a parámetros de método
- `IS_REPEATABLE` - Permite múltiples `#[Requires()]` en mismo parámetro
- Implementa `StepDependencyInterface`

### 5.2 Uso Básico: Solo Clave (Existencia)

```php
class ProcessOrderStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('order_id')]  // Requiere que exista 'order_id'
        WorkflowContainer $container
    ): void {
        $orderId = $container->get('order_id');
        // Garantizado que $orderId no es null
        // Puede ser cualquier tipo
    }

    public function getDescription(): string { return 'Process order'; }
}
```

**Validación**: Solo verifica que la clave existe en el contenedor.

### 5.3 Uso Tipado: Clave + Tipo

```php
class ChargePaymentStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('amount', 'float')]     // Requiere float
        #[Requires('currency', 'string')]  // Requiere string
        #[Requires('card_token', 'string')] // Requiere string
        WorkflowContainer $container
    ): void {
        $amount = $container->get('amount');      // float garantizado
        $currency = $container->get('currency');  // string garantizado
        $token = $container->get('card_token');   // string garantizado
        
        // Procesa pago...
    }

    public function getDescription(): string { return 'Charge payment'; }
}
```

**Validación**: Verifica existencia AND tipo exacto.

---

## 6. Tipos Soportados

### 6.1 Primitivos

```php
#[Requires('name', 'string')]      // string
#[Requires('age', 'int')]          // int
#[Requires('price', 'float')]      // float
#[Requires('is_active', 'bool')]   // bool
#[Requires('flags', 'array')]      // array
#[Requires('data', 'object')]      // object
#[Requires('items', 'iterable')]   // array o Traversable
#[Requires('value', 'scalar')]     // int, float, string, bool
```

### 6.2 Clases (instanceof)

```php
#[Requires('user', 'User')]              // instanceof User
#[Requires('created', 'DateTime')]       // instanceof DateTime
#[Requires('logger', 'Psr\Log\Logger')] // instanceof Psr\Log\Logger
#[Requires('file', 'SplFileObject')]     // instanceof SplFileObject
```

### 6.3 Tipos Opcionales (con ?)

Prefijo `?` permite valores null:

```php
#[Requires('description', '?string')]    // string o null
#[Requires('updated', '?DateTime')]      // DateTime o null
#[Requires('metadata', '?array')]        // array o null
```

---

## 7. Excepciones y Errores

### 7.1 Excepción Base

Ubicación: [`src/Exception/WorkflowException/WorkflowStepDependencyNotFulfilledException.php`](../src/Exception/WorkflowException/WorkflowStepDependencyNotFulfilledException.php)

```php
class WorkflowStepDependencyNotFulfilledException extends WorkflowException
{
    // Lanzada cuando validación de dependencia falla
}
```

### 7.2 Mensajes de Error

**Clave ausente**:
```
Missing 'order_id' in container
```

**Tipo inválido**:
```
Value for 'amount' has an invalid type. Expected float, got int
```

**Tipo inválido con clase**:
```
Value for 'user' has an invalid type. Expected User, got object (stdClass)
```

**Tipo opcional violado**:
```
Value for 'description' has an invalid type. Expected ?string, got array
```

### 7.3 Captura en Workflow

```php
$result = (new Workflow('payment'))
    ->process(new ChargePaymentStep())
    ->executeWorkflow($container, false);

if (!$result->success()) {
    $exception = $result->getException();
    // Exception será WorkflowStepDependencyNotFulfilledException
    echo $exception->getMessage();
}
```

---

## 8. Patrones de Uso

### 8.1 Patrón 1: Validación Simple

```php
$container = (new WorkflowContainer())
    ->set('order_id', 12345);

$workflow = new Workflow('order-process');
$workflow->process(new ProcessOrderStep());

$result = $workflow->executeWorkflow($container);

// ProcessOrderStep ejecuta porque 'order_id' existe ✅
```

### 8.2 Patrón 2: Validación Tipada

```php
$container = (new WorkflowContainer())
    ->set('amount', 99.99)      // float ✅
    ->set('currency', 'USD')    // string ✅
    ->set('card_token', 'tok_'); // string ✅

$workflow = new Workflow('payment');
$workflow->process(new ChargePaymentStep());

$result = $workflow->executeWorkflow($container);

// ChargePaymentStep ejecuta, todos los tipos coinciden ✅
```

### 8.3 Patrón 3: Error - Tipo Incorrecto

```php
$container = (new WorkflowContainer())
    ->set('amount', '99.99')    // string ✗ (esperaba float)
    ->set('currency', 'USD')    // string ✅
    ->set('card_token', 'tok_'); // string ✅

$result = $workflow->executeWorkflow($container, false);

// ChargePaymentStep NO ejecuta
// Falla: "Value for 'amount' has an invalid type. Expected float, got string"
```

### 8.4 Patrón 4: Error - Clave Ausente

```php
$container = (new WorkflowContainer())
    // amount no está en container ✗
    ->set('currency', 'USD')
    ->set('card_token', 'tok_');

$result = $workflow->executeWorkflow($container, false);

// ChargePaymentStep NO ejecuta
// Falla: "Missing 'amount' in container"
```

### 8.5 Patrón 5: Tipos Opcionales

```php
class ProcessUserStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('user_id', 'int')]
        #[Requires('email', '?string')]  // Puede ser string o null
        WorkflowContainer $container
    ): void {
        $userId = $container->get('user_id');   // int garantizado
        $email = $container->get('email');      // string o null garantizado
    }

    public function getDescription(): string { return 'Process user'; }
}

// Válidos:
$container1 = (new WorkflowContainer())
    ->set('user_id', 123)
    ->set('email', 'test@example.com');  // string ✅

$container2 = (new WorkflowContainer())
    ->set('user_id', 123)
    ->set('email', null);  // null ✅

// Inválido:
$container3 = (new WorkflowContainer())
    ->set('user_id', 123)
    ->set('email', 123);  // int ✗ (no es string ni null)
```

---

## 9. Múltiples Dependencias

### 9.1 Múltiples en Mismo Parámetro

```php
class ApprovalStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('approval_count', 'int')]
        #[Requires('rejection_count', 'int')]
        #[Requires('status', 'string')]
        WorkflowContainer $container
    ): void {
        $approvals = $container->get('approval_count');
        $rejections = $container->get('rejection_count');
        $status = $container->get('status');
        
        // Lógica de aprobación...
    }

    public function getDescription(): string { return 'Approval step'; }
}

// Validación: Se verifican TODAS las #[Requires]
// Si alguna falla → step no ejecuta
```

### 9.2 Dependencias en Cascada

```php
class DataProcessingStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('raw_data', 'array')]
        #[Requires('parser', 'DataParser')]
        #[Requires('schema', '?array')]
        #[Requires('strict_mode', 'bool')]
        WorkflowContainer $container
    ): void {
        // Todos los datos validados, lista para procesar
        $data = $container->get('raw_data');      // array
        $parser = $container->get('parser');      // DataParser instance
        $schema = $container->get('schema');      // array o null
        $strict = $container->get('strict_mode'); // bool
    }

    public function getDescription(): string { return 'Process data'; }
}
```

---

## 10. Comparación: Dependencias vs Validación Manual

### 10.1 SIN Sistema de Dependencias

```php
class OldWayStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        WorkflowContainer $container
    ): void {
        // Validación manual
        if (!$container->has('order_id')) {
            $control->failStep("Missing 'order_id'");
            return;
        }
        
        $orderId = $container->get('order_id');
        
        if (!is_int($orderId)) {
            $control->failStep("order_id must be int");
            return;
        }
        
        // Finalmente, lógica real
        // ...
    }

    public function getDescription(): string { return 'Process order'; }
}
```

**Problemas**:
- Código repetitivo en cada step
- Validación manual propensa a errores
- Complejo de mantener
- Mezcla validación con lógica

### 10.2 CON Sistema de Dependencias (PHP 8+)

```php
class NewWayStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('order_id', 'int')]  // Declarativo
        WorkflowContainer $container
    ): void {
        // Garantizado: order_id existe y es int
        $orderId = $container->get('order_id');
        
        // Directamente lógica real
        // ...
    }

    public function getDescription(): string { return 'Process order'; }
}
```

**Ventajas**:
- Declarativo y clara intención
- Validación automática
- No requiere código defensivo
- Separación clara entre validación e implementación
- Message consistente en todas las faltas

---

## 11. Cómo Funciona: Bajo el Capó

### 11.1 Ejecución del Middleware

En `resolveMiddleware()` (StepExecutionTrait.php):

```php
if (PHP_MAJOR_VERSION >= 8) {
    array_unshift($middlewares, new WorkflowStepDependencyCheck());
}
```

**Resultado**: `WorkflowStepDependencyCheck` se prepende como PRIMER middleware.

### 11.2 Validación en WorkflowStepDependencyCheck

```php
class WorkflowStepDependencyCheck
{
    public function __invoke(
        callable $next,
        WorkflowControl $control,
        WorkflowContainer $container,
        WorkflowStep $step
    ) {
        // 1. Obtiene el método 'run'
        $runMethod = new ReflectionMethod($step, 'run');

        // 2. Busca atributos #[Requires] en el parámetro $container
        foreach ($runMethod->getParameters() as $parameter) {
            foreach ($parameter->getAttributes(
                StepDependencyInterface::class,
                ReflectionAttribute::IS_INSTANCEOF
            ) as $dependencyAttribute) {
                // 3. Instancia el atributo
                $dependency = $dependencyAttribute->newInstance();
                
                // 4. Ejecuta check()
                $dependency->check($container);  // Lanza si falla
            }
        }

        // 5. Si todas pasan, ejecuta el step
        return $next();
    }
}
```

**Orden**:
1. Reflection del método `run()`
2. Busca parámetros con atributos `StepDependencyInterface`
3. Ejecuta `check()` en cada uno
4. Si alguno falla → excepción
5. Si todos pasan → ejecuta step

### 11.3 Validación en Requires

```php
public function check(WorkflowContainer $container): void
{
    // 1. Verifica que la clave existe
    if (!$container->has($this->key)) {
        throw new WorkflowStepDependencyNotFulfilledException(
            "Missing '{$this->key}' in container"
        );
    }

    // 2. Si sin tipo especificado → validación completa
    if ($this->type === null) {
        return;  // Solo verificamos existencia
    }

    $value = $container->get($this->key);

    // 3. Si permite null (tipo opcional)
    if (str_starts_with($this->type, '?') && $value === null) {
        return;  // null es válido para tipo opcional
    }

    // 4. Valida tipo
    $type = str_replace('?', '', $this->type);
    
    if (preg_match('/^(string|bool|int|float|object|array|iterable|scalar)$/', $type)) {
        // Primitivo
        $checkMethod = 'is_' . $type;
        if ($checkMethod($value)) {
            return;  // Tipo válido
        }
    } elseif (class_exists($type) && ($value instanceof $type)) {
        // Clase
        return;  // Instancia válida
    }

    // 5. Tipo no válido
    throw new WorkflowStepDependencyNotFulfilledException(
        "Value for '{$this->key}' has an invalid type. Expected $this->type, got " .
        gettype($value) . (is_object($value) ? " ({get_class($value)})" : '')
    );
}
```

---

## 12. Implementar Dependencia Personalizada

### 12.1 Crear Interfaz

```php
// src/MyApp/OrderExists.php

namespace MyApp\Step\Dependency;

use PHPWorkflow\Step\Dependency\StepDependencyInterface;
use PHPWorkflow\State\WorkflowContainer;
use PHPWorkflow\Exception\WorkflowException\WorkflowStepDependencyNotFulfilledException;

#[Attribute(Attribute::TARGET_PARAMETER | Attribute::IS_REPEATABLE)]
class OrderExists implements StepDependencyInterface
{
    public function check(WorkflowContainer $container): void
    {
        $orderId = $container->get('order_id');
        
        // Valida que el order existe en BD (ejemplo)
        if (!$this->orderExists($orderId)) {
            throw new WorkflowStepDependencyNotFulfilledException(
                "Order #{$orderId} does not exist in database"
            );
        }
    }
    
    private function orderExists(int $orderId): bool
    {
        // Valida contra BD, cache, etc.
        return true;  // Simplified
    }
}
```

### 12.2 Usar en Step

```php
class ProcessOrderStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[OrderExists()]  // Validación personalizada
        #[Requires('order_id', 'int')]  // Validación builtin
        WorkflowContainer $container
    ): void {
        // Garantizado: order_id existe, es int, y existe en BD
        $orderId = $container->get('order_id');
        // ...
    }

    public function getDescription(): string { return 'Process order'; }
}
```

---

## 13. Precondiciones vs Dependencias Formales

### 13.1 Diferencia

**Dependencias Formales** (Sistema Requires):
- Validación de tipo y existencia
- Parameter-level checking
- Automática en PHP 8+
- Falla el step si no se cumple

**Precondiciones** (Validación en Step):
```php
class ProcessOrderStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('order_id', 'int')]
        WorkflowContainer $container
    ): void {
        $orderId = $container->get('order_id');
        
        // Precondición: order no está cancelada
        if ($container->get('order_status') === 'CANCELLED') {
            $control->skipStep('Order is cancelled');
            return;
        }
        
        // Precondición: usuario tiene permisos
        if (!$this->hasPerms($container->get('user'))) {
            $control->failStep('Insufficient permissions');
            return;
        }
        
        // Lógica principal...
    }
}
```

**Cuándo usar**:
- **Requires**: Clave existe Y tipo correcto
- **Precondiciones**: Lógica condicional dentro del step

---

## 14. Casos de Uso Prácticos

### 14.1 E-commerce: Validación de Checkout

```php
class ValidateCheckoutStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('cart_items', 'array')]
        #[Requires('customer_id', 'int')]
        #[Requires('shipping_address', 'array')]
        #[Requires('payment_method', 'string')]
        #[Requires('coupon_code', '?string')]
        WorkflowContainer $container
    ): void {
        // Todas las dependencias validadas
        $items = $container->get('cart_items');
        $customerId = $container->get('customer_id');
        // ... resto de lógica
    }

    public function getDescription(): string { return 'Validate checkout'; }
}
```

### 14.2 API Processing: Type-Safe Parameters

```php
class CreateUserStep implements WorkflowStep
{
    public function run(
        WorkflowControl $control,
        #[Requires('email', 'string')]
        #[Requires('name', 'string')]
        #[Requires('age', 'int')]
        #[Requires('terms_accepted', 'bool')]
        #[Requires('phone', '?string')]
        #[Requires('validator', 'EmailValidator')]
        WorkflowContainer $container
    ): void {
        // Garantizado tipos correctos
        $email = $container->get('email');        // string
        $name = $container->get('name');          // string
        $age = $container->get('age');            // int
        $termsAccepted = $container->get('terms_accepted'); // bool
        $phone = $container->get('phone');        // string o null
        $validator = $container->get('validator'); // EmailValidator instance
        
        // Procesa validación y creación...
    }

    public function getDescription(): string { return 'Create user'; }
}
```

---

## 15. Debugging de Dependencias

### 15.1 Error en Log

```
Process log for workflow 'payment':
Process:
  - Charge payment: failed (Value for 'amount' has an invalid type. Expected float, got string)
```

### 15.2 Inspeccionar Contenedor

```php
$result = (new Workflow('test'))
    ->process(new ChargePaymentStep())
    ->executeWorkflow($container, false);

if (!$result->success()) {
    echo "Error: " . $result->getException()->getMessage();
    
    // Inspecciona qué estaba en el contenedor
    $finalContainer = $result->getContainer();
    echo "amount = " . var_export($finalContainer->get('amount'), true);
    echo "currency = " . var_export($finalContainer->get('currency'), true);
}
```

---

## 16. Limitaciones

### 16.1 Solo PHP 8+

Sistema no funciona en PHP 7.x por falta de Attributes.

```php
// PHP 7: Manual validation required
if (PHP_MAJOR_VERSION < 8) {
    // No #[Requires] available
    // Implementa validación manual
}
```

### 16.2 Solo en Parámetro $container

Las dependencias se declaran en el parámetro del contenedor:

```php
// ✅ CORRECTO - Dependencia sobre $container
public function run(
    WorkflowControl $control,
    #[Requires('key')]
    WorkflowContainer $container
): void { }

// ❌ INCORRECTO - No funciona en $control
public function run(
    #[Requires('key')]
    WorkflowControl $control,
    WorkflowContainer $container
): void { }
```

### 16.3 Solo Validación, No Inyección

Requires valida pero NO inyecta. Debes obtener del contenedor:

```php
#[Requires('user_id', 'int')]
WorkflowContainer $container

// Debes hacer:
$userId = $container->get('user_id');

// NO inyecta automáticamente
// Es validación + contenedor normal
```

---

## 17. Hallazgos Clave

### 17.1 Automático en PHP 8+

No requiere configuración. Se valida antes de cada step automáticamente.

### 17.2 LIFO en Middleware Chain

`WorkflowStepDependencyCheck` se prepende como PRIMER middleware → ejecuta ANTES de cualquier otro middleware.

Implicación: Custom middlewares que modifican container ocurren DESPUÉS de validación de dependencias.

### 17.3 Validación Type-Safe

Pequeño overhead pero garantiza tipos correctos en todas partes del step.

### 17.4 Integral con Exceptions

Las excepciones se capturan y reportan claramente en logs.

---

## 18. Conclusiones

El sistema de dependencias de php-workflow es **declarativo y type-safe**:

### 18.1 Fortalezas

- **Declarativo**: Claro qué requiere el step
- **Automático**: Valida sin código extra
- **Type-safe**: Garantiza tipos en runtime
- **Composable**: Múltiples dependencias
- **Extensible**: Implementar dependencias personalizadas

### 18.2 Responsabilidades del Usuario

- Usar PHP 8.0+
- Declarar dependencias explícitamente
- Implementar dependencias personalizadas según necesidad
- Interpretar mensajes de error

### 18.3 Best Practices

- **Declarar TODOS los requisitos** para cada step
- **Usar tipos específicos** para documentación
- **Usar opcionales `?`** cuando sea apropiado
- **Implementar validaciones personalizadas** para lógica compleja
- **NO mezclar** dependencias formales con validación manual innecesaria

### 18.4 Comparación con Alternativas

| Enfoque | Ventajas | Desventajas |
|---------|----------|-------------|
| Validación manual | Flexible, trabajaen PHP 7 | Repetitivo, propensoa errores |
| Requires system | Declarativo, automático | Requiere PHP 8+, menos flexible |
| Hybrid | Lo mejor de ambos | Más complejo |

**Conclusión**: Usa Requires para validación estándar, código manual para lógica compleja.

---

## 19. Referencias de Código

### Ubicaciones Clave

- Interfaz: [src/Step/Dependency/StepDependencyInterface.php](../src/Step/Dependency/StepDependencyInterface.php)
- Implementación: [src/Step/Dependency/Requires.php](../src/Step/Dependency/Requires.php)
- Middleware: [src/Middleware/WorkflowStepDependencyCheck.php](../src/Middleware/WorkflowStepDependencyCheck.php)
- Excepción: [src/Exception/WorkflowException/WorkflowStepDependencyNotFulfilledException.php](../src/Exception/WorkflowException/WorkflowStepDependencyNotFulfilledException.php)

### Tests Recomendados

- [tests/StepDependencyTest.php](../tests/StepDependencyTest.php) - Casos completos
  - testMissingKeyFails() - Clave ausente
  - testInvalidTypedValueFails() - Tipo incorrecto
  - testProvidedTypedKeySucceeds() - Caso éxito con tipos
  - testInvalidNullableTypedValueFails() - Tipos opcionales
  - testProvidedDateTimeSucceeds() - Clases como DateTime

---

## 20. Próximas Extensiones

Para mayor funcionalidad:

- **Grupo 7: WorkflowResult** - Acceso a datos tras ejecución
- **Grupo 8: Best Practices** - Patrones avanzados de validación
- **Grupo 9: Testing** - Tests con dependencias

