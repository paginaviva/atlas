# 📊 Análisis: Componentes de Datos de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Tablas

### **Tabla Básica**
```html
<table class="table">
  <thead>
    <tr>
      <th>ID</th>
      <th>Name</th>
      <th>Email</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>John</td>
      <td>john@example.com</td>
    </tr>
  </tbody>
</table>
```

**Variantes**:
- `.table-striped` - Filas alternas coloreadas
- `.table-hover` - Hover en filas
- `.table-dark` - Tema oscuro
- `.table-sm` - Compacta
- `.table-borderless` - Sin bordes

### **Tabla Avanzada (DataTables)**
- Sorting: columnas ordenables
- Pagination: paginación automática
- Search: búsqueda global
- Info: mostrando X de Y registros
- Ejemplo: `data-tables.html`

## Cards

```html
<div class="card">
  <div class="card-header">
    <h5 class="card-title">Card Title</h5>
  </div>
  <ul class="list-group list-group-flush">
    <li class="list-group-item">Item 1</li>
    <li class="list-group-item">Item 2</li>
  </ul>
  <div class="card-body">
    Card body content
  </div>
  <div class="card-footer">
    Footer text
  </div>
</div>
```

**Variantes**:
- `.card-header` - Encabezado coloreado
- `.card-footer` - Pie de card
- `.card-body` - Contenido principal
- `.card-text` - Párrafos
- `.card-link` - Enlaces

## Listas

```html
<!-- Unordered -->
<ul class="list-group">
  <li class="list-group-item">Item 1</li>
  <li class="list-group-item active">Item 2 (active)</li>
  <li class="list-group-item">Item 3</li>
</ul>

<!-- Ordered -->
<ol class="list-group list-group-numbered">
  <li class="list-group-item">First item</li>
  <li class="list-group-item">Second item</li>
</ol>
```

## Badges, Labels, Tags

```html
<!-- Badges -->
<span class="badge bg-primary">Primary</span>
<span class="badge bg-success rounded-pill">Success Pill</span>

<!-- En lista -->
<li class="d-flex justify-content-between">
  Item <span class="badge bg-primary">5</span>
</li>
```

**Colores**: primary, secondary, success, danger, warning, info, light, dark

## Alerts

```html
<!-- Success Alert -->
<div class="alert alert-success alert-dismissible" role="alert">
  Success message!
  <button class="btn-close" data-bs-dismiss="alert"></button>
</div>

<!-- Tipos -->
.alert-success, .alert-danger, .alert-warning, .alert-info, .alert-light, .alert-dark
```

## Indicadores de Estado/Progreso

```html
<!-- Progress Bar -->
<div class="progress">
  <div class="progress-bar" style="width: 25%">25%</div>
</div>

<!-- Spinner -->
<div class="spinner-border" role="status">
  <span class="visually-hidden">Loading...</span>
</div>

<!-- Badge Contador -->
<span class="badge bg-danger">99+</span>
```

## ✅ Checklist - TEMA 7 Completado

- ✅ Tablas básicas y avanzadas
- ✅ Cards structure
- ✅ Listas
- ✅ Badges y tags
- ✅ Alerts
- ✅ Indicadores de progreso
