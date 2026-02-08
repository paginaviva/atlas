---
Título: Conceptos Fundamentales de php-workflow
Descripción: Explicación de Steps Básicos, Loops, Middleware y Nested Workflows
Fecha: 2026-02-08
---

# Conceptos Fundamentales de php-workflow

Este documento explica los cuatro conceptos clave de **php-workflow**: steps básicos, loops, middleware y workflows anidados. Cada uno resuelve un aspecto diferente de la ejecución de workflows complejos.

---

## 1. Steps Básicos (Pasos Simples)

### ¿Qué Significa?

Un **Step** (paso) es la **unidad de trabajo más pequeña e indivisible** en un workflow. Es un bloque de código que implementa una acción específica y bien definida dentro del proceso.

Es comparable a una **función o tarea atómica** que:
- Realiza exactamente **una responsabilidad** (principio de responsabilidad única)
- Se ejecuta de forma **desacoplada** del resto del workflow
- Comunica su resultado mediante **estados predefinidos** (éxito, fallo, omitir, etc.)

### Funciones Principales

Un step básico tiene tres responsabilidades clave:

1. **Lectura de Datos**: Obtiene información del contenedor compartido (`WorkflowContainer`)
   ```php
   $usuario = $container->get('usuario');
   $email = $container->get('email');
   ```

2. **Procesamiento**: Ejecuta su lógica de negocio
   ```php
   // Validar, procesar, transformar, consultar BD, etc.
   if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
       // Comunicar que falló...
   }
   ```

3. **Escritura de Resultados**: Actualiza el contenedor con nuevos datos
   ```php
   $container->set('usuario_id', 123);
   $container->set('resultado', 'éxito');
   ```

### Finalidad

La finalidad de los steps es **descomponer un proceso complejo en tareas simples y reutilizables**:

- ✅ **Modularidad**: Cada step puede modificarse sin afectar otros
- ✅ **Reutilización**: El mismo step se puede usar en múltiples workflows
- ✅ **Testabilidad**: Cada step es fácil de probar de forma aislada
- ✅ **Legibilidad**: El workflow leyéndolo comunica claramente qué pasa
- ✅ **Mantenibilidad**: Cambios localizados en un solo lugar

### Estructura de un Step

Todo step implementa la interfaz `WorkflowStep`:

```php
interface WorkflowStep
{
    // Retorna descripción legible para logs
    public function getDescription(): string;

    // Método que ejecuta la lógica del step
    public function run(WorkflowControl $control, WorkflowContainer $container): void;
}
```

### Ejemplo Práctico

```php
// Step para validar un email
class ValidarEmailStep implements WorkflowStep
{
    public function getDescription(): string
    {
        return 'Validar formato de email del usuario';
    }

    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        $email = $container->get('email');

        // Validación
        if (empty($email)) {
            $control->failStep('Email no proporcionado');
            return;
        }

        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            $control->failStep('Email inválido: ' . $email);
            return;
        }

        // Si pasa validación, adjuntar información útil
        $control->attachStepInfo('email_validado', ['email' => $email]);
    }
}

// Usar en workflow
$workflow = new Workflow('Registro de Usuario')
    ->prepare(new ObtenerDatosStep())
    ->validate(new ValidarEmailStep())
    ->validate(new ValidarNombreStep())
    ->process(new GuardarUsuarioEnBDStep())
    ->executeWorkflow();
```

### Ciclo de Vida de un Step

```
1. Inyección: Se registra en una etapa específica (prepare, validate, process)
2. Espera: El workflow espera su turno
3. Ejecución: El motor ejecuta step->run()
4. Registro: El resultado se captura en los logs
5. Continuación: Se pasa al siguiente step
```

---

## 2. Loops (Bucles)

### ¿Qué Significa?

Un **Loop** es un step especial que **repite la ejecución de otros steps múltiples veces** dentro de un workflow. Es un **contenedor de steps** que se ejecuta iterativamente.

En lugar de escribir:
```php
// Malo: Código repetido
procesarItem($items[0]);
procesarItem($items[1]);
procesarItem($items[2]);
```

Usas un Loop:
```php
// Mejor: Código limpio y reutilizable
$loop = new Loop(new CollectionLoopControl('items'));
$loop->addStep(new ProcesarItemStep());
```

