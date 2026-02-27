# 12. Integración y Casos de Uso Prácticos en php-workflow

## 1. Objetivo

Documentar patrones de integración de php-workflow con frameworks populares (Laravel, Symfony), mostrar casos de uso prácticos end-to-end, y proporcionar ejemplos completamente funcionales que demuestran cómo usar la biblioteca en aplicaciones reales.

---

## 2. Contexto

Después de entender php-workflow, necesitas saber:
- **Cómo integrarlo** con frameworks existentes
- **Casos de uso reales**: E-commerce, procesos administrativos, ETL, etc.
- **Ejemplos end-to-end**: Código completo, ejecutable, con manejo de errores

---

## 3. Integración: Standalone (Sin Framework)

### 3.1 Setup Básico

```php
// composer.json
{
    "require": {
        "wol-soft/php-workflow": "^1.0"
    }
}

// bootstrap.php
require_once 'vendor/autoload.php';

use PHPWorkflow\Workflow;
use PHPWorkflow\State\WorkflowContainer;
use PHPWorkflow\WorkflowControl;
use PHPWorkflow\Step\WorkflowStep;

// Aquí defini tus steps y workflows
```

### 3.2 DI Container Manual

```php
class WorkflowRegistry {
    private UserRepository $userRepo;
    private PaymentGateway $paymentGateway;
    private EmailService $emailService;
    
    public function __construct(
        UserRepository $userRepo,
        PaymentGateway $paymentGateway,
        EmailService $emailService
    ) {
        $this->userRepo = $userRepo;
        $this->paymentGateway = $paymentGateway;
        $this->emailService = $emailService;
    }
    
    public function checkoutWorkflow(): Workflow {
        return (new Workflow('checkout'))
            ->before(new LoadUserStep($this->userRepo))
            ->validate(new ValidateCartStep())
            ->validate(new ValidatePaymentStep())
            ->process(new CalculateTotalStep())
            ->process(new ProcessPaymentStep($this->paymentGateway))
            ->onSuccess(new SendConfirmationStep($this->emailService))
            ->onError(new RollbackPaymentStep($this->paymentGateway));
    }
    
    public function userRegistrationWorkflow(): Workflow {
        return (new Workflow('user-registration'))
            ->validate(new ValidateEmailStep())
            ->validate(new ValidatePasswordStep())
            ->process(new CreateUserStep($this->userRepo))
            ->onSuccess(new SendVerificationEmailStep($this->emailService));
    }
}

// Uso
$registry = new WorkflowRegistry($userRepo, $paymentGateway, $emailService);
$workflow = $registry->checkoutWorkflow();
$result = $workflow->executeWorkflow($container);
```

---

## 4. Integración: Laravel

### 4.1 Service Provider

```php
// app/Providers/WorkflowServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Workflows\OrderWorkflowFactory;
use App\Workflows\PaymentWorkflowFactory;

class WorkflowServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Registra factories en contenedor
        $this->app->singleton(OrderWorkflowFactory::class, function ($app) {
            return new OrderWorkflowFactory(
                $app->make('App\Repositories\OrderRepository'),
                $app->make('App\Services\PaymentService'),
                $app->make('App\Services\EmailService')
            );
        });
        
        $this->app->singleton(PaymentWorkflowFactory::class, function ($app) {
            return new PaymentWorkflowFactory(
                $app->make('App\Services\PaymentGateway')
            );
        });
    }
    
    public function boot(): void
    {
        // Registramos en config/app.php
    }
}

// config/app.php
'providers' => [
    // ...
    App\Providers\WorkflowServiceProvider::class,
]
```

### 4.2 Uso en Controller

```php
// app/Http/Controllers/CheckoutController.php
namespace App\Http\Controllers;

use App\Workflows\OrderWorkflowFactory;
use Illuminate\Http\Request;
use PHPWorkflow\State\WorkflowContainer;

class CheckoutController extends Controller
{
    public function __construct(private OrderWorkflowFactory $workflowFactory) {}
    
    public function execute(Request $request)
    {
        $container = new WorkflowContainer();
        $container->set('user', Auth::user());
        $container->set('cart', session('cart'));
        $container->set('shipping_address', $request->input('shipping'));
        
        $workflow = $this->workflowFactory->checkoutWorkflow();
        
        try {
            $result = $workflow->executeWorkflow($container);
            
            if ($result->success()) {
                return response()->json([
                    'status' => 'success',
                    'order_id' => $result->getContainer()->get('order_id'),
                ], 201);
            } else {
                return response()->json([
                    'status' => 'error',
                    'error' => $result->getException()->getMessage(),
                ], 400);
            }
        } catch (\Exception $e) {
            return response()->json([
                'status' => 'error',
                'error' => $e->getMessage(),
            ], 500);
        }
    }
}
```

