# Ejemplo de archivo XML de definición de Workflow

A continuación se muestra un ejemplo básico de cómo definir un workflow en un archivo XML para php-workflow. Este archivo describe las actividades y las transiciones entre ellas.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<workflow>
    <activity name="Inicio" class="App\Workflow\Activity\InicioActivity" />
    <activity name="ValidarDatos" class="App\Workflow\Activity\ValidarDatosActivity" />
    <activity name="Procesar" class="App\Workflow\Activity\ProcesarActivity" />
    <activity name="Fin" class="App\Workflow\Activity\FinActivity" />

    <transition from="Inicio" to="ValidarDatos" />
    <transition from="ValidarDatos" to="Procesar" condition="success" />
    <transition from="ValidarDatos" to="Fin" condition="error" />
    <transition from="Procesar" to="Fin" />
</workflow>
```

## Explicación rápida
- **activity**: Define una actividad (paso) del workflow, con su nombre y la clase PHP que la implementa.
- **transition**: Define la transición entre actividades. Puede incluir condiciones para ramificar el flujo.

Este archivo debe guardarse, por ejemplo, como `workflows/ejemplo.xml`.

En los siguientes pasos se explicará cómo implementar las clases de actividad y cómo cargar este workflow en tu aplicación.