### Funciones Principales

1. **Iteración Controlada**: Repite steps según una lógica definida
   - Por contador (ej: 5 veces)
   - Sobre colecciones (ej: para cada item en lista)
   - Hasta una condición (ej: mientras haya trabajo)

2. **Control Fino**: Permite manipular el flujo dentro de cada iteración
   - `continue()`: Salta a la siguiente iteración
   - `break()`: Sale del loop
   - `skipStep()`: Omite un step en esta iteración
   - `failStep()`: Marca la iteración como fallida

3. **Aislamiento de Iteraciones**: Cada iteración es independiente
   - Los errores pueden capturarse por iteración
   - Puedes continuar incluso si una iteración falla (configurable)
   - La siguiente iteración comienza con datos actualizados

### Finalidad

Los loops resuelven el problema de **procesar colecciones sin duplicar código**:

- ✅ **Procesar listas**: Items de un array, registros de BD, líneas de archivo
- ✅ **Reintentos**: Repetir una acción hasta n intentos
- ✅ **Generación de trabajo**: Crear tareas derivadas dinámicamente
- ✅ **Tolerancia a fallos**: Continuar aunque alguna iteración falle
- ✅ **Rastreo por iteración**: Debuggear qué pasó en la iteración 5

### Componentes

Los loops tienen dos partes:

#### A. LoopControl (Decisor)

Interfaz que decide **si hay otra iteración**:

```php
interface LoopControl
{
    public function getDescription(): string;
    
    public function executeNextIteration(
        int $iteration,                    // Número de iteración (0, 1, 2, ...)
        WorkflowControl $control,          // Control de flujo
        WorkflowContainer $container       // Datos compartidos
    ): bool;  // true = continuar, false = terminar loop
}
```

#### B. Loop (Ejecutor)

El step que contiene la lógica de iteración:

```php
$loop = new Loop($loopControl, $continueOnError = false);
$loop->addStep(new ProcessarStep());
$loop->addStep(new ValidarStep());
```

### Ejemplo Práctico

```php
// LoopControl: Itera sobre items en el contenedor
class CollectionLoop implements LoopControl
{
    private string $listaKey;
    private string $itemKey;

    public function __construct(string $listaKey, string $itemKey = 'item')
    {
        $this->listaKey = $listaKey;
        $this->itemKey = $itemKey;
    }

    public function getDescription(): string
    {
        return "Procesar items de: {$this->listaKey}";
    }

    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        $items = $container->get($this->listaKey) ?? [];

        if (empty($items)) {
            return false;  // No hay más items → terminar loop
        }

        // Extrae el primer item
        $item = array_shift($items);

        // Actualiza el contenedor para que el step actual lo procese
        $container->set($this->itemKey, $item);
        $container->set($this->listaKey, $items);  // Actualiza cantidad restante

        return true;  // Continuar con esta iteración
    }
}

// Uso en workflow
$container = (new WorkflowContainer())
    ->set('ordenes', [
        ['id' => 1, 'monto' => 100],
        ['id' => 2, 'monto' => 200],
        ['id' => 3, 'monto' => 150],
    ]);

$loop = new Loop(new CollectionLoop('ordenes', 'orden_actual'));
$loop->addStep(new ValidarOrdenStep());
$loop->addStep(new ProcesarPagoStep());
$loop->addStep(new EnviarConfirmacionStep());

$workflow = new Workflow('Procesar Lote de Órdenes')
    ->process($loop)
    ->executeWorkflow($container);

// Resultado: Cada orden se procesa iterativamente
// Iteración 0: orden {'id': 1, 'monto': 100}
// Iteración 1: orden {'id': 2, 'monto': 200}
// Iteración 2: orden {'id': 3, 'monto': 150}
// Iteración 3: Loop termina (no hay más items)
```

### Ciclo de Ejecución de un Loop

