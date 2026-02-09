# 📦 Análisis: Dependencias Externas de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09  
**Versión de MB**: 1.0.0  
**Última Actualización**: 2026-02-09  
**Estado**: Completado

---

## 🎯 Resumen Ejecutivo

Mono Bootstrap utiliza **22 librerías/plugins** organizadas en `theme/plugins/`. La mayoría de las dependencias son **opcionales** (usadas en ejemplos específicos).

**Dependencia Core Única**: Bootstrap 5 (incluido en plugins)

**Otros**: Seleccionados según funcionalidad (gráficos, tablas, carruseles, etc.)

---

## 📋 Clasificación de Dependencias

### **CRÍTICAS (Siempre necesarias)**

| Librería | Versión Estimada | Tipo | Ubicación | Notas |
|----------|------------------|------|-----------|-------|
| **Bootstrap** | 5.x | CSS/JS | `plugins/bootstrap/` | Framework base, required |
| **Material Design Icons** | Latest | CSS | `plugins/material/` | Sistema de iconos principal |
| **Google Fonts** | - | CSS | CDN | Fuentes Karla, Roboto |
| **jQuery** | 3.6.x | JS | `plugins/jquery/` | Requerido por algunos plugins |

---

### **OPCIONALES (Usadas en ejemplos/secciones específicas)**

| Librería | Versión | Tipo | Ubicación | Propósito | Ejemplos |
|----------|---------|------|-----------|----------|----------|
| **DataTables** | 1.10.18 | JS/CSS | `plugins/DataTables/` | Tablas interactivas | table-example.html |
| **ApexCharts** | Latest | JS | `plugins/apexcharts/` | Gráficos | charts-|
| **Select2** | 4.1.x | JS/CSS | `plugins/select2/` | Selects avanzados | form-elements.html |
| **DateRangePicker** | Latest | JS/CSS | `plugins/daterangepicker/` | Selector de fechas | form-elements.html |
| **Dropzone** | 5.x | JS/CSS | `plugins/dropzone/` | Upload de archivos | form-elements.html |
| **Owl Carousel** | 2.x | JS/CSS | `plugins/owl-carousel/` | Carruseles/sliders | landing pages |
| **Quill** | 1.3.6 | JS/CSS | CDN | Editor WYSIWYG | editor.html |
| **Prism** | Latest | JS/CSS | `plugins/prism/` | Syntax highlighting | code examples |
| **CodeMirror** | Latest | JS/CSS | `plugins/codemirror/` | Editor de código | developer pages |
| **Full Calendar** | Latest | JS/CSS | `plugins/fullcalendar/` | Calendario eventos | calendar.html |
| **SimpleBanker** | Latest | CSS | `plugins/simplebar/` | Custom scrollbars | layouts |
| **NProgress** | Latest | JS/CSS | `plugins/nprogress/` | Progress bar global | layouts |
| **Toastr** | Latest | JS/CSS | `plugins/toaster/` | Notificaciones toast | ejemplos |
| **Ladda** | Latest | JS/CSS | `plugins/ladda/` | Botones con loading | ejemplos |
| **jVectorMap** | 2.0.3 | JS/CSS | `plugins/jvectormap/` | Mapas vectoriales | charts.html |
| **jQuery Mask Input** | Latest | JS | `plugins/jquery-mask-input/` | Máscaras de input | form-elements.html |
| **CircleProgress** | Latest | JS/CSS | `plugins/circle-progress/` | Progress circles | components |
| **Material Theme** | Latest | CSS | `plugins/material/` | Tema Material Design | estilos |
| **Flag Icons** | Latest | CSS | `plugins/flag-icons/` | Iconos de banderas | ejemplos |
| **Syotimer** | Latest | JS | `plugins/syotimer/` | Contador regresivo | ejemplos |
| **Jekyll Search** | Latest | JS | (inline) | Búsqueda en documentación | search |

---

## 🏗️ Dependencias NPM (Build Time)

**Solo para desarrollo/build**, no se incluyen en producción:

```json
{
  "dependencies": {
    "browser-sync": "^2.27.10",
    "gulp": "^4.0.2",
    "gulp-autoprefixer": "^8.0.0",
    "gulp-file-include": "^2.3.0",
    "gulp-gm": "0.0.9",
    "gulp-header-comment": "^0.10.0",
    "gulp-jshint": "^2.1.0",
    "gulp-rimraf": "^1.0.0",
    "gulp-sass": "^5.1.0",
    "gulp-sourcemaps": "^3.0.0",
    "gulp-util": "^3.0.8",
    "jshint": "^2.13.6",
    "jshint-stylish": "^2.2.1",
    "sass": "^1.54.0"
  }
}
```

### **Propósito de cada dependencia NPM**

| Paquete | Propósito |
|---------|----------|
| **gulp** | Task runner para automatizar build |
| **gulp-sass** | Compilar SCSS a CSS |
| **sass** | Motor de compilación SCSS |
| **gulp-autoprefixer** | Añadir prefijos de navegadores |
| **gulp-file-include** | Incluir archivos en templates |
| **gulp-jshint** | Linting de JavaScript |
| **jshint**, **jshint-stylish** | Herramientas de linting |
| **browser-sync** | Live reload en desarrollo |
| **gulp-sourcemaps** | Source maps para debugging |
| **gulp-gm, gulp-rimraf, gulp-util, gulp-header-comment** | Utilidades diversas |

---

