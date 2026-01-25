# Workflow: API Prompt Workflow
**Fecha de creación:** 2026-01-23 20:44

---

## Estructura del Workflow en XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<workflow name="APIPromptWorkflow">
    <!-- Paso 1: Ejecutar primer prompt contra API OpenAI -->
    <activity name="EjecutarPromptP1" class="App\Workflow\Activity\EjecutarPromptP1Activity" />
    
    <!-- Paso 2: Procesar y dividir JSON en registros -->
    <activity name="ProcesarJSON" class="App\Workflow\Activity\ProcesarJSONActivity" />
    
    <!-- Paso 3: Presentar registros en formulario F1 -->
    <activity name="MostrarFormularioF1" class="App\Workflow\Activity\MostrarFormularioF1Activity" />
    
    <!-- Paso 4: Usuario selecciona un registro -->
    <activity name="SeleccionarRegistro" class="App\Workflow\Activity\SeleccionarRegistroActivity" />
    
    <!-- Paso 5: Ejecutar segundo prompt P2 con registro seleccionado -->
    <activity name="EjecutarPromptP2" class="App\Workflow\Activity\EjecutarPromptP2Activity" />
    
    <!-- Paso 6: Mostrar resultado en formulario F2 -->
    <activity name="MostrarFormularioF2" class="App\Workflow\Activity\MostrarFormularioF2Activity" />
    
    <!-- Paso 7: Usuario decide si proceder -->
    <activity name="DecisionProceder" class="App\Workflow\Activity\DecisionProcederActivity" />
    
    <!-- Paso 8: Ejecutar prompt P3 y obtener resultado en Markdown (FMD1) -->
    <activity name="EjecutarPromptP3" class="App\Workflow\Activity\EjecutarPromptP3Activity" />
    
    <!-- Paso 9: Ejecutar prompt P4 y obtener resultado en Markdown (FMD2) -->
    <activity name="EjecutarPromptP4" class="App\Workflow\Activity\EjecutarPromptP4Activity" />
    
    <!-- Paso 10: Ejecutar SQL INSERT con resultados FMD1 y FMD2 -->
    <activity name="EjecutarSQLInsert" class="App\Workflow\Activity\EjecutarSQLInsertActivity" />
    
    <!-- Actividad de finalización -->
    <activity name="Finalizar" class="App\Workflow\Activity\FinalizarActivity" />
    
    <!-- Transiciones entre actividades -->
    <transition from="EjecutarPromptP1" to="ProcesarJSON" />
    <transition from="ProcesarJSON" to="MostrarFormularioF1" />
    <transition from="MostrarFormularioF1" to="SeleccionarRegistro" />
    <transition from="SeleccionarRegistro" to="EjecutarPromptP2" />
    <transition from="EjecutarPromptP2" to="MostrarFormularioF2" />
    <transition from="MostrarFormularioF2" to="DecisionProceder" />
    
    <!-- Decisión: si el usuario decide proceder -->
    <transition from="DecisionProceder" to="EjecutarPromptP3" condition="proceder" />
    
    <!-- Decisión: si el usuario decide NO proceder, vuelve a F1 -->
    <transition from="DecisionProceder" to="MostrarFormularioF1" condition="no_proceder" />
    
    <transition from="EjecutarPromptP3" to="EjecutarPromptP4" />
    <transition from="EjecutarPromptP4" to="EjecutarSQLInsert" />
    
    <!-- Paso 11: Después del INSERT, volver a F1 para seleccionar otro registro -->
    <transition from="EjecutarSQLInsert" to="MostrarFormularioF1" condition="continuar" />
    
    <!-- Opción de finalizar después del INSERT -->
    <transition from="EjecutarSQLInsert" to="Finalizar" condition="finalizar" />
</workflow>
```

---

## Diagrama de flujo

```
EjecutarPromptP1 → ProcesarJSON → MostrarFormularioF1 → SeleccionarRegistro → EjecutarPromptP2 
→ MostrarFormularioF2 → DecisionProceder
                            ↓ (proceder)         ↓ (no_proceder)
                    EjecutarPromptP3      MostrarFormularioF1
                            ↓
                    EjecutarPromptP4
                            ↓
                    EjecutarSQLInsert
                    ↓ (continuar)  ↓ (finalizar)
          MostrarFormularioF1    Finalizar