```
Loop.run()
  ├─ Iteración 0:
  │   ├─ LoopControl.executeNextIteration(0)? → true
  │   ├─ ValidarOrdenStep
  │   ├─ ProcesarPagoStep
  │   └─ EnviarConfirmacionStep
  ├─ Iteración 1:
  │   ├─ LoopControl.executeNextIteration(1)? → true
  │   ├─ ValidarOrdenStep
  │   ├─ ProcesarPagoStep
  │   └─ EnviarConfirmacionStep
  ├─ Iteración 2:
  │   ├─ LoopControl.executeNextIteration(2)? → true
  │   ├─ ValidarOrdenStep
  │   ├─ ProcesarPagoStep
  │   └─ EnviarConfirmacionStep
  ├─ Iteración 3:
  │   └─ LoopControl.executeNextIteration(3)? → false (TERMINAR)
  └─ Loop finalizado
```

---

## 3. Middleware (Interceptores)

### ¿Qué Significa?

**Middleware** son **funciones interceptoras** que se ejecutan **alrededor de cada step**, permitiendo ejecutar lógica común (cross-cutting concerns) sin modificar los steps.

Es el patrón de **cadena de responsabilidad**: cada middleware puede procesarpre-ejecución y post-ejecución de un step.

### Funciones Principales

1. **Profiling/Timing**: Medir cuánto tarda cada step
   ```php
   // Middleware que mide tiempo
   $start = microtime(true);
   // Ejecuta step
   $tiempo = microtime(true) - $start;
   // Adjunta: "Step tomó 12.34ms"
   ```

2. **Validación de Dependencias**: Verificar precondiciones antes de ejecutar
   ```php
   // Middleware que valida que el contenedor tiene datos requeridos
   if (!$container->has('usuario_id')) {
       throw new Exception('Falta usuario_id');
   }
   // Ejecuta step
   ```

3. **Logging/Auditoría**: Registrar quién ejecutó qué y cuándo
   ```php
   // Middleware que registra ejecución
   $log->info("Iniciando step: {$step->getDescription()}");
   // Ejecuta step
   $log->info("Step completado");
   ```

4. **Transacciones de BD**: Envolver steps en transacciones
   ```php
   // Middleware que crea transacción
   $transaccion->begin();
   try {
       // Ejecuta step
       $transaccion->commit();
   } catch (Exception $e) {
       $transaccion->rollback();
   }
   ```

5. **Rate Limiting**: Controlar frecuencia de ejecución
   ```php
   // Middleware que limita llamadas externas
   if ($rateLimiter->exceeded()) {
       sleep(1);  // Esperar
   }
   // Ejecuta step
   ```

### Finalidad

Los middlewares resuelven el problema de **aplicar lógica repetitiva a todos los steps sin duplicar código**:

- ✅ **Separación de Responsabilidades**: Lógica transversal separada de steps
- ✅ **Reutilización**: El mismo middleware en múltiples workflows
- ✅ **Mantenibilidad**: Cambiar profiling, logging, etc. en un solo lugar
- ✅ **Flexibilidad**: Activar/desactivar middlewares según necesidad
- ✅ **Sin Intrusión**: Los steps no requieren cambios

### Estructura de un Middleware

Un middleware es un **callable** que recibe:

```php
$middleware = function(
    callable $next,                 // Siguiente middleware o step
    WorkflowControl $control,       // Control de flujo
    WorkflowContainer $container,   // Datos compartidos
    WorkflowStep $step              // El step siendo ejecutado
) {
    // Lógica PRE-ejecución
    $resultado = $next();           // Ejecuta siguiente
    // Lógica POST-ejecución
    return $resultado;
};
```

### Construcción: Cadena LIFO

Los middlewares se **apilan en orden inverso** (Last In, First Out):

```php
// Definición
$workflow = new Workflow('Mi Proceso',
    new ProfileStep(),              // 3º en ejecutarse PRE, 1º en ejecutarse POST
    new LoggingMiddleware(),         // 2º en ejecutarse PRE, 2º en ejecutarse POST
    new TransactionMiddleware()      // 1º en ejecutarse PRE, 3º en ejecutarse POST
);

// Ejecución real:
// TransactionMiddleware PRE
//   → LoggingMiddleware PRE
//     → ProfileStep PRE
//       → Step.run()
//       ← ProfileStep POST
//     ← LoggingMiddleware POST
//   ← TransactionMiddleware POST
```

### Ejemplo Práctico

