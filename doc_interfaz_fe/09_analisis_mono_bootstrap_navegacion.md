# 🧭 Análisis: Componentes de Navegación de Mono Bootstrap

**Fecha de Análisis**: 2026-02-09 | **Estado**: Completado

---

## Navbar (Top Navigation)

```html
<nav class="navbar navbar-expand-lg navbar-light bg-light">
  <div class="container-fluid">
    <a class="navbar-brand" href="index.html">Mono</a>
    
    <button class="navbar-toggler" data-bs-toggle="collapse" 
            data-bs-target="#navbarNav">
      <span class="navbar-toggler-icon"></span>
    </button>
    
    <div class="collapse navbar-collapse" id="navbarNav">
      <ul class="navbar-nav ms-auto">
        <li class="nav-item">
          <a class="nav-link active" href="#">Home</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="#">Features</a>
        </li>
      </ul>
    </div>
  </div>
</nav>
```

**Variantes**:
- `navbar-expand-lg` - Responsive
- `navbar-dark bg-dark` - Tema oscuro
- `navbar-sticky-top` - Fijo en top
- `navbar-light / navbar-dark`

## Sidebar (Lateral Navigation)

```html
<nav class="sidebar sidebar-offcanvas" id="sidebar">
  <div class="sidebar-brand">
    <h2><span class="logo-icon">M</span></h2>
  </div>
  
  <div class="sidebar-body">
    <ul class="nav">
      <li class="nav-item">
        <a class="nav-link" href="index.html">
          <i class="material-icons">dashboard</i>
          <span>Dashboard</span>
        </a>
      </li>
      
      <!-- Sub menu (collapsed) -->
      <li class="nav-item">
        <a class="nav-link" data-bs-toggle="collapse" 
           href="#components">
          <i class="material-icons">layers</i>
          <span>Components</span>
          <i class="menu-arrow"></i>
        </a>
        <div class="collapse" id="components">
          <ul class="nav flex-column sub-menu">
            <li class="nav-item">
              <a href="components-button.html">Buttons</a>
            </li>
            <li class="nav-item">
              <a href="components-card.html">Cards</a>
            </li>
          </ul>
        </div>
      </li>
    </ul>
  </div>
</nav>
```

**Características**:
- Colapsible / expandible
- Submenús anidados
- Iconos (Material Design Icons)
- Responsive offcanvas

## Breadcrumb

```html
<ol class="breadcrumb">
  <li class="breadcrumb-item"><a href="index.html">Home</a></li>
  <li class="breadcrumb-item"><a href="#">Library</a></li>
  <li class="breadcrumb-item active">Data</li>
</ol>
```

## Pagination

```html
<nav aria-label="...">
  <ul class="pagination">
    <li class="page-item disabled">
      <a class="page-link" href="#" tabindex="-1">Previous</a>
    </li>
    <li class="page-item"><a class="page-link" href="#">1</a></li>
    <li class="page-item active"><a class="page-link" href="#">2</a></li>
    <li class="page-item"><a class="page-link" href="#">3</a></li>
    <li class="page-item">
      <a class="page-link" href="#">Next</a>
    </li>
  </ul>
</nav>
```

**Tamaños**: `.pagination-lg`, `.pagination-sm`

## Tabs

```html
<ul class="nav nav-tabs">
  <li class="nav-item">
    <a class="nav-link active" data-bs-toggle="tab" href="#tab1">
      Tab 1
    </a>
  </li>
  <li class="nav-item">
    <a class="nav-link" data-bs-toggle="tab" href="#tab2">
      Tab 2
    </a>
  </li>
</ul>

<div class="tab-content">
  <div class="tab-pane fade show active" id="tab1">
    Content of tab 1
  </div>
  <div class="tab-pane fade" id="tab2">
    Content of tab 2
  </div>
</div>
```

## Pills (Similar a Tabs)

```html
<ul class="nav nav-pills">
  <li class="nav-item">
    <a class="nav-link active" data-bs-toggle="pill" href="#pill1">
      Pill 1
    </a>
  </li>
  <li class="nav-item">
    <a class="nav-link" data-bs-toggle="pill" href="#pill2">
      Pill 2
    </a>
  </li>
</ul>
```

## Stepper/Wizard

*No incluido en Mono BS, pero se puede implementar con CSS personalizado*

```html
<div class="stepper">
  <div class="step active">
    <h4>Step 1</h4>
  </div>
  <div class="step">
    <h4>Step 2</h4>
  </div>
</div>
```

## ✅ Checklist - TEMA 9 Completado

- ✅ Navbar estructura
- ✅ Sidebar/lateral nav
- ✅ Breadcrumbs
- ✅ Pagination
- ✅ Tabs/pills
- ✅ Navegación responsive