```

---

## Descripción de las actividades

### 1. EjecutarPromptP1
- **Objetivo**: Ejecutar el primer prompt (P1) contra la API de OpenAI.
- **Entrada**: Prompt P1 configurado.
- **Salida**: Respuesta en formato JSON.
- **Comportamiento**: Envía una petición síncrona a la API de OpenAI y espera el resultado.

### 2. ProcesarJSON
- **Objetivo**: Procesar y dividir el JSON recibido en registros individuales.
- **Entrada**: JSON completo de la API.
- **Salida**: Array de registros procesados.
- **Comportamiento**: Parsea el JSON y extrae los registros en un formato manejable.

### 3. MostrarFormularioF1
- **Objetivo**: Presentar los registros al usuario en un formulario web (F1).
- **Entrada**: Array de registros.
- **Salida**: Interfaz web con lista de registros.
- **Comportamiento**: Renderiza un formulario HTML con los registros disponibles para selección.

### 4. SeleccionarRegistro
- **Objetivo**: Capturar la selección del usuario.
- **Entrada**: Selección del usuario desde F1.
- **Salida**: Registro seleccionado.
- **Comportamiento**: Espera interacción del usuario y valida la selección.

### 5. EjecutarPromptP2
- **Objetivo**: Ejecutar el segundo prompt (P2) con el registro seleccionado.
- **Entrada**: Registro seleccionado por el usuario.
- **Salida**: Respuesta de la API.
- **Comportamiento**: Envía P2 a la API de OpenAI incluyendo el registro seleccionado.

### 6. MostrarFormularioF2
- **Objetivo**: Mostrar el resultado de P2 al usuario en el formulario F2.
- **Entrada**: Resultado de P2.
- **Salida**: Interfaz web con el resultado.
- **Comportamiento**: Renderiza el resultado en un formulario para revisión del usuario.

### 7. DecisionProceder
- **Objetivo**: Permitir al usuario decidir si continúa o no.
- **Entrada**: Decisión del usuario desde F2.
- **Salida**: Condición "proceder" o "no_proceder".
- **Comportamiento**: Si el usuario decide no proceder, vuelve a F1. Si procede, continúa al siguiente paso.

### 8. EjecutarPromptP3
- **Objetivo**: Ejecutar el tercer prompt (P3) con el resultado de F2.
- **Entrada**: Resultado de F2.
- **Salida**: Resultado en formato Markdown (FMD1).
- **Comportamiento**: Envía P3 a la API y recibe FMD1.

### 9. EjecutarPromptP4
- **Objetivo**: Ejecutar el cuarto prompt (P4) con el resultado de F2.
- **Entrada**: Resultado de F2.
- **Salida**: Resultado en formato Markdown (FMD2).
- **Comportamiento**: Envía P4 a la API y recibe FMD2.

### 10. EjecutarSQLInsert
- **Objetivo**: Insertar en base de datos los resultados concatenados.
- **Entrada**: FMD1 y FMD2.
- **Salida**: Confirmación de INSERT exitoso.
- **Comportamiento**: Concatena FMD1 y FMD2, ejecuta SQL INSERT en la base de datos (que puede estar en servidor externo).

### 11. Bucle de continuación
- **Objetivo**: Permitir procesar múltiples registros del JSON inicial.
- **Comportamiento**: Después del INSERT, el workflow puede volver a F1 para seleccionar otro registro o finalizar.

### 12. Finalizar
- **Objetivo**: Terminar el workflow.
- **Comportamiento**: Cierra el workflow y libera recursos.

---

## Puntos de decisión

1. **DecisionProceder**: El usuario decide si continúa con P3 y P4, o vuelve a F1.
2. **EjecutarSQLInsert**: Después del INSERT, el usuario puede continuar procesando registros o finalizar.

---

## Requisitos técnicos

- Conexión a API de OpenAI configurada.
- Servidor web para renderizar formularios F1 y F2.
- Conexión a base de datos (puede ser remota).
- Soporte para ejecución síncrona de peticiones HTTP.

---

## Características del workflow

- **Ejecución síncrona**: Cada llamada a la API de OpenAI espera respuesta antes de continuar.
- **Intervención manual**: El usuario debe interactuar en los puntos de decisión (F1, F2, DecisionProceder).
- **Base de datos externa**: El SQL INSERT puede ejecutarse en una base de datos fuera del servidor PHP.
- **Procesamiento múltiple**: El workflow permite procesar varios registros del JSON inicial sin reiniciar.

---

## Próximos pasos

1. Implementar las clases de actividad para cada paso.
2. Configurar la conexión a la API de OpenAI.
3. Crear las vistas/formularios F1 y F2.
4. Configurar la conexión a la base de datos.
5. Implementar la lógica de condiciones en las transiciones.