### 4.3 Job para Ejecutar Workflow Async

```php
// app/Jobs/ProcessCheckout.php
namespace App\Jobs;

use App\Workflows\OrderWorkflowFactory;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use PHPWorkflow\State\WorkflowContainer;

class ProcessCheckout implements ShouldQueue
{
    use Queueable;
    
    public function __construct(
        private int $userId,
        private array $cartItems,
        private string $orderId
    ) {}
    
    public function handle(OrderWorkflowFactory $factory): void
    {
        $container = new WorkflowContainer();
        $container->set('user_id', $this->userId);
        $container->set('items', $this->cartItems);
        $container->set('order_id', $this->orderId);
        
        $workflow = $factory->checkoutWorkflow();
        $result = $workflow->executeWorkflow($container, false);
        
        if (!$result->success()) {
            Log::error("Checkout failed for order {$this->orderId}", [
                'exception' => $result->getException()->getMessage(),
                'logs' => $result->debug(),
            ]);
            
            // Notifica usuario y limpia
            $this->release(60);  // Reintentar en 60 segundos
        } else {
            Log::info("Checkout completed for order {$this->orderId}");
        }
    }
}

// En controller
dispatch(new ProcessCheckout($user->id, $cart, $orderId));
```

### 4.4 Middleware Integrado

```php
// app/Workflows/Middleware/LoggingMiddleware.php
namespace App\Workflows\Middleware;

use PHPWorkflow\Step\WorkflowStep;
use PHPWorkflow\WorkflowControl;
use PHPWorkflow\State\WorkflowContainer;
use Illuminate\Support\Facades\Log;

class LoggingMiddleware
{
    public function __invoke(WorkflowStep $step): WorkflowStep
    {
        return new class($step) implements WorkflowStep {
            public function __construct(private WorkflowStep $inner) {}
            
            public function run(WorkflowControl $control, WorkflowContainer $container): void
            {
                $start = microtime(true);
                
                try {
                    $this->inner->run($control, $container);
                    $duration = (microtime(true) - $start) * 1000;
                    
                    Log::debug("Step executed", [
                        'step' => $this->inner->getDescription(),
                        'duration_ms' => $duration,
                    ]);
                } catch (\Exception $e) {
                    Log::error("Step failed", [
                        'step' => $this->inner->getDescription(),
                        'error' => $e->getMessage(),
                    ]);
                    throw $e;
                }
            }
            
            public function getDescription(): string {
                return $this->inner->getDescription();
            }
        };
    }
}
```

---

## 5. Integración: Symfony

### 5.1 Service Configuration

```yaml
# config/services.yaml
services:
  App\Workflows\OrderWorkflowFactory:
    arguments:
      - '@App\Repository\OrderRepository'
      - '@App\Service\PaymentService'
      - '@App\Service\EmailService'
  
  App\Workflows\PaymentWorkflowFactory:
    arguments:
      - '@App\Service\PaymentGateway'
```

### 5.2 Uso en Symfony Controller

```php
// src/Controller/CheckoutController.php
namespace App\Controller;

use App\Workflows\OrderWorkflowFactory;
use PHPWorkflow\State\WorkflowContainer;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Annotation\Route;

class CheckoutController extends AbstractController
{
    public function __construct(private OrderWorkflowFactory $workflowFactory) {}
    
    #[Route('/api/checkout', methods: ['POST'])]
    public function checkout(Request $request): JsonResponse
    {
        $container = new WorkflowContainer();
        $container->set('user', $this->getUser());
        $container->set('cart', $this->getCart());
        $container->set('shipping', $request->json()->get('shipping'));
        
        $workflow = $this->workflowFactory->checkoutWorkflow();
        $result = $workflow->executeWorkflow($container, false);
        
        if ($result->success()) {
            return $this->json([
                'status' => 'success',
                'order_id' => $result->getContainer()->get('order_id'),
            ], 201);
        } else {
            return $this->json([
                'status' => 'error',
                'error' => $result->getException()->getMessage(),
            ], 400);
        }
    }
}
```

### 5.3 Event Listener para Workflow Completion

