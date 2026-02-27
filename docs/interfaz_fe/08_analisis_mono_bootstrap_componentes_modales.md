# 🔲 Análisis: Componentes Modales y Overlays de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Modal

```html
<div class="modal fade" id="exampleModal">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Modal Title</h5>
        <button class="btn-close" data-bs-dismiss="modal"></button>
      </div>
      <div class="modal-body">
        Modal content...
      </div>
      <div class="modal-footer">
        <button class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
        <button class="btn btn-primary">Save</button>
      </div>
    </div>
  </div>
</div>

<!-- Trigger -->
<button data-bs-toggle="modal" data-bs-target="#exampleModal">Open</button>
```

**Tamaños**: `.modal-sm`, `.modal-lg`, `.modal-xl`

**Variantes**:
- `.modal-fullscreen` - Pantalla completa
- `.modal-dialog-centered` - Verticalmente centrado
- `.modal-dialog-scrollable` - Scroll en body

## Collapse (Expandible)

```html
<button data-bs-toggle="collapse" data-bs-target="#collapseExample">
  Toggle
</button>

<div class="collapse" id="collapseExample">
  <div class="card card-body">
    Content to expand/collapse
  </div>
</div>
```

## Dropdown

```html
<div class="dropdown">
  <button class="btn btn-secondary dropdown-toggle" 
          data-bs-toggle="dropdown">
    Dropdown button
  </button>
  <ul class="dropdown-menu">
    <li><a class="dropdown-item" href="#">Action</a></li>
    <li><a class="dropdown-item" href="#">Another action</a></li>
    <li><hr class="dropdown-divider"></li>
    <li><a class="dropdown-item" href="#">Something else</a></li>
  </ul>
</div>
```

## Tooltip

```html
<button type="button" class="btn btn-secondary" 
        data-bs-toggle="tooltip" 
        data-bs-title="Tooltip on top">
  Hover me
</button>

<script>
  // Initialize
  const tooltipList = document.querySelectorAll('[data-bs-toggle="tooltip"]')
  [...tooltipList].map(tooltip => new bootstrap.Tooltip(tooltip))
</script>
```

**Posiciones**: `data-bs-placement="top|bottom|left|right"`

## Popover

```html
<button type="button" class="btn btn-lg btn-danger" 
        data-bs-toggle="popover" 
        data-bs-title="Popover title"
        data-bs-content="Popover content">
  Click to toggle popover
</button>
```

**Similar a Tooltip pero con más contenido**

## ✅ Checklist - TEMA 8 Completado

- ✅ Modal estructura y variantes
- ✅ Collapse/accordion
- ✅ Dropdown menus
- ✅ Tooltips
- ✅ Popovers
- ✅ Interactividad Bootstrap JS
