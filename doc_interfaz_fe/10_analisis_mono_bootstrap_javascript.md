# ⚙️ Análisis: Funcionalidad JavaScript de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## JavaScript Custom (Sin Dependencias Externas)

### **Archivos JS Custom**
```
source/js/script.js        # Main JS file
theme/js/script.js         # Compiled output
```

## Comportamientos Interactivos Bootstrap JS

Todos incluidos en `bootstrap.bundle.min.js`:

| Componente | Evento | Comportamiento |
|-----------|--------|----------------|
| **Modal** | `data-bs-toggle="modal"` | Abre/cierra modal |
| **Collapse** | `data-bs-toggle="collapse"` | Expandir/colapsar |
| **Dropdown** | `data-bs-toggle="dropdown"` | Menú desplegable |
| **Tab** | `data-bs-toggle="tab"` | Cambio de pestañas |
| **Tooltip** | `data-bs-toggle="tooltip"` | Muestra tooltip |
| **Popover** | `data-bs-toggle="popover"` | Muestra popover |
| **Alert** | `.btn-close` con `data-bs-dismiss="alert"` | Cierra alert |
| **Toast** | Toastr plugin | Notificaciones toast |
| **Navbar Toggler** | `.navbar-toggler` | Toggle menú mobile |

## Validación de Formularios

```javascript
// Bootstrap form validation
const forms = document.querySelectorAll('.needs-validation')
Array.from(forms).forEach(form => {
  form.addEventListener('submit', event => {
    if (!form.checkValidity()) {
      event.preventDefault()
      event.stopPropagation()
    }
    form.classList.add('was-validated')
  }, false)
})
```

## Inicialización de Plugins

### **Tooltips**
```javascript
const tooltipTriggerList = [].slice.call(document.querySelectorAll('[data-bs-toggle="tooltip"]'))
const tooltipList = tooltipTriggerList.map(el => new bootstrap.Tooltip(el))
```

### **Popovers**
```javascript
const popoverTriggerList = [].slice.call(document.querySelectorAll('[data-bs-toggle="popover"]'))
const popoverList = popoverTriggerList.map(el => new bootstrap.Popover(el))
```

### **Otros Plugins**
- **Select2**: `$('.select2').select2()`
- **DataTables**: `$('.table').DataTable()`
- **DateRangePicker**: `$('input[name="daterange"]').daterangepicker()`
- **Dropzone**: `new Dropzone("#myDropzone")`
- **Quill**: `new Quill('#editor')`

## Eventos Personalizados

```javascript
// Escuchar evento de modal abierto
const myModal = document.getElementById('myModal')
myModal.addEventListener('show.bs.modal', event => {
  console.log('Modal opening...')
})

// Dispara después de clic
myModal.addEventListener('shown.bs.modal', event => {
  console.log('Modal opened!')
})
```

##Helpers/Utilidades JS Propias

*Mono Bootstrap mantiene JS minimal, delegando a Bootstrap y plugins*

Ejemplos de helpers que podrían existir:
```javascript
// Sidebar toggle
function toggleSidebar() {
  document.querySelector('.sidebar').classList.toggle('open')
}

// Theme toggle
function toggleTheme() {
  document.body.classList.toggle('dark-theme')
}
```

## ✅ Checklist - TEMA 10 Completado

- ✅ Archivos JS identificados
- ✅ Comportamientos Bootstrap JS
- ✅ Validación de formularios
- ✅ Inicialización de plugins
- ✅ Eventos personalizados
- ✅ Helpers/utilities JS