```php
// src/EventListener/WorkflowCompletionListener.php
namespace App\EventListener;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use App\Event\WorkflowCompletedEvent;

class WorkflowCompletionListener implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            WorkflowCompletedEvent::class => 'onWorkflowCompleted',
        ];
    }
    
    public function onWorkflowCompleted(WorkflowCompletedEvent $event): void
    {
        $result = $event->getWorkflowResult();
        
        // Dispatch eventos de dominio basados en resultado
        if ($result->success()) {
            // Dispatch domain events
        } else {
            // Notify about failure
        }
    }
}
```

---

## 6. Caso de Uso 1: E-Commerce Checkout

### 6.1 Workflow Definition

```php
class CheckoutWorkflow
{
    public function build(
        OrderRepository $orderRepo,
        PaymentGateway $gateway,
        EmailService $email,
        InventoryService $inventory
    ): Workflow {
        return (new Workflow('e-commerce-checkout'))
            // Validación temprana
            ->validate(new ValidateCartStep())
            ->validate(new ValidateUserStep())
            ->validate(new ValidateAddressStep())
            
            // Preparación
            ->before(new ReserveInventoryStep($inventory))
            ->before(new CalculateTaxStep())
            
            // Procesamiento principal
            ->process(new CalculateTotalStep())
            ->process(new ProcessPaymentStep($gateway))
            ->process(new CreateOrderStep($orderRepo))
            ->process(new UpdateInventoryStep($inventory))
            
            // Éxito
            ->onSuccess(new SendOrderConfirmationStep($email))
            ->onSuccess(new UpdateCustomerMetricsStep($orderRepo))
            
            // Error
            ->onError(new ReleaseReservedInventoryStep($inventory))
            ->onError(new RefundPaymentStep($gateway))
            ->onError(new NotifyCustomerOfFailureStep($email))
            
            // Limpieza
            ->after(new LogCheckoutStep());
    }
}
```

### 6.2 Ejecución Completa

```php
// En un controller o service
class CheckoutService
{
    public function executeCheckout(User $user, Cart $cart, Address $address): WorkflowResult
    {
        // Preparar contenedor
        $container = new WorkflowContainer();
        $container->set('user', $user);
        $container->set('cart', $cart);
        $container->set('address', $address);
        
        // Obtener workflow
        $workflow = $this->factory->build(
            $this->orderRepo,
            $this->gateway,
            $this->email,
            $this->inventory
        );
        
        // Ejecutar
        $result = $workflow->executeWorkflow($container, false);
        
        // Manejo de resultado
        if ($result->success()) {
            $order = $result->getContainer()->get('order');
            
            return new CheckoutSuccess($order->id, $order->total);
        } else {
            $error = $result->getException();
            
            // Log para debugging
            Log::error('Checkout failed', [
                'user_id' => $user->id,
                'error' => $error->getMessage(),
                'debug_log' => $result->debug(),
            ]);
            
            throw new CheckoutFailedException($error->getMessage());
        }
    }
}
```

---

## 7. Caso de Uso 2: User Onboarding

### 7.1 Workflow Multi-Step

```php
class UserOnboardingWorkflow
{
    public function build(): Workflow {
        return (new Workflow('user-onboarding'))
            ->prepare(new PrepareBillingDataStep())
            
            ->validate(new ValidateEmailStep())
            ->validate(new ValidatePasswordStrengthStep())
            ->validate(new CheckEmailAvailabilityStep())
            
            ->before(new GenerateVerificationTokenStep())
            
            ->process(new CreateUserAccountStep())
            ->process(new CreateDefaultWorkspaceStep())
            ->process(new CreateApiTokenStep())
            
            ->onSuccess(new SendVerificationEmailStep())
            ->onSuccess(new RecordOnboardingMetricsStep())
            ->onSuccess(new TriggerWelcomeSequenceStep())
            
            ->onError(new LogOnboardingFailureStep())
            
            ->after(new CleanupTempDataStep());
    }
}

// Uso
$result = $workflow->executeWorkflow($container);

// Output esperado
if ($result->success()) {
    $newUser = $result->getContainer()->get('user');
    $apiToken = $result->getContainer()->get('api_token');
    
    return [
        'user_id' => $newUser->id,
        'email' => $newUser->email,
        'api_token' => $apiToken,
    ];
}
```

---

## 8. Caso de Uso 3: Data Processing Pipeline (ETL)

### 8.1 Workflow con Loops

