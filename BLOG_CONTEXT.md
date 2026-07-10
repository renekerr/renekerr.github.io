# BLOG_CONTEXT.md — Low Priv Notes

Documento de contexto para Claude Code. Describe el estado actual del blog, su estructura, convenciones y flujo de trabajo.

---

## Descripción del proyecto

**Blog:** Low Priv Notes  
**URL:** https://renekerr.github.io  
**Repositorio GitHub:** https://github.com/renekerr/renekerr.github.io  
**Tecnología:** Jekyll + tema Lanyon  
**Idioma:** Inglés  

### Propósito

Blog técnico de ciberseguridad con foco en vectores alternativos de explotación y escalada de privilegios. El enfoque principal es documentar rutas de ataque no convencionales — las que la mayoría de writeups ignoran porque ya tienen root por la vía fácil.

No se incluyen flags ni contraseñas en ningún post.

---

## Estructura del repositorio

```
renekerr.github.io/
├── _config.yml          # Configuración principal de Jekyll
├── _includes/
│   ├── head.html        # Cabecera HTML
│   └── sidebar.html     # Barra lateral (solo Home y About)
├── _layouts/
│   ├── default.html     # Layout principal (modificado: <br> entre título y tagline)
│   ├── page.html        # Layout para páginas estáticas
│   └── post.html        # Layout para posts
├── _posts/              # Posts publicados (formato: YYYY-MM-DD-titulo.md)
├── public/
│   └── css/
│       ├── lanyon.css   # CSS del tema Lanyon
│       ├── poole.css    # CSS base de Poole
│       └── syntax.css   # CSS para resaltado de código
├── about.md             # Página About
├── index.html           # Página principal (muestra excerpt + Read more)
├── Gemfile              # Dependencias Ruby
├── .gitignore           # Excluye _site/
└── BLOG_CONTEXT.md      # Este archivo
```

---

## Configuración actual — _config.yml

```yaml
title:               Low Priv Notes
tagline:             'Exploring alternative exploitation paths'
description:         'Notes, CTF writeups and security research. Focusing on alternative privilege escalation vectors.'
url:                 'https://renekerr.github.io'
baseurl:             ''
paginate:            5
paginate_path:       "/page:num"
permalink:           pretty

author:
  name:              Rene Kerr
  url:               https://github.com/renekerr

plugins:
  - jekyll-paginate

markdown:            kramdown
```

---

## Convenciones de posts

### Nombre de archivo
```
YYYY-MM-DD-titulo-del-post.md
```

### Front matter obligatorio
```yaml
---
layout: post
title: "Título del post"
date: YYYY-MM-DD
categories: [categoria]
tags: [tag1, tag2, tag3]
---
```

### Categorías en uso
- `ctf` — writeups de CTF (TryHackMe, HackTheBox)
- `websec` — laboratorios de seguridad web (PortSwigger)
- `privesc` — técnicas de escalada de privilegios
- `notes` — notas y referencias técnicas

### Tags frecuentes
`thm`, `htb`, `portswigger`, `sqli`, `xss`, `csrf`, `ssrf`, `privesc`, `linux`, `windows`, `sudo`, `suid`, `wget`, `python`, `bash`

### Estructura estándar de un post CTF
```markdown
---
layout: post
title: "Nombre CTF — Subtítulo descriptivo"
date: YYYY-MM-DD
categories: [ctf]
tags: [thm, privesc, linux]
---

## Overview

- **Platform:** TryHackMe / HackTheBox
- **Difficulty:** Easy / Medium / Hard
- **Focus:** descripción del foco del post

---

## Reconnaissance

## Enumeration

## Initial Access

## Privilege Escalation

### Vector 1 — Nombre del vector

### Vector 2 — Nombre del vector

## Key Takeaways

## References
```

---

## Modificaciones al tema Lanyon

### 1. Título y tagline en líneas separadas
En `_layouts/default.html` se añadió `<br>` entre el título y el tagline:

```html
<h3 class="masthead-title">
  <a href="{{ site.baseurl }}/" title="Home">{{ site.title }}</a>
  <br>
  <small>{{ site.tagline }}</small>
</h3>
```

### 2. Sidebar simplificado
`_includes/sidebar.html` muestra únicamente Home y About — eliminados los enlaces a Download, GitHub project y versión.

### 3. Index con excerpt
`index.html` muestra `{{ post.excerpt }}` en lugar de `{{ post.content }}`, seguido de un enlace "Read more →".

---

## Flujo de trabajo

### Arrancar el servidor local
```bash
cd ~/Documents/PENTESTING_KB/repos/renekerr.github.io
bundle exec jekyll serve
```

Si el puerto 4000 está ocupado:
```bash
lsof -ti:4000 | xargs kill -9
bundle exec jekyll serve
```

Blog disponible en: http://localhost:4000

### Publicar un post nuevo
1. Crear archivo en `_posts/` con formato `YYYY-MM-DD-nombre.md`
2. Añadir front matter correcto
3. Escribir en Markdown estándar (sin links internos de Obsidian)
4. Hacer push:

```bash
git add .
git commit -m "Add post: nombre del post"
git push
```

El blog se actualiza en https://renekerr.github.io en 1-2 minutos.

---

## Gemfile

```ruby
source "https://rubygems.org"

gem "jekyll", "~> 4.4"
gem "jekyll-paginate"
gem "webrick"
```

---

## Entorno

- **macOS** — MacBook Air M3
- **Ruby:** 4.0.5 (instalado via Homebrew)
- **Jekyll:** 4.4.1
- **Bundler:** 4.0.16
- **PATH Ruby:** `/opt/homebrew/opt/ruby/bin:/opt/homebrew/lib/ruby/gems/4.0.0/bin`
- **Repositorio local:** `~/Documents/PENTESTING_KB/repos/renekerr.github.io`
- **Autenticación GitHub:** Personal Access Token (scope: repo)

---

## Reglas del proyecto

1. No incluir flags, contraseñas ni hashes reales en ningún post
2. No usar links internos de Obsidian (`[[archivo]]`) en posts del blog
3. No subir `_site/` al repositorio (está en `.gitignore`)
4. Los posts documentan vectores alternativos — no la vía estándar que usa todo el mundo
5. Markdown estándar en todos los posts

---

*Última actualización: Julio 2026*
