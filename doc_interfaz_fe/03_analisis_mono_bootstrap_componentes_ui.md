# 🎨 Análisis: Sistema de Componentes UI de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09  
**Versión de MB**: 1.0.0  
**Última Actualización**: 2026-02-09  
**Estado**: Completado

---

## 🎯 Resumen Ejecutivo

Mono Bootstrap incluye **40+ componentes UI** basados en Bootstrap 5. Todos los componentes tienen:
- Estructura HTML documentada
- Clases CSS reutilizables (prefijo `btn-`, `.card`, `.badge`, etc.)
- Variantes (tamaños, colores, estados)
- Ejemplos funcionales en archivos HTML dedicados

---

## 📋 Inventario de Componentes

### **COMPONENTES BÁSICOS**

| Componente | Archivo | Clases Base | Variantes | Notas |
|-----------|---------|------------|-----------|-------|
| **Botones** | button-default.html | `.btn`, `.btn-primary`, `.btn-secondary` | lg, sm, block, outline | 6+ tipos (default, primary, success, danger, warning, info) |
| **Botones Dropdown** | button-dropdown.html | `.btn-group`, `.dropdown` | - | Menús desplegables en botones |
| **Botones Grupo** | button-group.html | `.btn-group`, `.btn-group-vertical` | - | Agrupaciones verticales/horizontales |
| **Botones Loading** | button-loading.html | `.btn`, `.btn-loader` | - | Animación de loading (Ladda plugin) |
| **Botones Social** | button-social.html | `.btn-social-[platform]` | - | Facebook, Twitter, LinkedIn, etc. |
| **Badge** | badge.html | `.badge`, `.badge-[color]` | primary, secondary, success, danger, warning, info, light, dark | Etiquetas pequeñas |
| **Breadcrumb** | breadcrumb.html | `.breadcrumb`, `.breadcrumb-item` | - | Navegación por ruta |
| **Alerts** | alert.html | `.alert`, `.alert-[type]` | success, warning, danger, info | Dismissible |
| **Card** | card.html | `.card`, `.card-header`, `.card-body`, `.card-footer` | - | Contenedores flexibles |

### **COMPONENTES DE FORMULARIOS**

| Componente | Archivo | Clases Base | Variantes | Notas |
|-----------|---------|------------|-----------|-------|
| **Input Básico** | basic-input.html | `.form-control`, `.form-label` | text, email, password, number, url | Estados: normal, focused, disabled |
| **Checkbox** | checkbox-radio.html | `.form-check`, `.form-check-input` | - | Styled checkboxes |
| **Radio** | checkbox-radio.html | `.form-check`, `.form-check-input[type=radio]` | - | Styled radios |
| **Select** | form-advance.html | `.form-select`, `.form-control` | - | Usa Select2 para avanzado |
| **Validación** | form-validation.html | `.is-valid`, `.is-invalid`, `.invalid-feedback` | - | Estados de validación |
| **Date Input** | form-advance.html | `.form-control.date` | - | Usa DateRangePicker |
| **Textarea** | basic-input.html | `.form-control` | - | Multilínea |

### **COMPONENTES DE DATOS**

| Componente | Archivo | Clases Base | Notas |
|-----------|---------|------------|-------|
| **Tabla Básica** | bootstarp-tables.html | `.table`, `.table-hover`, `.table-striped` | Bootstrap nativo |
| **Tabla Avanzada** | data-tables.html | `.table`, `[data-table]` | Con DataTables plugin, sorting, paging |
| **Badge en Tabla** | badge.html | (dentro de tablas) | Estados, colores variados |
| **Card Grid** | card.html | `.row`, `.col`, `.card` | Layout grid de cards |

### **COMPONENTES DE NAVEGACIÓN**

| Componente | Archivo | Clases Base | Notas |
|-----------|---------|------------|-------|
| **Navbar** | index.html | `.navbar`, `.navbar-expand` | Top navigation |
| **Sidebar** | index.html | `.sidebar`, `.sidebar-nav` | Lateral navigation, collapsible |
| **Pagination** | pagination.html | `.pagination`, `.page-item`, `.page-link` | Numerada, previous/next |
| **Tabs** | tabs.html | `.nav`, `.nav-tabs`, `.tab-content` | Content switching |
| **Pills** | pills.html | `.nav`, `.nav-pills` | Similar a tabs |
| **Breadcrumb** | breadcrumb.html | `.breadcrumb` | Ruta de ubicación |

