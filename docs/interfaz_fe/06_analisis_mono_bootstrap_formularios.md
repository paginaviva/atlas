# 📝 Análisis: Componentes Formularios de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Tipos de Input

| Tipo | HTML | Clases | Estados | Ejemplo |
|------|------|--------|---------|---------|
| **Text** | `<input type="text">` | `.form-control` | Normal, focused, invalid | basic-input.html |
| **Email** | `<input type="email">` | `.form-control` | Validación email | form-validation.html |
| **Password** | `<input type="password">` | `.form-control` | Oculto | basic-input.html |
| **Number** | `<input type="number">` | `.form-control` | Min/max | basic-input.html |
| **Checkbox** | `<input type="checkbox">` | `.form-check-input` | Checked, disabled | checkbox-radio.html |
| **Radio** | `<input type="radio">` | `.form-check-input` | Selected | checkbox-radio.html |
| **Select** | `<select>` | `.form-select`, `.form-control` | Options, grouping | form-advance.html |
| **Textarea** | `<textarea>` | `.form-control` | Rows, resize | basic-input.html |
| **Date** | `<input type="date">` | `.form-control` | Date picker (plugin) | form-advance.html |
| **File** | `<input type="file">` | `.form-control` | Multiple | form-advance.html |

## Estructura Estándar

```html
<!-- Label + Input (Horizontal) -->
<div class="form-group">
  <label for="exampleInput" class="form-label">Input Label</label>
  <input type="text" class="form-control" id="exampleInput" placeholder="Enter text">
</div>

<!-- Checkbox -->
<div class="form-check">
  <input class="form-check-input" type="checkbox" id="exampleCheck">
  <label class="form-check-label" for="exampleCheck">Check me out</label>
</div>

<!-- Select -->
<div class="form-group">
  <label for="exampleSelect">Select Option</label>
  <select class="form-select" id="exampleSelect">
    <option>Option 1</option>
    <option>Option 2</option>
  </select>
</div>
```

## Estados de Validación

```html
<!-- Valid State -->
<input class="form-control is-valid" type="email">
<div class="valid-feedback">Looks good!</div>

<!-- Invalid State -->
<input class="form-control is-invalid" type="email">
<div class="invalid-feedback">Please enter valid email</div>

<!-- Disabled -->
<input class="form-control" type="text" disabled>
<input class="form-control" type="text" readonly>
```

## Layouts de Formularios

```html
<!-- Horizontal Layout -->
<div class="row">
  <div class="col-md-6">
    <div class="form-group">
      <label>First Name</label>
      <input class="form-control">
    </div>
  </div>
  <div class="col-md-6">
    <div class="form-group">
      <label>Last Name</label>
      <input class="form-control">
    </div>
  </div>
</div>

<!-- Vertical Layout -->
<form>
  <div class="form-group mb-3">
    <label>Email</label>
    <input class="form-control">
  </div>
  <div class="form-group mb-3">
    <label>Password</label>
    <input class="form-control" type="password">
  </div>
  <button class="btn btn-primary">Submit</button>
</form>
```

## Validación JavaScript

```javascript
// Form validation
const form = document.querySelector('form');
form.addEventListener('submit', function(e) {
  if (!form.checkValidity()) {
    e.preventDefault();
    e.stopPropagation();
  }
  form.classList.add('was-validated');
});
```

## Plugins de Formularios

- **Select2**: Selectpickers mejorados con búsqueda
- **DateRangePicker**: Selectores de fecha/rango
- **Dropzone**: Upload de archivos
- **jQuery Mask**: Máscaras de input (teléfono, etc.)

## ✅ Checklist - TEMA 6 Completado

- ✅ Tipos de input documentados
- ✅ Estructura HTML
- ✅ Estados de validación
- ✅ Layouts de formularios
- ✅ Validación JS
- ✅ Plugins disponibles
