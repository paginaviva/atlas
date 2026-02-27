# ⚖️ Reglas y Directrices para el Proyecto

**Fecha de Creación**: 2026-02-09  
**Última Actualización**: 2026-02-09  
**Estado**: En Vigencia  
**Propósito**: Marco normativo que encuadra el trabajo de desarrollo

---

## 📌 Reglas del Usuario

### **Comunicación y Aprobación**

1. **No asumir decisiones técnicas**: Préguntar antes de implementar arquitecturas, patrones o tecnologías no confirmadas
2. **Solicitar aprobación antes de ejecutar**: Especialmente en cambios estructurales o creación de archivos/carpetas
3. **Documentar decisiones**: Guardar confirmaciones del usuario en los documentos de referencia
4. **Reportar estado**: Informar del progreso antes de pasar a siguiente fase
5. **Escalar dudas**: Si algo es ambiguo, preguntar explícitamente en lugar de asumir

### **Revisión de Código/Documentos**

6. **Revisión obligatoria**: El usuario revisa y aprueba antes de pasar a siguiente paso
7. **Correcciones iterativas**: Aceptar feedback y corregir sin resistencia
8. **Validar supuestos**: Confirmar interpretación de requisitos

---

## 📋 Reglas del Proyecto

### **Scope y Delimitación**

1. **Minimalismo**: Solo lo estrictamente necesario para v1, sin especular sobre versiones futuras
2. **Un solo objetivo por fase**: Enfocarse en una tarea a la vez
3. **Estructura escalable**: Preparar directorio/archivos para crecimiento futuro sin implementar todo ahora
4. **Independencia inicial**: Workflows v1 sin dependencias externas (no integrar con otros proyectos)

### **Versiones y Roadmap**

5. **No inventar versiones**: Solo considerar lo confirmado (v1 = steps básicos + OpenAI + HTTP)
6. **Separación clara**: Distinguir entre "lo que es v1" y "lo que será futuro"
7. **No adelantar trabajo**: No implementar características de v2+ en v1

### **Especulación y Vacíos**

8. **No rellenar espacios**: Si no está confirmado, dejar como "punto a considerar"
9. **No especular arquitectura**: No inventar endpoints, esquemas JSON, o patrones sin confirmación
10. **Listas de verificación**: Usar en lugar de detalles inventados para aspectos pendientes

---

## 🎨 Reglas para el Frontend (FE)

### **Tecnologías y Dependencias**

1. **Stack confirmado**: Bootstrap + Vanilla JS + Servidor PHP (renderizado server-side)
2. **Sin frameworks adicionales**: No agregar React, Vue u otro framework sin autorización
3. **Dependencias mínimas**: Solo Bootstrap y sus requisitos core
4. **Assets estáticos**: Organizar en `/public/` con estructura clara

### **Estructura y Organización**

5. **Blade o templates simples**: PHP puro o template engine simple (no externo)
6. **Modularidad en vistas**: Separar componentes reutilizables
7. **Nombres descriptivos**: Carpetas y archivos reflejen su propósito

### **Interacción y UX**

8. **Vanilla JS**: Validaciones cliente, mostrar/ocultar elementos, confirmaciones
9. **Accesibilidad básica**: Seguir estándares Bootstrap
10. **No complejidad innecesaria**: Mantener simplicidad en el cliente

### **Comunicación con Backend**

11. **Formularios POST/GET**: Preferencia por HTML tradicional
12. **AJAX opcional**: Solo si se requiere experiencia mejorada (aún con Vanilla JS)
13. **Validación doble**: Cliente (UX) + Servidor (seguridad)

---

## 🔌 Reglas para el Backend (BE)

### **Arquitectura**

1. **Router PHP único**: Punto de entrada centralizado
2. **Separación de concerns**: Controladores, servicios, modelos (si aplica)
3. **Reutilización del core Atlas**: Integrar php-workflow sin modificarlo innecesariamente

### **APIs y Endpoints**

4. **RESTful básico**: Seguir convenciones HTTP (GET/POST/PUT/DELETE)
5. **Respuestas consistentes**: JSON para AJAX, HTML para formularios tradicionales
6. **Manejo de errores**: Códigos HTTP apropiados y mensajes claros