```php
class DataImportWorkflow
{
    public function build(): Workflow {
        return (new Workflow('data-import'))
            ->prepare(new LoadSourceDataStep())
            
            ->validate(new ValidateFormatStep())
            ->validate(new ValidateSizeStep())
            
            ->before(new BackupExistingDataStep())
            
            ->process(
                (new Loop(new IterateRecordsControl()))
                    ->addStep(new TransformRecordStep())
                    ->addStep(new ValidateRecordStep())
                    ->addStep(new ImportRecordStep())
            )
            
            ->process(new RecalculateAggregatesStep())
            ->process(new RebuildSearchIndexStep())
            
            ->onSuccess(new GenerateImportReportStep())
            ->onSuccess(new NotifyAdminStep())
            
            ->onError(new RollbackImportStep())
            
            ->after(new CleanupTempFilesStep());
    }
}

// Resultado
$result = $workflow->executeWorkflow($container);

if ($result->success()) {
    $stats = $result->getContainer()->get('import_stats');
    echo "Imported: {$stats['success']} records";
}
```

---

## 9. Caso de Uso 4: Approval Workflow

### 9.1 Workflow con Decision Steps

```php
class ApprovalWorkflow
{
    public function build(): Workflow {
        return (new Workflow('expense-approval'))
            ->prepare(new LoadExpenseStep())
            
            ->validate(new ValidateAmountStep())
            ->validate(new ValidatePolicyStep())
            
            ->process(new DetermineApprovalLevelStep())  // Decision
            
            ->process(new NotifyManagerStep())
            ->process(new WaitForApprovalStep())  // Mock: espera input
            
            ->process(new ProcessApprovalResultStep())  // OnSuccess/OnError basado en input
            
            ->onSuccess(new UpdateExpenseStatusStep())
            ->onSuccess(new SendApprovalNotificationStep())
            
            ->onError(new LogRejectionStep())
            ->onError(new SendRejectionNotificationStep())
            
            ->after(new UpdateApprovalMetricsStep());
    }
}
```

---

## 10. Caso de Uso 5: Report Generation

### 10.1 Workflow End-to-End

```php
class ReportGenerationWorkflow
{
    public function build(): Workflow {
        return (new Workflow('report-generation'))
            ->prepare(new ParseReportParametersStep())
            
            ->validate(new ValidateDateRangeStep())
            ->validate(new ValidatePermissionsStep())
            
            ->before(new CacheCheckStep())  // Skip si está en cache
            
            ->process(new FetchDataFromDBStep())
            ->process(new TransformDataStep())
            ->process(
                (new Loop(new IterateChunksControl()))
                    ->addStep(new CalculateMetricsStep())
                    ->addStep(new FormatChunkStep())
            )
            ->process(new GenerateVisualizationsStep())
            ->process(new CreatePDFStep())
            ->process(new UploadToStorageStep())
            
            ->onSuccess(new SendDownloadLinkStep())
            ->onSuccess(new LogReportGenerationStep())
            
            ->onError(new NotifyGenerationFailureStep())
            
            ->after(new UpdateCacheStep());
    }
}

// Resultado
$result = $workflow->executeWorkflow($container);

if ($result->success()) {
    $reportUrl = $result->getContainer()->get('report_url');
    return response()->download($reportUrl);
}
```

---

## 11. Manejo de Errores en Producción

### 11.1 Wrapper para API Responses

```php
class WorkflowApiHandler
{
    public static function execute(
        Workflow $workflow,
        WorkflowContainer $container,
        int $httpSuccessCode = 200
    ) {
        try {
            $result = $workflow->executeWorkflow($container);
            
            if ($result->success()) {
                return response()->json(
                    $result->getContainer()->dump(),
                    $httpSuccessCode
                );
            } else {
                $exception = $result->getException();
                
                // Mapea excepciones a HTTP codes
                $statusCode = $this->mapExceptionToHttpCode($exception);
                
                return response()->json([
                    'error' => $exception->getMessage(),
                    'type' => get_class($exception),
                    'debug' => env('APP_DEBUG') ? $result->debug() : null,
                ], $statusCode);
            }
        } catch (\Exception $e) {
            Log::critical('Unexpected workflow error', [
                'exception' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
            ]);
            
            return response()->json([
                'error' => 'Internal server error',
            ], 500);
        }
    }
    
    private function mapExceptionToHttpCode(\Exception $e): int
    {
        if ($e instanceof ValidationException) return 422;
        if ($e instanceof AuthorizationException) return 403;
        if ($e instanceof ResourceNotFoundException) return 404;
        if ($e instanceof DuplicateResourceException) return 409;
        return 400;
    }
}

// Uso
return WorkflowApiHandler::execute($workflow, $container, 201);
```