### **COMPONENTES OVERLAY/MODALES**

| Componente | Archivo | Clases Base | Notas |
|-----------|---------|------------|-------|
| **Modal** | modal.html | `.modal`, `.modal-dialog`, `.modal-content` | Diálogos, popups |
| **Collapse** | collapse.html | `.collapse`, `[data-bs-toggle=collapse]` | Expandible/colapsible |
| **Dropdown** | button-dropdown.html | `.dropdown`, `.dropdown-menu` | Menús desplegables |
| **Tooltip** | tooltip.html | `[data-bs-toggle=tooltip]` | Hints de Bootstrap |
| **Popover** | popover.html | `[data-bs-toggle=popover]` | Más info flotante |

### **COMPONENTES UTILITARIOS**

| Componente | Archivo | Clases Base | Notas |
|-----------|---------|------------|-------|
| **Progress Bar** | progress.html | `.progress`, `.progress-bar` | Barras de progreso, porcentajes |
| **Spinner/Loader** | spinner.html | `.spinner-border`, `.spinner-grow` | Animaciones de carga |
| **Badge** | badge.html | `.badge` | Etiquetas, contadores |
| **Carousel** | carousel.html | `.carousel`, `.carousel-item` | Sliders de imágenes (Owl Carousel) |

### **COMPONENTES ESPECIALIZADOS**

| Componente | Archivo | Plugins | Notas |
|-----------|---------|---------|-------|
| **Editor WYSIWYG** | editor.html | Quill | Editor de texto enriquecido |
| **Calendar** | calendar.html | FullCalendar | Calendario de eventos |
| **Map Vectorial** | google-maps.html | jVectorMap | Mapas interactivos |
| **Charts** | apex-charts.html | ApexCharts | Gráficos dinámicos (línea, pie, radar, etc.) |
| **Code Highlight** | code-example.html | Prism | Resaltador de sintaxis |

---

## 🎨 Clases CSS Base y Nomenclatura

### **Convención de Clases Bootstrap**
```
.btn                          # Botón base
.btn-primary                  # Variante primaria
.btn-lg                       # Tamaño grande
.btn-outline-primary          # Outline style
.btn-block                    # Ancho 100%
.btn-disabled                 # Deshabilitado
```

### **Colores Estándar**
```
primary    (azul)
secondary  (gris)
success    (verde)
danger     (rojo)
warning    (amarillo/naranja)
info       (celeste)
light      (blanco/gris claro)
dark       (gris oscuro/negro)
```

### **Tamaños**
```
.small, .sm     # Pequeño
(default)       # Mediano
.lg             # Grande
.xl             # Extra grande
```

---

## 📊 Ejemplos de Estructura HTML

### **Botón**
```html
<button class="btn btn-primary">Primary Button</button>
<button class="btn btn-primary btn-lg">Large Button</button>
<button class="btn btn-outline-primary">Outline</button>
<button class="btn btn-primary" disabled>Disabled</button>
```

### **Card**
```html
<div class="card">
  <div class="card-header">
    <h5 class="card-title">Title</h5>
  </div>
  <div class="card-body">
    Content goes here
  </div>
  <div class="card-footer">
    Footer info
  </div>
</div>
```

### **Tabla**
```html
<table class="table table-striped table-hover">
  <thead>
    <tr>
      <th>Header 1</th>
      <th>Header 2</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Data 1</td>
      <td>Data 2</td>
    </tr>
  </tbody>
</table>
```

### **Modal**
```html
<div class="modal fade" id="exampleModal">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Modal Title</h5>
        <button class="btn-close" data-bs-dismiss="modal"></button>
      </div>
      <div class="modal-body">
        Content...
      </div>
      <div class="modal-footer">
        <button class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
        <button class="btn btn-primary">Save changes</button>
      </div>
    </div>
  </div>
</div>
```

---

## ✅ Checklist - TEMA 3 Completado

- ✅ Inventario completo de componentes (40+)
- ✅ Archivos HTML donde encontrar cada componente
- ✅ Clases CSS documentadas
- ✅ Variantes listadas
- ✅ Ejemplos de estructura HTML
- ✅ Convención de nombrado

---

**Próximo**: TEMA 4 - Layouts y Templates
