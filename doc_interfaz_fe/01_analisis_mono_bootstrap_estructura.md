# 📁 Análisis: Estructura y Organización de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09  
**Versión de MB**: 1.0.0 (repo: https://github.com/themefisher/mono-bootstrap)  
**Última Actualización**: 2026-02-09  
**Estado**: Completado  

---

## 🎯 Resumen Ejecutivo

Mono Bootstrap es un template admin moderno basado en Bootstrap 5. Utiliza **Gulp** como build tool y **SCSS** para estilos. Tiene dos carpetas principales:
- **`source/`**: Archivos fuente (SCSS, plantillas, imágenes originales)
- **`theme/`**: Archivos compilados/finales listos para usar (191 archivos HTML)

**Tipo de proyecto**: Static HTML Site Generator con build pipeline

---

## 📊 Estructura General de Directorios

```
mono-bootstrap/
├── source/                    ← Archivos fuente
│   ├── images/               ← Imágenes originales
│   ├── scss/                 ← Estilos SCSS
│   ├── js/                   ← JavaScript custom
│   ├── plugins/              ← Plugins/librerías externas
│   ├── static/               ← Assets estáticos
│   └── [archivos .html]      ← Plantillas HTML fuente
│
├── theme/                     ← Archivos compilados (output)
│   ├── images/               ← Imágenes optimizadas
│   ├── css/                  ← CSS compilado (minificado)
│   ├── js/                   ← JavaScript compilado
│   ├── plugins/              ← Plugins compilados
│   ├── data/                 ← Datos (JSON, etc.)
│   └── [191 archivos .html]  ← HTML generado
│
├── .git/                      ← Repositorio Git
├── gulpfile.js                ← Configuración Gulp (build)
├── package.json               ← Dependencias npm
├── README.md                  ← Documentación
├── LICENSE                    ← MIT License
├── netlify.toml               ← Configuración Netlify
├── vercel.json                ← Configuración Vercel
└── [otros archivos config]
```

---

## 🔍 Análisis Detallado de Carpetas

### **`/source/` - Archivos Fuente**

**Propósito**: Código fuente que se transforma en output final

#### **`source/images/`**
- Contiene imágenes originales (no optimizadas)
- Formatos: PNG, JPG, SVG
- Categorización: avatars, header images, hero images, iconography

#### **`source/scss/`**
- Archivos SCSS organizados por componente
- Estructura modular: variables, mixins, componentes, layouts
- Se compilan a CSS único en `theme/css/`

#### **`source/js/`**
- JavaScript custom (no frameworks)
- Scripts para interactividad: sidebar collapse, modals, datepickers, etc.
- Se minifica y copia a `theme/js/`

#### **`source/plugins/`**
- Librerías JavaScript/CSS externas (Chart.js, Bootstrap, Select2, etc.)
- Se copian sin modificación a `theme/plugins/`

#### **`source/static/`**
- Archivos estáticos varios (fonts, documentación, datos)
- Se copian tal cual a `theme/`

#### **Archivos `.html` en `/source/`**
- Plantillas HTML individuales (approx. 191)
- Organizadas por sección: pages, components, layouts
- Incluyen includes mediante `file-include` (gulp-file-include)

---

### **`/theme/` - Directorio de Salida Compilado**

**Propósito**: Archivos finales, listos para producción

#### **`theme/css/`**
- `style.css` (compilado de SCSS)
- `bootstrap.css` (Bootstrap 5)
- Otros CSS compilados
- Todos minificados en producción

#### **`theme/js/`**
- `script.js` (JavaScript custom compilado)
- `bootstrap.bundle.js` (Bootstrap JS)
- Otros JS compilados y minificados

#### **`theme/plugins/`**
- Copias de plugins/librerías desde source
- Chart.js, Select2, Datatables, etc.

#### **`theme/images/`**
- Imágenes optimizadas (generadas por Gulp)
- Carpetas: avatars, header, hero, icons, etc.

#### **`theme/data/`**
- Datos JSON para ejemplos
- Configuraciones de sample data para componentes

#### **`theme/[archivos .html]`** (191 archivos)
- Páginas HTML compiladas desde plantillas source
- Cada página es un archivo separado
- Todas funcionan de forma standalone

---

## 🏗️ Convenciones de Nombrado y Organización

### **Carpetas**
- **Minúsculas**: `source`, `theme`, `images`, `scss`, `js`, `plugins`
- **Semánticas**: Los nombres reflejan claramente el propósito

### **Archivos HTML**
- **kebab-case**: `profile-page.html`, `form-elements.html`, `modal-examples.html`
- **Descriptivos**: Nombre de página + categoría
- **Ejemplos**:
  - `dashboard.html` (página principal)
  - `components-button.html` (componentes de botones)
  - `form-elements.html` (elementos de formularios)
  - `table-example.html` (ejemplos de tablas)

### **Archivos CSS/JS**
- **kebab-case**: `style.css`, `script.js`, `bootstrap.bundle.js`
- **Descriptivos**: Función clara

### **Archivos SCSS**
- **subdirectories por componente**: `_buttons.scss`, `_cards.scss`, `_forms.scss`
- **Prefijo underscore**: Indica partials SCSS (no compilados independientemente)

---

## 📄 Punto de Entrada

### **Archivo Principal**
```
theme/index.html
```

**Contenido**: Página de inicio del dashboard
- Incluye sidebar con navegación
- Incluye header con controles
- Link a todas las secciones

### **Estructura HTML Estándar**
Todas las páginas HTML siguen patrón similar:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mono - Admin Dashboard</title>
    
    <!-- CSS Bootstrap -->
    <link rel="stylesheet" href="css/bootstrap.css">
    <!-- CSS Custom -->
    <link rel="stylesheet" href="css/style.css">
    <!-- Plugins -->
    <link rel="stylesheet" href="plugins/[plugin].css">
</head>
<body>
    <!-- Sidebar -->
    <!-- Main Content -->
    <!-- Footer -->
    
    <!-- Scripts Bootstrap -->
    <script src="js/bootstrap.bundle.js"></script>
    <!-- Scripts Custom -->
    <script src="js/script.js"></script>
    <!-- Plugins -->
    <script src="plugins/[plugin].js"></script>
</body>
</html>
```

---

## 🛠️ Build Process y Tooling

### **Build Tool**
- **Gulp 4.0.2**: Task runner para automatizar procesos

### **Scripts NPM Disponibles**
```bash
npm run dev      # Ejecuta Gulp en modo watch (desarrollo)
npm run build    # Compila para producción
npm run download # Descarga recursos
npm run deploy   # Deploy (configuración externa)
```

### **Tareas Gulp Principales**
- **Compilación SCSS** → CSS
- **Minificación CSS/JS**
- **Optimización de imágenes**
- **Inclusión de archivos** (plantillas)
- **BrowserSync** para live reload en desarrollo

---

## 📊 Estadísticas de Archivos

| Tipo | Cantidad |Ubicación|
|------|----------|---------|
| Archivos HTML | 191 | `theme/` |
| Archivos SCSS | ~30+ | `source/scss/` |
| Archivos JS | ~5-10 | `source/js/` + `source/plugins/` |
| Imágenes | ~40+ | `theme/images/` |
| Plugins/Librerías | ~15+ | `theme/plugins/` |
| CSS Compilados | 3-4 | `theme/css/` |
| CSS Plugins | ~10+ | `theme/plugins/` |

---

## 🗂️ Organización de Páginas HTML (Ejemplos)

### **Categorías de Páginas**

**Dashboard/Home**
- `index.html` - Dashboard principal

**Componentes**
- `components-[tipo].html` - Ej: `components-button.html`, `components-card.html`

**Formularios**
- `form-elements.html` - Elementos de form
- `form-layouts.html` - Layouts de forms

**Tablas**
- `table-*.html` - Tablas con diferentes estilos/plugins

**Ejemplos**
- `[feature]-example.html` - Ejemplos de features específicas

---

## 🔗 Sistema de Referencias

### **Includes en Plantillas**
El proyecto usa **gulp-file-include** para reutilizar componentes:

```html
<!-- Incluye header compartido -->
@@include('_header.html')

<!-- Incluye sidebar compartido -->
@@include('_sidebar.html')

<!-- Incluye footer compartido -->
@@include('_footer.html')
```

**Ventaja**: No repetir HTML; cambios centralizados

---

## 📝 Archivos de Configuración

| Archivo | Propósito |
|---------|----------|
| `gulpfile.js` | Definición de tareas de build |
| `package.json` | Dependencias npm |
| `netlify.toml` | Deploy a Netlify |
| `vercel.json` | Deploy a Vercel |
| `.editorconfig` | Formato de código |
| `.jshintrc` | Configuración JSHint |
| `.gitignore` | Archivos a ignorar en Git |

---

## 🎓 Conclusiones de Estructura

1. **Arquitectura Clara**: Separación clara entre source y output
2. **Build Pipeline**: Automatización con Gulp para desarrollo/producción
3. **Modular**: Archivos organizados por función
4. **Escalable**: Sistema de includes permite reutilización
5. **Bien Documentado**: Nombres descriptivos, estructura intuitiva
6. **191 páginas HTML**: Exhaustivo conjunto de ejemplos y componentes

---

## ✅ Checklist - TEMA 1 Completado

- ✅ Árbol de directorios completo
- ✅ Propósito de cada carpeta principal
- ✅ Punto de entrada identificado (index.html)
- ✅ Convenciones de nombrado documentadas
- ✅ Distinción entre source y output
- ✅ Build process entendido

---

**Próximo**: TEMA 2 - Dependencias Externas