```php
// Middleware: Medir tiempo de ejecución
class PerfilarStep
{
    public function __invoke(callable $next, WorkflowControl $control)
    {
        $inicio = microtime(true);

        try {
            $resultado = $next();  // Ejecuta el step
        } finally {
            // Adjunta timing SIEMPRE (incluso si hay excepción)
            $ms = (microtime(true) - $inicio) * 1000;
            $control->attachStepInfo('timing', [
                'milliseconds' => $ms
            ]);
        }

        return $resultado;
    }
}

// Middleware: Registrar ejecución en logs
class RegistrarStep
{
    public function __construct(private Logger $logger) {}

    public function __invoke(callable $next, WorkflowControl $control, WorkflowContainer $container, WorkflowStep $step)
    {
        $this->logger->info("Iniciando: {$step->getDescription()}");

        try {
            $resultado = $next();
            $this->logger->info("Completado: {$step->getDescription()}");
            return $resultado;
        } catch (Exception $e) {
            $this->logger->error("Falló: {$step->getDescription()}: {$e->getMessage()}");
            throw $e;
        }
    }
}

// Uso: Todos los steps tendrán profiling y logging automáticos
$workflow = new Workflow('Procesar Usuario',
    new PerfilarStep(),           // Mide tiempo
    new RegistrarStep($logger)    // Registra en logs
);

$workflow
    ->validate(new ValidarEmailStep())
    ->validate(new ValidarNombreStep())
    ->process(new GuardarEnBDStep())
    ->executeWorkflow();

// Resultado en logs:
// "Iniciando: Validar formato de email del usuario"
// "Completado: Validar formato de email del usuario" - timing: 2.34ms
// "Iniciando: Validar nombre de usuario"
// "Completado: Validar nombre de usuario" - timing: 1.56ms
// "Iniciando: Guardar usuario en BD"
// "Completado: Guardar usuario en BD" - timing: 45.78ms
```

### Middlewares Incluidos en php-workflow

1. **ProfileStep**: Mide tiempo de ejecución
2. **WorkflowStepDependencyCheck**: (PHP 8+) Valida dependencias antes de ejecutar

---

## 4. Nested Workflows (Workflows Anidados)

### ¿Qué Significa?

Un **Nested Workflow** es un **workflow completo que se ejecuta como si fuera un step** dentro de otro workflow. Es **composición jerárquica** de workflows.

En lugar de tener todo en un workflow:
```php
$workflow = new Workflow('Proceso Complejo')
    ->prepare(...100 steps aquí...)
    ->validate(...50 steps aquí...)
    ->process(...80 steps aquí...)
    ->executeWorkflow();  // Demasiado grande, inmanejable
```

Lo divides en sub-workflows:
```php
$subproceso1 = new Workflow('Preparación');  // 100 steps + etapas
$subproceso2 = new Workflow('Validación');   // 50 steps + etapas
$subproceso3 = new Workflow('Procesamiento'); // 80 steps + etapas

$workflow = new Workflow('Proceso Complejo')
    ->process(new NestedWorkflow($subproceso1))
    ->process(new NestedWorkflow($subproceso2))
    ->process(new NestedWorkflow($subproceso3))
    ->executeWorkflow();  // Limpio, modular
```

### Funciones Principales

1. **Composición**: Combinar workflows existentes sin reescribir
   ```php
   $registro = new Workflow('Registrar Usuario');
   $validacion = new Workflow('Validar Datos');
   
   $perfil = new Workflow('Crear Perfil')
       ->process(new NestedWorkflow($validacion))  // Reutiliza
       ->process(new NestedWorkflow($registro))    // Reutiliza
       ->executeWorkflow();
   ```

2. **Encapsulación de Complejidad**: Workflows complejos como unidades simples
   ```php
   new NestedWorkflow($workflowSuperComplejo)
   // ↓ Se ve como un step simple normalmente
   ```

3. **Herencia de Contexto**: Datos compartidos entre padre e hijo
   ```php
   // Workflow padre
   $container = new WorkflowContainer()
       ->set('usuario', ['id' => 1, 'email' => 'user@example.com']);

   // Workflow hijo accede a los mismos datos
   $workflow->executeWorkflow($container);
   ```

