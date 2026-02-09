# 🎨 Análisis: Estilos y Customización de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Paleta de Colores

```
Primary:     #0055B8 (azul)
Secondary:   #6C757D (gris)
Success:     #28A745 (verde)
Danger:      #DC3545 (rojo)
Warning:     #FFC107 (naranja/amarillo)
Info:        #17A2B8 (celeste)
Light:       #F8F9FA (blanco/gris muy claro)
Dark:        #343A40 (casi negro)
```

## Tipografía

**Fuentes**:
- **Karla** (400, 700) - Body text, default
- **Roboto** (400, 700) - Alternative sans-serif
- Servidas desde Google Fonts (CDN)

**Tamaños de Base**:
- Body: 14px
- h1: 28px, h2: 24px, h3: 20px, h4: 16px, h5: 14px, h6: 12px

## Sistema de Espaciado

```
$spacer: 1rem = 16px

Utilities:
.m-0 a .m-5          (margin: 0 a 3rem)
.p-0 a .p-5          (padding: 0 a 3rem)
.mt-2, .mb-3, etc.   (marginal top/bottom/left/right)
.mx-auto, .my-auto   (auto margins)
```

## Grid System

```
12 columnas fluidas
Containers: .container (ancho fijo), .container-fluid (100%)

.row                 # Flex row
.col                 # Ancho automático
.col-12              # Ancho completo
.col-md-6            # 50% en tablets+
.col-lg-4            # 33% en laptops+
```

##  Temas Disponibles

**Light** (default): Background blanco, texto oscuro
**Dark**: Background oscuro, texto claro (por CSS class)

## Archivos SCSS de Customización

```
source/scss/
├── _variables.scss       # Variables de colores, fonts, espaciado
├── _buttons.scss         # Estilos de botones
├── _cards.scss           # Estilos de cards
├── _forms.scss           # Estilos de formularios
├── _sidebar.scss         # Estilos sidebar custom
├── _navbar.scss          # Top navbar styles
├── ... más componentes
└── style.scss            # Main file que compila todo
```

## Cómo Customizar Estilos

### **Opción 1: Modificar variables SCSS**
```scss
// En source/scss/_variables.scss
$primary: #your-color;
$font-family-base: 'Your Font';
// Recompilar: npm run build
```

###**Opción 2: Agregar CSS custom**
```css
/* Append a theme/css/style.css */
.my-custom-class {
  color: #custom;
}
```

### **Opción 3: Override en HTML**
```html
<style>
  .btn-primary { background: #custom; }
</style>
```

## Responsive Design Patterns

```css
/* Mobile first */
.container { width: 100%; }

@media (min-width: 768px) {
  .container { width: 750px; }
}

@media (min-width: 992px) {
  .container { width: 970px; }
}

@media (min-width: 1200px) {
  .container { width: 1140px; }
}
```

## ✅ Checklist - TEMA 5 Completado

- ✅ Paleta de colores
- ✅ Tipografía y fuentes
- ✅ Sistema de espaciado
- ✅ Grid system
- ✅ Temas disponibles
- ✅ Archivos SCSS
- ✅ Customización explicada
