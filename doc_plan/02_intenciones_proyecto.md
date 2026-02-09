# 📌 Intenciones e Ideas del Proyecto Atlas Panel

**Fecha de Creación**: 2026-02-09  
**Última Actualización**: 2026-02-09  
**Estado**: En Desarrollo

---

## 🎯 Objetivo General

Crear una herramienta standalone basada en **php-workflow (Atlas)** que permita ejecutar procesos/APIs de forma modular mediante un panel de administración.

---

## 📋 Definiciones Confirmadas

### **Atlas (php-workflow) - Rol del Proyecto**
- Herramienta **standalone**, no integrada en otros proyectos
- Ejecutor de procesos/workflows
- Motor base para toda la funcionalidad del panel

### **alcance del proyecto**
- No se integrará en otros desarrollos (v1)
- Sin dependencias de proyectos externos (v1)

---

## 🖥️ Panel de Administración

**Ubicación**: `/admin`

**Propósito**: Facilitar la creación, configuración y gestión de workflows sin escribir código

**Puntos a considerar**:
- [ ] Estructura de directorios del panel
- [ ] Punto de entrada (archivo principal)
- [ ] Organización de vistas/templates
- [ ] Gestión de assets estáticos (CSS, JS, imágenes)
- [ ] Navegación entre secciones

---

## 👤 Flujo de Usuario - Crear y Gestionar Workflows

**Secuencia confirmada**:

1. **Acceder al panel** → FE en `/admin`
2. **Crear un workflow** → Nombre y descripción
3. **Añadir pasos al workflow** → Definir qué hace cada paso
4. **Encadenar pasos** → Configurar entrada/salida entre pasos
5. **Probar funcionamiento** → Ejecutar y ver resultados
6. **Pasar a producción** → Workflow listo para ser ejecutado

**Puntos a considerar**:
- [ ] Interfaz para crear workflows
- [ ] Interfaz para añadir/editar steps
- [ ] Interfaz para encadenar steps (mapeo de datos)
- [ ] Interfaz para pruebas/testing
- [ ] Estados de workflow (draft, testing, production)
- [ ] Indicadores visuales de estado

---

## 🚀 Versión 1 (v1) - Capacidades Iniciales

### **Steps Soportados en v1**

**Definición**: Unidad de trabajo indivisible, atómica, modular

**Casos de uso confirmados**:
1. Llamadas a **OpenAI API** (ejecución de prompts)
2. Inserciones **HTTP** (GET/POST/PUT/DELETE)
3. Triggers desde **formularios de terceros**

**Funcionalidades NO incluidas en v1**:
- ❌ Loops
- ❌ Middleware
- ❌ Nested Workflows
- ❌ Dependencias complejas entre steps
- ❌ Sistema de Login/Autenticación
- ❌ Base de Datos relacional

### **Características de Steps en v1**

**Puntos a considerar**:
- [ ] Tipos de steps disponibles (OpenAI, HTTP, etc.)
- [ ] Parámetros configurables por tipo de step
- [ ] Validación de parámetros
- [ ] Prueba aislada de un step
- [ ] Reutilización de steps
- [ ] Mapeo de datos (input/output)

---

## 🔄 Desarrollo Incremental

**Estrategia confirmada**: Versiones progresivas de Atlas que añaden capacidades

**Versiones futuras**:
- **v1.1+**: Expansiones de v1 (a definir)
- **v2.0+**: Nuevas capacidades (a definir)
- **vN.0+**: Evolución según necesidades

**Aprendizaje del usuario**: A medida que aumentan versiones, el usuario aprende nuevas capacidades de forma progresiva

**Puntos a considerar**:
- [ ] Roadmap de versiones futuras
- [ ] Deprecación de features (si aplica)
- [ ] Migración entre versiones
- [ ] Documentación evolutiva

---

## 📁 Almacenamiento v1

**Tecnología**: JSON files (simula base de datos)

**Ubicación**: `/data/`

**Puntos a considerar**:
- [ ] Estructura de carpetas para almacenamiento
- [ ] Esquema de archivos JSON
- [ ] Metadatos necesarios (timestamps, versiones, etc.)
- [ ] Validación de integridad de datos
- [ ] Migración futura a BD relacional
- [ ] Backup y recuperación

---

## 🏗️ Arquitectura Técnica

### **Stack Seleccionado**

| Componente | Tecnología |
|---|---|
| **Backend** | PHP (server-side rendering) |
| **Frontend** | Bootstrap + Vanilla JS |
| **Router** | Router PHP personalizado |
| **Almacenamiento** | JSON files (v1) |
| **Panel Admin** | Mono Bootstrap template |

### **Patrones de Comunicación**

- **FE → BE**: HTTP Requests (formularios, AJAX con Vanilla JS)
- **Respuesta BE**: HTML renderizado server-side o JSON (si AJAX)
- **Persistencia**: JSON files en `/data/`

**Puntos a considerar**:
- [ ] Endpoints específicos necesarios
- [ ] Formatos de petición/respuesta
- [ ] Manejo de errores HTTP
- [ ] Validación de datos en servidor
- [ ] Seguridad de endpoints

---

## 🎨 Panel Admin - Mono Bootstrap

**Estado**: Aún no incorporado (Fase 1: Análisis)

**Puntos a considerar**:
- [ ] Estructura interna del template
- [ ] Componentes UI disponibles
- [ ] Dependencias (CSS, JS, librerías)
- [ ] Cómo adaptarlo al proyecto
- [ ] Ubicación de assets
- [ ] Personalización necesaria

---

## 🔌 Backend - Servicios Necesarios

**Puntos a considerar**:
- [ ] Manejo de workflows (crear, leer, actualizar, eliminar)
- [ ] Manejo de steps dentro de workflows
- [ ] Ejecución de workflows
- [ ] Historial de ejecuciones
- [ ] Logging y debugging
- [ ] Validación de configuraciones
- [ ] Integración con Atlas core

---

## 📊 Estructura del Proyecto

```
/workspaces/atlas/
├── /admin/              ← Panel administrativo
├── /src/                ← Core de Atlas (existente)
├── /data/               ← Almacenamiento JSON (v1)
├── /public/             ← Assets estáticos
├── /doc_plan/           ← Documentación de plan
└── [otros]
```

**Puntos a considerar**:
- [ ] Carpeta `/api/` para endpoints
- [ ] Estructura de rutas
- [ ] Organización de controladores/servicios
- [ ] Inyección de dependencias (si aplica)

---

## 🔄 Próximas Fases

**Pendiente de definir**:
- Detalles específicos de Fase 1 (Análisis Mono Bootstrap)
- Cronograma de desarrollo
- Recursos necesarios
- Criterios de éxito para cada fase

---

**Nota**: Este documento se actualiza conforme se confirmen nuevas intenciones o ideas del proyecto.