4. **Propagación de Estado**: Resultados, warnings y logs se propagan hacia arriba
   ```php
   $result = $workflow->executeWorkflow();
   // Incluye información de workflows anidados dentro
   $result->debug();  // Muestra árbol completo
   ```

5. **Manejo Automático de Errores**: Fallos del hijo se propagan inteligentemente
   ```php
   new NestedWorkflow($workflow)
   // Si el workflow anidado falla:
   // → Automáticamente marca este step como fallido
   // → El workflow padre termina (a menos que haya manejo de errores)
   ```

### Finalidad

Los workflows anidados resuelven el problema de **manejar procesos extremadamente complejos de forma organizada**:

- ✅ **Escalabilidad**: Procesos grandes manejables dividiéndolos
- ✅ **Reutilización**: Workflows son componibles
- ✅ **Mantenibilidad**: Cada workflow tiene responsabilidad clara
- ✅ **Reusabilidad**: El mismo workflow en múltiples contextos
- ✅ **Testabilidad**: Cada workflow probable independientemente
- ✅ **Inteligibilidad**: La estructura del proceso es clara

### Componentes

#### A. NestedWorkflow (Step especial)

Implementa `WorkflowStep` para ejecutar otro workflow:

```php
class NestedWorkflow implements WorkflowStep
{
    private ExecutableWorkflow $workflowAnidado;
    private ?WorkflowContainer $contenedorEspecifico;

    public function __construct(
        ExecutableWorkflow $workflowAnidado,
        ?WorkflowContainer $contenedorEspecifico = null
    ) {
        $this->workflowAnidado = $workflowAnidado;
        $this->contenedorEspecifico = $contenedorEspecifico;
    }

    public function getDescription(): string
    {
        return $this->workflowAnidado->getName();
    }

    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        // Ejecuta el workflow anidado
        // Si falla, marca este step como fallido
        // Propaga warnings del hijo
    }
}
```

#### B. NestedContainer (Contenedor heredado)

Contenedor que hereda datos del padre:

```php
// Lookup (búsqueda) en cascada:
$container->get('key')
  ├─ Si existe en contenedor específico → devuelve de ahí
  └─ Si no → devuelve del contenedor padre

// Set (escritura) en doble modo:
$container->set('key', 'value')
  ├─ Escribe en contenedor específico (si existe)
  └─ TAMBIÉN escribe en contenedor padre (propagación)
```

**Beneficio**: Los datos creados en el hijo se propagan automáticamente al padre.

### Ejemplo Práctico

```php
// Definir sub-workflows reutilizables
$validar = new Workflow('Validación de Datos')
    ->validate(new ValidarEmailStep())
    ->validate(new ValidarNombreStep())
    ->validate(new ValidarTelefonoStep());

$guardar = new Workflow('Guardar en BD')
    ->process(new CrearRegistroEnUsersStep())
    ->process(new CrearRegistroEnProfilesStep())
    ->after(new EnviarConfirmacionStep());

$notificar = new Workflow('Notificaciones')
    ->process(new EnviarEmailStep())
    ->process(new EnviarSmsStep());

// Workflow principal: pequeño, limpio, legible
$registro = new Workflow('Registrar Usuario Completo')
    ->prepare(new ObtenerDatosDelFormuarioStep())
    ->process(new NestedWorkflow($validar))      // Reutiliza validación
    ->process(new NestedWorkflow($guardar))      // Reutiliza guardado
    ->process(new NestedWorkflow($notificar))    // Reutiliza notificaciones
    ->after(new LimpiarTemporalesStep());

// Contexto compartido
$container = new WorkflowContainer()
    ->set('formulario', $_POST)
    ->set('timestamp', time());

// Ejecutar
$resultado = $registro->executeWorkflow($container);

// Resultado: Incluye estructura jerárquica
$resultado->debug();
/**
 * Registrar Usuario Completo (estado: ok)
 *   └─ Obtener Datos del Formulario (estado: ok)
 *   └─ Validación de Datos (estado: ok) [NESTED]
 *       ├─ Validar Email (estado: ok)
 *       ├─ Validar Nombre (estado: ok)
 *       └─ Validar Teléfono (estado: ok)
 *   └─ Guardar en BD (estado: ok) [NESTED]
 *       ├─ Crear Registro en Users (estado: ok)
 *       ├─ Crear Registro en Profiles (estado: ok)
 *       └─ Enviar Confirmación (estado: ok)
 *   └─ Notificaciones (estado: ok) [NESTED]
 *       ├─ Enviar Email (estado: ok)
 *       └─ Enviar SMS (estado: ok)
 *   └─ Limpiar Temporales (estado: ok)
 */
```

