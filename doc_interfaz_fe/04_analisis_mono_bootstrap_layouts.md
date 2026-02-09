# 🏗️ Análisis: Layouts y Templates de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Layout Base Estándar

```html
<!DOCTYPE html>
<html>
<head>
    <!-- Meta, CSS -->
</head>
<body class="antialiased border-top border-primary">
    
    <!-- NAVBAR (top) -->
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <!-- Logo, menu items, user dropdown -->
    </nav>
    
    <!-- MAIN CONTAINER -->
    <div class="page-wrapper">
        
        <!-- SIDEBAR (lateral) -->
        <nav class="sidebar sidebar-offcanvas" id="sidebar">
            <!-- Navigation items, collapsible menus -->
        </nav>
        
        <!-- MAIN CONTENT AREA -->
        <div class="main-panel">
            
            <!-- Breadcrumb -->
            <ol class="breadcrumb">
                <li class="breadcrumb-item"><a href="index.html">Dashboard</a></li>
                <li class="breadcrumb-item active">Current page</li>
            </ol>
            
            <!-- PAGE CONTENT -->
            <div class="content-wrapper">
                <div class="row">
                    <!-- Column content -->
                </div>
            </div>
            
        </div>
    </div>
    
    <!-- FOOTER -->
    <footer class="footer">
        <!-- Copyright, links -->
    </footer>
    
    <!-- Scripts -->
</body>
</html>
```

---

## Variantes de Layout

### **1. Full-Width Dashboard (index.html)**
- Navbar en top
- Sidebar colapsible a un lado
- Content area responsive
- Para dashboards principales

### **2. Sin Sidebar (single-page.html)**
- Solo navbar top
- Full width content
- Para landing pages, login, etc.

### **3. Sidebar Expandido (default)**
- Sidebar por defecto abierto
- Colapsible a icono
- Responsive: se oculta en mobile

### **4. Dark Theme Variante**
- Background oscuro
- Texto claro
- Navbar/sidebar dark
- Mismo HTML, diferentes clases

---

## Componentes Estructurales

| Componente | Ubicación | Clases | Función |
|-----------|-----------|--------|----------|
| **Navbar** | Top | `.navbar`, `.navbar-expand-lg` | Navegación principal, user menu |
| **Sidebar** | Left | `.sidebar`, `.sidebar-offcanvas` | Menú lateral, collapsible |
| **Main Panel** | Centro | `.main-panel` | Área principal de contenido |
| **Content Wrapper** | Main Panel | `.content-wrapper` | Contenedor de página |
| **Footer** | Bottom | `.footer` | Pie de página |
| **Breadcrumb** | Dentro Main | `.breadcrumb` | Ruta de ubicación |

---

## Responsividad

### **Breakpoints Bootstrap**
```
xs (default)  < 576px    (teléfono)
sm            ≥ 576px    (tablet pequeña)
md            ≥ 768px    (tablet)
lg            ≥ 992px    (laptop)
xl            ≥ 1200px   (desktop grande)
xxl           ≥ 1400px   (ultra wide)
```

### **Comportamiento Responsive**
- **Mobile**: Sidebar oculto (icono hamburgesa), navbar colapsible
- **Tablet**: Sidebar visible, layout reajustado
- **Desktop**: Full sidebar + content + footer

### **Clases Utilities Comunes**
```
.d-none                    # Ocultar siempre
.d-md-none                 # Ocultar en md+
.d-lg-block                # Visible en lg+
.col-12 col-md-6           # Responsive columns
.flex-column flex-md-row    # Responsive flex
```

---

## ✅ Checklist - TEMA 4 Completado

- ✅ Layout base documentado
- ✅ Variantes de layout
- ✅ Componentes estructurales
- ✅ Responsividad explicada
- ✅ Breakpoints definidos

