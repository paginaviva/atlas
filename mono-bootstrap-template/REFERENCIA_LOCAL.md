# 📦 Mono Bootstrap Template - Referencia Local

**Versión**: 1.0.0  
**Fuente Original**: https://github.com/themefisher/mono-bootstrap  
**Licencia**: MIT (ver LICENSE)  
**Fecha de Clone**: 2026-02-09

---

## 🎯 Propósito

Este directorio contiene una copia del template Mono Bootstrap para:
- **Referencia permanente** de la estructura original
- **Validación** de la documentación en `/doc_interfaz_fe/`
- **Consulta rápida** de componentes, configuraciones y build process
- **Mantenimiento** de fuentes para mejoras futuras

---

## 📁 Estructura

```
mono-bootstrap-template/
├── source/                 # Archivos fuente (HTML, SCSS, JS)
├── theme/                  # Build output (HTML compilado, CSS compilado)
├── plugins/                # Dependencias externas (DataTables, ApexCharts, etc.)
├── package.json           # Dependencias NPM (Gulp, Sass, etc.)
├── gulpfile.js            # Build pipeline (Gulp 4.0.2)
├── README.md              # Documentación original
└── LICENSE                # Licencia MIT
```

---

## 🔧 Build & Setup

**Instalar dependencias**:
```bash
npm install
```

**Desarrollo (watch)**:
```bash
npm run dev
```

**Build producción**:
```bash
npm run build
```

---

## 📚 Documentación Relacionada

Toda la especificación técnica de Mono Bootstrap está documentada en:

```
/doc_interfaz_fe/
├── 00_indice_maestro_mono_bootstrap.md
├── 01_analisis_mono_bootstrap_estructura.md
├── 02_analisis_mono_bootstrap_dependencias.md
├── 03_analisis_mono_bootstrap_componentes_ui.md
├── 04_analisis_mono_bootstrap_layouts.md
├── 05_analisis_mono_bootstrap_estilos.md
├── 06_analisis_mono_bootstrap_formularios.md
├── 07_analisis_mono_bootstrap_componentes_datos.md
├── 08_analisis_mono_bootstrap_componentes_modales.md
├── 09_analisis_mono_bootstrap_navegacion.md
├── 10_analisis_mono_bootstrap_javascript.md
├── 11_analisis_mono_bootstrap_iconografia.md
└── 12_analisis_mono_bootstrap_utilidades.md
```

**Para consultar componentes específicos**, ver:
- **Botones, Cards, Badges**: `03_componentes_ui.md`
- **Tablas, Listas**: `07_componentes_datos.md`
- **Formularios**: `06_formularios.md`
- **Layouts**: `04_layouts.md`
- **Estilos y Colores**: `05_estilos.md`

---

## 📊 Especificaciones

| Item | Valor |
|------|-------|
| **Base** | Bootstrap 5 |
| **Template Engine** | HTML estático |
| **Build Tool** | Gulp 4.0.2 |
| **Styling** | SCSS → CSS |
| **Componentes** | 40+ |
| **Plugins** | 23 librerías externas |
| **Ejemplos HTML** | 191 páginas |
| **Iconos** | Material Design Icons |

---

## ⚠️ Notas Importantes

- `node_modules/` está en `.gitignore` → necesita `npm install`
- `theme/` es output del build, no editar directamente
- Modificaciones deben hacerse en `source/`
- Para integración en Atlas, usar especificaciones de `/doc_interfaz_fe/`

---

## 🔄 Cómo Usar Esta Referencia

1. **Consultar componentes**: Busca en `theme/html-pages/`
2. **Validar documentación**: Compara con arquivos en `/doc_interfaz_fe/`
3. **Entender build**: Revisa `gulpfile.js`
4. **Ver ejemplos**: Abre archivos HTML en `theme/`

---

**Última actualización**: 2026-02-09  
**Por**: Atlas Documentation System