### 11.2 Logging Estructurado

```php
class WorkflowLogging
{
    public static function logResult(
        WorkflowResult $result,
        string $workflowName,
        array $context = []
    ): void {
        $level = $result->success() ? 'info' : 'error';
        
        logger($level, "Workflow $workflowName completed", [
            'success' => $result->success(),
            'workflow' => $result->getWorkflowName(),
            'last_step' => get_class($result->getLastStep()),
            'exception' => $result->getException() ? get_class($result->getException()) : null,
            'warnings_count' => count($result->getWarnings()),
            'debug_log' => env('APP_DEBUG') ? $result->debug() : null,
            'context' => $context,
        ]);
    }
}

// Uso
$result = $workflow->executeWorkflow($container);
WorkflowLogging::logResult($result, 'checkout', ['user_id' => $user->id]);
```

---

## 12. Testing Integración con Framework

### 12.1 Testing en Laravel

```php
// tests/Feature/CheckoutWorkflowTest.php
class CheckoutWorkflowTest extends TestCase
{
    use RefreshDatabase;
    
    public function testSuccessfulCheckout(): void
    {
        $user = User::factory()->create();
        $cart = Cart::factory()->forUser($user)->create();
        $address = Address::factory()->forUser($user)->create();
        
        $result = (new CheckoutService(
            app(OrderRepository::class),
            app(PaymentRepository::class),
            app(EmailService::class),
            app(InventoryService::class)
        ))->executeCheckout($user, $cart, $address);
        
        $this->assertTrue($result->success());
        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'status' => 'completed',
        ]);
    }
    
    public function testCheckoutFailsWithInvalidCard(): void
    {
        $user = User::factory()->create();
        $cart = Cart::factory()->forUser($user)->create();
        
        $this->mock(PaymentGateway::class)
            ->shouldReceive('charge')
            ->andThrow(new PaymentGatewayException('Card declined'));
        
        $result = (new CheckoutService(...))->executeCheckout($user, $cart, $address);
        
        $this->assertFalse($result->success());
    }
}
```

---

## 13. Performance en Producción

### 13.1 Caching de Workflows

```php
class CachedWorkflowFactory
{
    private string $cachePrefix = 'workflow.';
    
    public function getCheckoutWorkflow(): Workflow
    {
        $cacheKey = $this->cachePrefix . 'checkout';
        
        return cache()->rememberForever($cacheKey, function () {
            return $this->buildCheckoutWorkflow();
        });
    }
    
    private function buildCheckoutWorkflow(): Workflow
    {
        // Builder logic
    }
}
```

### 13.2 Benchmark

```php
// Medir performance
$start = microtime(true);
$iterations = 1000;

for ($i = 0; $i < $iterations; $i++) {
    $workflow->executeWorkflow($container, false);
}

$duration = (microtime(true) - $start) * 1000;  // ms
$avgTime = $duration / $iterations;

echo "Total: {$duration}ms, Average: {$avgTime}ms per execution";
```

---

## 14. Conclusiones

### 14.1 Puntos Clave

✅ **Haz**:
- Inyecta dependencias en workflows
- Usa factories para crear workflows configurables
- Loguea resultados en producción
- Testa workflows de forma aislada
- Maneja errores explícitamente

❌ **No hagas**:
- Hard-code servicios en workflows
- Ignores resultados de workflows
- Olvides loguear excepciones
- Asumas que workflows siempre exitosos
- Reutilices objetos workflow sin resetear estado

### 14.2 Próximos Pasos

1. **Implementa** un workflow en tu proyecto
2. **Integra** con tu framework existente
3. **Testea** exhaustivamente
4. **Monitorea** en producción
5. **Itera** basado en feedback

---

## 15. Referencias

### Documentos Relacionados

- [Document 02 - Step Definition](02_definicion_pasos_workflow_php-workflow.md)
- [Document 10 - Best Practices](10_best_practices_patrones_workflow_php-workflow.md)
- [Document 11 - Testing](11_testing_strategies_workflow_php-workflow.md)

### Ejemplos en Repositorio

- [tests/WorkflowTest.php](../tests/WorkflowTest.php) - Ejemplos completos
- [README.md](../README.md) - Uso básico

### Links Externos

- [PSR-11: Container interface](https://www.php-fig.org/psr/psr-11/)
- [Laravel Service Providers](https://laravel.com/docs/providers)
- [Symfony Services](https://symfony.com/doc/current/service_container.html)