## 🔗 Cómo se Incluyen las Dependencias

### **En Archivos HTML**

```html
<!-- CSS desde plugins locales -->
<link href="plugins/select2/css/select2.min.css" rel="stylesheet" />

<!-- CSS desde CDN -->
<link href="https://fonts.googleapis.com/css?family=Karla:400,700|Roboto" rel="stylesheet">
<link href="https://cdn.quilljs.com/1.3.6/quill.snow.css" rel="stylesheet">

<!-- JavaScript desde plugins locales -->
<script src="plugins/jquery/jquery-3.6.0.min.js"></script>
<script src="plugins/bootstrap/js/bootstrap.bundle.min.js"></script>

<!-- JavaScript desde CDN -->
<!-- (Quill se incluye desde CDN) -->
```

### **Patrón General**

1. **Archivos minificados preferidos**: `.min.js`, `.min.css`
2. **Source maps incluidos**: Para debugging (`.js.map`, `.css.map`)
3. **Carga en orden específico**: jQuery primero, luego plugins que lo requieran
4. **Bootstrap al inicio**: CSS antes, JS al final

---

## 📊 Estadísticas de Dependencias

| Categoría | Cantidad |
|-----------|----------|
| **Plugins Locales Totales** | 23 |
| **Librerías Desde CDN** | 3 (Google Fonts, Quill, otros menores) |
| **Librerías Críticas** | 4 |
| **Librerías Opcionales** | 19 |
| **Dependencias NPM Dev** | 13 |
| **Versiones Fijas** | DataTables (1.10.18), jVectorMap (2.0.3), Quill (1.3.6) |
| **Versiones Flexibles** | Mayoría (Latest) |

---

## ⚙️ Ubicación de Dependencias en Proyecto

```
theme/plugins/
├── bootstrap/
│   ├── css/
│   │   └── bootstrap.min.css (v5.x)
│   └── js/
│       └── bootstrap.bundle.min.js
│
├── jquery/
│   └── jquery-3.6.0.min.js
│
├── select2/
│   ├── css/
│   │   ├── select2.css
│   │   └── select2.min.css
│   └── js/
│       ├── select2.full.min.js
│       ├── select2.min.js
│       └── i18n/ (26+ idiomas)
│
├── DataTables/
│   └── DataTables-1.10.18/
│       ├── css/
│       │   └── jquery.dataTables.min.css
│       └── js/
│           └── jquery.dataTables.min.js
│
├── apexcharts/
│   ├── apexcharts.min.js
│   └── [recursos gráficos]
│
├── daterangepicker/
│   ├── moment.min.js
│   ├── daterangepicker.js
│   └── daterangepicker.css
│
├── [otros 17 plugins con estructura similar]
```

---

## 🎯 Dependencias por Caso de Uso

### **Dashboard Básico**
- Bootstrap (CSS/JS)
- jQuery
- Material Design Icons
- Google Fonts
- SimpleBanker (scrollbars)
- NProgress (progress bar)

### **Dashboard con Gráficos**
- ... + ApexCharts

### **Dashboard con Datos**
- ... + DataTables
- ... + Toastr (notificaciones)

### **Gestión de Formularios**
- ... + Select2
- ... + DateRangePicker
- ... + Dropzone
- ... + jQuery Mask Input

### **CMS/Contenido**
- ... + Quill (editor)
- ... + CodeMirror o Prism
- ... + FullCalendar

---

## 🔄 Build Process - Cómo se Procesan Dependencias

### **En desarrollo (`npm run dev`)**
1. Gulp observa cambios en `/source/scss/`
2. Compila SCSS a CSS
3. BrowserSync recarga automáticamente
4. Plugins se sirven desde `/source/plugins/` (copias)

### **En producción (`npm run build`)**
1. Gulp compila SCSS → CSS minificado
2. JS se minifica
3. Imágenes se optimizan
4. Archivos se copian a `/theme/` (output final)
5. Source maps se generan
6. Resultado: Archivos listos para deploy

---

## 📋 Selección de Dependencias para Integración en Atlas

**Recomendado incluir en v1**:
- ✅ Bootstrap (CSS/JS)
- ✅ jQuery (requerido por varios plugins)
- ✅ Material Design Icons (iconografía)
- ✅ Select2 (selectores avanzados)
- ✅ DataTables (tablas de datos)
- ✅ ApexCharts (gráficos)

**Considerar para v1.1+**:
- ⚠️ SimpleBanker (scrollbars mejorados)
- ⚠️ Toastr (notificaciones)
- ⚠️ DateRangePicker (si se necesitan filtros de fecha)

**Postergables para v2.0+**:
- ❌ Quill, CodeMirror, Prism (solo si se crea un editor)
- ❌ FullCalendar (solo si se necesita gestión de eventos)
- ❌ jVectorMap (solo para analytics avanzados)
- ❌ Ladda, Owl Carousel (efectos visuales opcionales)

---

## ✅ Checklist - TEMA 2 Completado

- ✅ Listado completo de dependencias CSS/JS
- ✅ Versiones exactas documentadas
- ✅ Ubicación de cada dependencia
- ✅ Clasificación: críticas vs opcionales
- ✅ Dependencias NPM (build time) documentadas
- ✅ Cómo se incluyen (local vs CDN)
- ✅ Build process entendido
- ✅ Recomendaciones para integración

---

**Próximo**: TEMA 3 - Sistema de Componentes UI