### **Persistencia**

7. **JSON files v1**: Usar `/data/` sin base de datos relacional
8. **Integridad de datos**: Validar antes de guardar
9. **Versionado simple**: Timestamps para auditoría básica

---

## 📚 Reglas de Documentación

### **Creación y Mantenimiento**

1. **Documentación es código**: Tan importante como el código ejecutable
2. **Ubicaciones claras**: 
   - `/doc_plan/` → Planificación y decisiones
   - `/doc_desarrollo/` → Guías técnicas y referencias (existente)
   - Inline comments → Código complejo

3. **Actualización inmediata**: No deja documentación desfasada
4. **Referencias en archivos**: Enlazar entre documentos relevantes

### **Contenido**

5. **No inventar detalles**: Solo documentar lo realizado o confirmado
6. **Puntos a considerar**: Usar listas para aspectos pendientes
7. **Cambios registrados**: Todo cambio importante entra en documentación
8. **Legibilidad primero**: Markdown limpio, bien estructurado

### **Formato**

9. **Estructura consistente**: Encabezados jerárquicos, tablas, listas
10. **Ejemplos concretos**: Cuando sea relevante, incluir ejemplos reales
11. **Secciones atómicas**: Cada sección debe ser entendible por sí sola

---

## 📅 Reglas de Planificación

### **Fases y Entregas**

1. **Una fase a la vez**: Completar fase antes de iniciar siguiente
2. **Entregas definidas**: Cada fase tiene output/entregable claro
3. **Revisión y aprobación**: Usuario aprueba antes de avanzar
4. **Sin solapamiento**: Evitar trabajar en múltiples fases simultaneamente

### **Roadmap y Versiones**

5. **v1 solo v1**: Enfocarse únicamente en capacidades v1
6. **Roadmap futuro**: Documentar ideas para futuro sin implementar
7. **No promesas**: No comprometer features sin confirmación técnica

### **Cambios de Alcance**

8. **Cambios documentados**: Cualquier variación se registra
9. **Renegociar si es necesario**: Grandes cambios requieren replanning
10. **Traceabilidad**: Saber por qué cambió algo y cuándo

---

## ✅ Reglas Básicas y Generalistas

### **Actitud y Metodología**

1. **Transparencia total**: Informar estado real, no esconder problemas
2. **Preguntar > asumir**: Mejor una pregunta "tonta" que un trabajo en vano
3. **Iterativo o incremental**: Avanzar paso a paso, validando frecuentemente
4. **Humildad**: Aceptar correcciones y mejorar constantemente

### **Código y Archivos**

5. **Limpieza**: No dejar archivos temporales, comentarios residuales o código muerto
6. **Convenciones**: Seguir estándares de nombrado (snake_case, PascalCase, etc.)
7. **Comentarios útiles**: Explicar el "por qué", no el "qué" obvio
8. **Evitar duplicación**: Reutilizar, no copiar-pegar

### **Errores y Problemas**

9. **Reportar inmediato**: Si algo falla, avisaer de inmediato
10. **Proponer solución**: No solo error, sino camino forward
11. **Logging y debugging**: Dejar trail para troubleshooting

### **Gestión de Entregables**

12. **Lista de tareas**: Usar `manage_todo_list` para trabajo complejo
13. **Marcar completadas**: Actualizar estado inmediatamente tras terminar
14. **No batch**: Marcar items conforme se completan, no al final

### **Herramientas y Helpers**

15. **Usar `manage_todo_list`**: Para multi-step work
16. **Usar `multi_replace_string_in_file`**: Para múltiples edits eficientes
17. **Parallelizar reads**: Lectura de múltiples archivos simultánea cuando sea posible
18. **No ejecutar sin contexto**: Leer primero, luego actuar

---

## 🔄 Actualización de este Documento

- Se revisa y actualiza cuando nuevas reglas emerjan
- Se consulta antes de iniciar cualquier fase
- Se amplía conforme avanza el proyecto
- Usuario puede proponer nuevas reglas

---

**Base**: Este documento es el marco normativo. Todas las decisiones se toman dentro de este encuadre.
