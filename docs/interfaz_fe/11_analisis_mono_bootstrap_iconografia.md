# 🎯 Análisis: Iconografía de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Sistema de Iconos

**Librería Principal**: Material Design Icons (MDI)

**Versión**: Latest (incluido en `plugins/material/`)

**Ubicación**: `/plugins/material/css/materialdesignicons.min.css`

---

## Cómo Incluir en HTML

```html
<!-- CSS en head -->
<link href="plugins/material/css/materialdesignicons.min.css" rel="stylesheet" />

<!-- Uso en HTML -->
<i class="material-icons">dashboard</i>
<i class="material-icons">person</i>
<i class="material-icons">settings</i>
```

---

## Sintaxis

```html
<!-- Básico -->
<i class="material-icons">icon-name</i>

<!-- Con tamaño -->
<i class="material-icons">icon-name</i> <!-- 24px default -->

<!-- En botón -->
<button class="btn btn-primary">
  <i class="material-icons">add</i> Add Item
</button>

<!-- En sidebar -->
<a href="#">
  <i class="material-icons">home</i>
  <span>Home</span>
</a>
```

## Iconos Más Utilizados en Mono BS

| Icono | Nombre | Ubicación | Uso |
|-------|--------|-----------|-----|
| 📊 dashboard | `dashboard` | Sidebar | Inicio |
| 👤 person | `person` | Header user | Perfil |
| ⚙️ settings | `settings` | Sidebar | Configuración |
| 📝 edit | `edit` | Tablas | Editar |
| 🗑️ delete | `delete` | Tablas | Eliminar |
| ➕ add | `add` | Botones | Crear |
| ✓ check | `check` | Modales | Confirmar |
| ✗ close | `close` | Modales | Cerrar |
| 📤 upload | `upload` | Dropzone | Subir |
| 📥 download | `download` | Botones | Descargar |
| 🔍 search | `search` | Navbar | Buscar |
| 📧 mail | `mail` | Iconos generales | Email |
| 📞 phone | `phone` | Iconos generales | Teléfono |
| 🏠 home | `home` | Breadcrumbs | Inicio |

## Personalización de Iconos

### **Tamaño**
```css
/* Default: 24px */
.material-icons {
  font-size: 18px;      /* Pequeño */
  font-size: 24px;      /* Default */
  font-size: 32px;      /* Grande */
  font-size: 48px;      /* Muy grande */
}
```

### **Color**
```html
<!-- Con clases Bootstrap -->
<i class="material-icons text-primary"></i>
<i class="material-icons text-success"></i>
<i class="material-icons text-danger"></i>

<!-- O CSS inline -->
<i class="material-icons" style="color: red;">delete</i>
```

### **Rotación/Animación**
```css
.material-icons.spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
```

## Búsqueda de Iconos

Referencia online: https://fonts.google.com/icons

Todos los iconos disponibles en la versión incluida están en esta referencia.

## ✅ Checklist - TEMA 11 Completado

- ✅ Sistema de iconos identificado (Material Design Icons)
- ✅ Cómo incluir en HTML
- ✅ Sintaxis de uso
- ✅ Iconos principales documentados
- ✅ Personalización (tamaño, color, animación)
- ✅ Referencia de búsqueda
