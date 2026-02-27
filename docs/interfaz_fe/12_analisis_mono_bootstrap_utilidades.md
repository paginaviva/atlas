# 🔧 Análisis: Utilidades y Helpers de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Clases Utilidad CSS Comunes

### **Spacing (Margin/Padding)**

```html
<!-- Margin -->
.m-0, .m-1, .m-2, .m-3, .m-4, .m-5         /* 0-3rem */
.mt-2, .mb-3, .ml-1, .mr-2                  /* Top, Bottom, Left, Right */
.mx-auto, .my-2                             /* X (horizontal), Y (vertical) */
.m-md-2                                     /* Responsive */

<!-- Padding -->
.p-0 a .p-5
.pt-3, .pb-2, .pl-1, .pr-3
.px-2, .py-3
```

### **Display**

```html
.d-none                   /* display: none */
.d-inline                 /* display: inline */
.d-block                  /* display: block */
.d-flex                   /* display: flex */
.d-grid                   /* display: grid */
.d-md-block               /* Responsive: block en md+ */
.d-lg-none                /* Responsive: hidden en lg+ */
```

### **Flexbox**

```html
.flex-row                 /* direction: row */
.flex-column              /* direction: column */
.justify-content-start    /* flex-start */
.justify-content-center   /* center */
.justify-content-between  /* space-between */
.align-items-center       /* vertical center */
.gap-2                    /* gap/spacing entre items */
```

### **Grid/Columnas**

```html
.container                /* Ancho fijo (max-width) */
.container-fluid          /* 100% ancho */
.row                      /* flex row */
.col                      /* ancho auto */
.col-12                   /* 100% */
.col-md-6                 /* 50% en tablets+ */
.col-lg-4                 /* 33% en laptops+ */
.col-md-6 col-lg-4        /* Responsive combinado */
```

### **Tipografía**

```html
.text-start               /* text-align: start */
.text-center              /* center */
.text-end                 /* end */
.text-muted               /* color: #6C757D */
.text-primary             /* color: primary */
.fw-bold                  /* font-weight: bold */
.fw-normal                /* font-weight: normal */
.fs-1 a .fs-6             /* font sizes h1-h6 */
.text-uppercase           /* uppercase */
.text-lowercase           /* lowercase */
```

### **Colores**

```html
.bg-primary               /* background color */
.bg-success               /* background: green */
.text-danger              /* text color red */
.border-primary           /* border color */
.border-top               /* border-top */
.rounded                  /* border-radius */
.rounded-circle           /* 50% border-radius */
```

### **Visibilidad**

```html
.visible                  /* visible */
.invisible                /* visibility: hidden */
.bg-transparent           /* background: transparent */
.opacity-25               /* opacity: 0.25 */
.opacity-50               /* opacity: 0.50 */
.opacity-75               /* opacity: 0.75 */
.opacity-100              /* opacity: 1 */
```

## Breakpoints de MediaQuery

```scss
xs  (default)  /* No media query needed */
sm  576px      @media (min-width: 576px)
md  768px      @media (min-width: 768px)
lg  992px      @media (min-width: 992px)
xl  1200px     @media (min-width: 1200px)
xxl 1400px     @media (min-width: 1400px)
```

## Patrones Responsive Comunes

```html
<!-- Hidden en mobile, visible en md+ -->
<div class="d-none d-md-block">
  Content for desktop
</div>

<!-- Responsive column -->
<div class="row">
  <div class="col-12 col-md-6 col-lg-4">
    Column content
  </div>
</div>

<!-- Responsive flex -->
<div class="d-flex flex-column flex-md-row gap-2">
  <div>Item 1</div>
  <div>Item 2</div>
</div>
```

## Grid System Ejemplo

```html
<!-- 12 columnas -->
<div class="container">
  <div class="row">
    <div class="col-lg-3">Sidebar (25%)</div>
    <div class="col-lg-9">Content (75%)</div>
  </div>
  
  <!-- 3 columnas equal -->
  <div class="row">
    <div class="col-md-4">Col 1</div>
    <div class="col-md-4">Col 2</div>
    <div class="col-md-4">Col 3</div>
  </div>
</div>
```

## Documentación Incluida

**README.md**: Instrucciones generales
**Live Demo**: https://demo.themefisher.com/mono/
**Docs**: https://docs.themefisher.com/mono/ (externa)

## ✅ Checklist - TEMA 12 Completado

- ✅ Utilidades CSS principales
- ✅ Breakpoints y responsive design
- ✅ Grid system
- ✅ Patrones responsive
- ✅ Spacing, display, flexbox
- ✅ Tipografía y colores utilities
- ✅ Documentación