### Herencia de Contexto: Ejemplo Detallado

```php
// Workflow padre
$padre = new Workflow('Padre');

// Workflow hijo
$hijo = new Workflow('Hijo');

// Contenedor del padre
$containerPadre = (new WorkflowContainer())
    ->set('compartido', 'del-padre')
    ->set('id_usuario', 42);

// Ejecutar
$resultado = $padre
    ->process(new NestedWorkflow($hijo))
    ->executeWorkflow($containerPadre);

// Dentro del paso NestedWorkflow:
// 1. Crea NestedContainer($containerPadre, null)
// 2. El hijo accede a datos así:
//    $container->get('compartido')   // 'del-padre' ← del padre
//    $container->get('id_usuario')   // 42 ← del padre
//    $container->set('resultado', 'éxito')
//    // Se escribe en AMBOS: hijo y padre ← PROPAGACIÓN
//
// 3. Cuando termina el hijo, el padre tiene:
//    $containerPadre->get('resultado')  // 'éxito' ← propagado del hijo!
```

### Cascada de Workflows (Múltiples Niveles)

```
Nivel 1: Workflow Principal
  │
  ├─ Paso 1
  ├─ NestedWorkflow (Nivel 2a)
  │   │
  │   ├─ Paso 2a
  │   └─ NestedWorkflow (Nivel 3)
  │       │
  │       ├─ Paso 3
  │       └─ NestedWorkflow (Nivel 4)
  │           │
  │           └─ Paso 4 [Profundo!]
  │
  ├─ NestedWorkflow (Nivel 2b)
  │   └─ Paso 2b
  │
  └─ Paso 5

Datos compartidos fluyen en cascada:
Nivel 1 → Nivel 2a → Nivel 3 → Nivel 4
Y todo lo modificado en Nivel 4 se propaga hacia arriba
```

---

## Comparación y Cuándo Usar Cada Uno

| Concepto | Cuándo Usar | Ventaja Principal |
|----------|-------------|------------------|
| **Steps Básicos** | Acciones simples, únicas | Sencillez, enfoque |
| **Loops** | Procesar colecciones, reintentos | Evitar duplicación, iteración limpia |
| **Middleware** | Lógica transversal (profiling, logging, etc.) | Aplicar a TODOS los steps sin tocarlos |
| **Nested Workflows** | Procesos complejos, reutilización | Modularidad, composición, escalabilidad |

---

## Resumen Visual

```
┌─────────────────────────────────────────────────┐
│          WORKFLOW (Orquestador)                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  PREPARE ─► Step Básico (A)                    │
│             Step Básico (B)                    │
│                                                 │
│  VALIDATE ─► Step Básico (C)                   │
│             Middleware {                       │
│               Logging                          │
│               Profiling   ─► Step (D)         │
│             }                                  │
│                                                 │
│  PROCESS  ─► Loop {                            │
│               ├─ Step (E)                      │
│               └─ Step (F)  ✕ Iteración 1       │
│               ├─ Step (E)                      │
│               └─ Step (F)  ✕ Iteración 2       │
│             }                                  │
│                                                 │
│           ─► NestedWorkflow {                  │
│               Workflow Interno                │
│               ├─ Validate                      │
│               ├─ Process                       │
│               └─ After                         │
│             }                                  │
│                                                 │
│  AFTER    ─► Step Básico (G)                   │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Conclusión

php-workflow proporciona **cuatro herramientas complementarias**:

1. **Steps**: Para responsabilidades simples
2. **Loops**: Para repetición controlada
3. **Middleware**: Para comportamiento transversal
4. **Nested Workflows**: Para composición y modularidad

Combinadas, permiten construir **procesos complejos, mantenibles y escalables** de forma elegante y sin duplicación de código.
