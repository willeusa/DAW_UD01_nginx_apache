# Cuaderno de Prácticas de Laboratorio: Fundamentos y Despliegue Declarativo de Servidores Web
**Módulo Profesional:** 0614 – Despliegue de Aplicaciones Web  
**Marco Curricular:** Real Decreto 405/2023, de 29 de mayo (BOE núm. 132, 03/06/2023)  
**Ciclo Formativo:** Técnico Superior en Desarrollo de Aplicaciones Web (DAW) – 2º Curso  
**Centro Docente:** Campus Cámara Comercio Sevilla  
**Temporalización:** 1.er Trimestre · Semana 8 (Viernes, 08:30 – 10:25)  
**Resultados de Aprendizaje Evaluados:**
* **RA1:** Implanta arquitecturas web analizando y aplicando criterios de funcionalidad (Criterios: *b, c, f, g, i*).
* **RA2:** Implanta aplicaciones web en servidores web, evaluando y aplicando criterios de configuración para su funcionamiento seguro (Criterios: *a, b, c, d, f, g, h, j*).
* **RA5:** Verifica la ejecución de aplicaciones web comprobando los parámetros de configuración de servicios de red (Criterios: *a, b, g*).

---

## 1. Contexto Profesional del Proyecto

> **Entorno de Simulación:**  
> La consultora tecnológica sevillana *GuadalTech Soluciones Digitales S.L.* ha asumido la auditoría y estabilización de la infraestructura de su cliente *Iberian Logistic Hub*. La auditoría preliminar ha destapado prácticas contrarias a los estándares de la industria: modificaciones directas sobre entornos en producción, errores sintácticos que bloquean el arranque de demonios web, permisos de archivo indiscriminados (`chmod 777`) y caídas de servicio continuadas durante despliegues no planificados.
>
> **Tu Rol en la Empresa:**  
> Como **Ingeniero de Despliegue / DevOps Junior**, se te encomienda sanear los entornos de pruebas y preproducción, auditar los puertos y sockets de red, desacoplar sitios virtuales mediante arquitecturas modulares, mitigar riesgos perimetrales catalogados en **OWASP Top 10** y documentar exhaustivamente el procedimiento técnico con trazabilidad forense.

---

## 2. Requisitos del Entorno Técnico

Cada alumno trabajará sobre una máquina virtual o contenedor Linux (Debian 12 / Ubuntu Server 24.04 LTS) provisto de los siguientes paquetes base:
* Servidor web **Nginx** (`nginx/1.24+`) y **Apache HTTP Server** (`apache2/2.4+`).
* Herramientas de red y diagnóstico: `iproute2` (`ss`), `lsof`, `curl`, `strace`, `bind9-dnsutils` (`dig`).
* Intérpretes de soporte: `python3` (para simulación de microservicios).
* Acceso a privilegios de superusuario mediante `sudo`.

---

## 3. Bloque de Ejercicios Prácticos de Laboratorio

```
                      ESQUEMA METODOLÓGICO DE LA PRÁCTICA
  ┌────────────────────────────────────────────────────────────────────────┐
  │  [Fase 1] Auditoría Sintáctica y Contextos (nginx -t / nginx -T)       │
  │     │                                                                  │
  │     ▼                                                                  │
  │  [Fase 2] Resolución de Rutas en Disco (root vs. alias)                │
  │     │                                                                  │
  │     ▼                                                                  │
  │  [Fase 3] Inspección de Sockets y Conflicto bind() (ss / lsof)         │
  │     │                                                                  │
  │     ▼                                                                  │
  │  [Fase 4] Despliegue Modular Multi-Host (Nginx vs. Apache)             │
  │     │                                                                  │
  │     ▼                                                                  │
  │  [Fase 5] Recarga en Caliente Zero-Downtime (Señal SIGHUP)             │
  │     │                                                                  │
  │     ▼                                                                  │
  │  [Fase 6] Hardening OWASP A01/A05 y Trazabilidad RGPD                  │
  └────────────────────────────────────────────────────────────────────────┘
```

---

### Ejercicio 1: Auditoría Forense de Contextos y Gramática Declarativa (Nginx)

#### Enunciado
Un técnico novato modificó el fichero `/etc/nginx/sites-available/portal.conf` dejando una configuración rota que impide arrancar el servidor web. Se te entrega el siguiente volcado íntegro:

```nginx
# Archivo defectuoso entregado por el cliente
server {
    listen 80
    server_name portal.guadaltech.local;

    events {
        worker_connections 1024;
    }

    root /var/www/portal/html

    location / {
        index index.html;
        server_tokens off;
    }

    location /metricas {
        worker_processes auto;
        try_files $uri $uri/ =404;
    }
}
```

#### Tareas a Desarrollar
1. **Identificación estática de errores de ámbito (*Context / Scope*):**
   * Consulta el *Alphabetical Index of Directives* oficial de Nginx (`nginx.org/en/docs/dirindex.html`).
   * Justifica por qué las directivas `events` y `worker_processes` provocan un fallo fatal de inicio dentro de este archivo.
2. **Corrección de errores sintácticos:**
   * Localiza las líneas donde se ha infringido la delimitación básica de instrucciones en Nginx.
3. **Auditoría dinámica CLI:**
   * Ejecuta `sudo nginx -t` y documenta el mensaje exacto emitido por el compilador de configuración.
   * Ejecuta `sudo nginx -T` y explica qué utilidad tiene este comando para auditar la inclusión modular de archivos frente a `nginx -t`.
4. **Refactorización profesional:**
   * Reescribe el archivo en `/etc/nginx/sites-available/portal.conf` situando cada directiva en su contexto legítimo (`main`, `http`, `server` o `location`).

---

### Ejercicio 2: El Dilema del Sistema de Archivos (`root` vs. `alias`)

#### Enunciado
La plataforma debe servir la web corporativa desde `/var/www/portal/html/` y, de forma simultánea, permitir la descarga de informes de auditoría alojados en una partición externa `/mnt/storage/informes/` bajo la URI `/documentos/`.

El cliente reporta que al ejecutar:
```bash
curl -I http://portal.guadaltech.local/documentos/auditoria-2026.pdf
```
el servidor responde invariablemente con un código `404 Not Found`, a pesar de que el archivo existe en `/mnt/storage/informes/auditoria-2026.pdf` con permisos `644` y propietario `www-data:www-data`.

Se observa el siguiente bloque en la configuración:
```nginx
location /documentos/ {
    root /mnt/storage/informes/;
    try_files $uri $uri/ =404;
}
```

#### Tareas a Desarrollar
1. **Deducción matemática de resolución:**
   * Aplica la fórmula de concatenación de la directiva `root`:
     ```text
     Ruta Física Resultante = root + URI
     ```
   * Escribe la ruta absoluta exacta en la que Nginx intentó localizar el fichero.
2. **Aplicación de la directiva `alias`:**
   * Corrige el bloque `location` utilizando `alias` bajo la fórmula:
     ```text
     Ruta Física Resultante = alias + URI - location
     ```
   * Demuestra qué ocurre técnicamente si se define `alias /mnt/storage/informes` (sin barra final `/`) cuando la directiva `location` es `/documentos/`.
3. **Verificación perimetral con `curl`:**
   * Sin tocar el servidor DNS corporativo, emula la resolución DNS apuntando al socket local mediante la cabecera `Host`:
     ```bash
     curl -Iv -H "Host: portal.guadaltech.local" http://127.0.0.1/documentos/auditoria-2026.pdf
     ```
   * Aporta la captura que demuestre la respuesta `HTTP/1.1 200 OK`.

---

### Ejercicio 3: Diagnóstico de Sockets y Mitigación del Conflicto `bind()`

#### Enunciado
Al desplegar una nueva API de seguimiento en Python, el servicio aborta su inicio mostrando:
```text
OSError: [Errno 98] Address already in use
```
Simultáneamente, Nginx falla al intentar enlazarse al puerto HTTP con:
```text
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
```

#### Tareas a Desarrollar
1. **Auditoría con `ss` y `lsof`:**
   * Ejecuta el comando de inspección rápida del kernel:
     ```bash
     sudo ss -tulpn | grep -E ':(80|443|8080)'
     ```
   * Ejecuta la inspección detallada en el subsistema VFS:
     ```bash
     sudo lsof -i :80
     ```
   * Documenta en una tabla comparativa: **PID**, **Usuario de ejecución**, **FD (Descriptor)** y **Nombre del binario** que retiene el socket.
2. **Identificación de la llamada al sistema:**
   * Utiliza `strace` para interceptar la llamada de red fallida durante el arranque de Nginx:
     ```bash
     sudo strace -e trace=bind,listen nginx -g "daemon off;"
     ```
   * Copia la línea exacta de salida y explica el significado del retorno `-1 EADDRINUSE`.
3. **Saneamiento y aislamiento arquitectónico:**
   * Detén el proceso conflictivo mediante su unidad de servicio (`systemctl`).
   * Configura la API interna en Python para que haga `bind()` exclusivamente en el socket de loopback `127.0.0.1:5000`, evitando la exposición accidental en `0.0.0.0` (OWASP A05).

---

### Ejercicio 4: Implementación Paralela de Hosts Virtuales (Nginx vs. Apache)

#### Enunciado
Se debe implantar un nuevo panel de control corporativo accesible bajo el FQDN `control.guadaltech.local` escuchando en el puerto no estándar `8080`. Se requiere que el acceso al endpoint `/privado/` quede restringido por directivas de red exclusivamente a la subred de gestión `192.168.10.0/24` y a la máquina local (`127.0.0.1`), denegando cualquier otro origen con un código `403 Forbidden`.

#### Tareas a Desarrollar
1. **Implementación modular en Nginx:**
   * Archivo: `/etc/nginx/sites-available/control.conf`
   * Directivas obligatorias: `listen 8080;`, `server_name control.guadaltech.local;`, `root /var/www/control/public;`, `index index.html;`.
   * Bloque `location /privado/` con directivas `allow` y `deny`.
   * Activación mediante enlace simbólico en `sites-enabled/`.
2. **Implementación modular en Apache HTTP Server:**
   * Archivo: `/etc/apache2/sites-available/control.conf`
   * Contenedores obligatorios: `<VirtualHost *:8080>`, `<Directory "/var/www/control/public">`, y `<Location "/privado">`.
   * Control de acceso con sintaxis moderna Apache 2.4: `Require ip 192.168.10.0/24 127.0.0.1` y `Require all denied`.
   * Activación con utilidades nativas Debian: `sudo a2ensite control.conf`.
3. **Auditoría de puertos:**
   * Añade en `/etc/apache2/ports.conf` la directiva `Listen 8080` para evitar fallos de enlace.
   * Verifica la coexistencia de ambos demonios en puertos diferenciados mediante `sudo ss -tulpn`.

---

### Ejercicio 5: Recarga en Caliente *Zero-Downtime* (Mecánica de la Señal `SIGHUP`)

#### Enunciado
En entornos corporativos con miles de usuarios conectados, está estrictamente prohibido ejecutar `systemctl restart nginx` salvo mantenimiento planificado. Debes aplicar un cambio en el archivo `portal.conf` y demostrar que el *Master Process* sustituye a los *Worker Processes* sin interrumpir peticiones activas.

#### Tareas a Desarrollar
1. **Identificación de la jerarquía de procesos:**
   * Obtén el árbol de procesos activos antes del cambio:
     ```bash
     ps -ef --forest | grep -E 'nginx: (master|worker)'
     ```
   * Registra el PID del *Master Process* (ejecutado por `root`) y los PIDs de los *Worker Processes* (ejecutados por `www-data`).
2. **Inyección de la señal `SIGHUP`:**
   * Aplica un cambio menor en la configuración (ej. variar un comentario o añadir un parámetro de log).
   * Valida la sintaxis: `sudo nginx -t`.
   * Envía la señal `SIGHUP` al *Master Process* mediante las dos vías profesionales equivalentes:
     * Vía binaria POSIX: `sudo kill -HUP $(cat /run/nginx.pid)`
     * Vía systemd: `sudo systemctl reload nginx`
3. **Auditoría de reemplazo de trabajadores:**
   * Vuelve a ejecutar `ps -ef --forest | grep -E 'nginx: (master|worker)'`.
   * Justifica por qué el PID del *Master Process* se mantuvo idéntico mientras que los PIDs de los *Workers* cambiaron por completo.
   * Explica cómo se garantiza el *zero-downtime* para un cliente que estaba transfiriendo un archivo en el momento exacto de la recarga.

---

### Ejercicio 6: *Hardening* Perimetral (OWASP Top 10) y Trazabilidad Legal (RGPD)

#### Enunciado
El servidor web perimetral debe ser asegurado contra vectores de reconocimiento automatizado y fuga de metadatos, garantizando que los ficheros críticos del proyecto queden inaccesibles desde el exterior.

#### Tareas a Desarrollar
1. **Mitigación de *Banner Grabbing* (OWASP A05 – *Security Misconfiguration*):**
   * Deshabilita la emisión de la versión exacta de Nginx en cabeceras de respuesta mediante `server_tokens off;`.
   * Comprueba con `curl -I http://127.0.0.1/` que la cabecera `Server:` muestra únicamente `Server: nginx` sin números de versión.
2. **Bloqueo perimetral de artefactos sensibles (OWASP A01 – *Broken Access Control*):**
   * Añade en el servidor Nginx una regla basada en expresiones regulares para bloquear peticiones dirigidas a variables de entorno (`.env`), repositorios Git (`.git`) o ficheros de configuración ocultos:
     ```nginx
     location ~ /\.(env|git|htaccess|dockerignore) {
         deny all;
         return 404;
     }
     ```
   * Justifica técnicamente por qué devolver un código `404 Not Found` en lugar de `403 Forbidden` constituye una buena práctica de seguridad frente a escáneres defensivos.
3. **Trazabilidad de accesos (Cumplimiento RGPD / LOPDGDD):**
   * Configura registros de acceso independientes para cada sitio virtual en `/var/log/nginx/` con el formato estándar `combined`.
   * Escribe la directiva necesaria para registrar la IP real del cliente en lugar de la dirección de bucle local cuando se sitúa tras un proxy:
     ```nginx
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     ```
   * Inspecciona con `tail -f /var/log/nginx/portal_access.log` y verifica que tu petición queda registrada con marca temporal, IP de origen, verbo HTTP y código de respuesta.

---

## 4. Checklist de Entregables y Evidencias Obligatorias

El trabajo se entregará en la plataforma virtual del Campus en un único archivo comprimido `DAW_S8_APELLIDO_NOMBRE.tar.gz` antes de las **10:25 horas** de la sesión práctica.

- [ ] **Documento de Memoria Técnica (`README.md` o PDF):**
  - Justificación razonada de los fallos encontrados en cada ejercicio.
  - Explicación de los comandos CLI utilizados y análisis de los resultados obtenidos.
- [ ] **Archivos de Configuración Refactorizados:**
  - `portal.conf` (Nginx, corregido y funcional).
  - `control_nginx.conf` y `control_apache.conf` (Virtual Hosts en puerto 8080).
- [ ] **Evidencias Empíricas (Capturas de terminal o archivos de volcado):**
  - Salida de `nginx -t` confirmando validación sintáctica limpia.
  - Salida de `ss -tulpn` evidenciando los puertos 80, 443 y 8080 en estado `LISTEN`.
  - Captura del comando `ps --forest` antes y después del envío de `SIGHUP`.
  - Prueba de descarga denegada a `.env` mediante `curl -I`.
  - Fragmento de 5 líneas de `/var/log/nginx/portal_access.log` con peticiones trazadas.

---

## 5. Rúbrica Analítica de Evaluación Profesional

Alineada con los criterios de evaluación del **Real Decreto 405/2023** para el módulo **0614 Despliegue de Aplicaciones Web**:

| Criterio Evaluado | Insuficiente (0.0 - 4.9) | Notable (5.0 - 8.4) | Sobresaliente (8.5 - 10.0) |
| :--- | :--- | :--- | :--- |
| **Arquitectura de Sockets y Red (RA1.b, RA1.c, RA2.a)** [Ponderación: 25%] | No comprende la llamada `bind()`; es incapaz de identificar colisiones de puerto o interpreta erróneamente la salida de `ss` y `lsof`. | Identifica los procesos que retienen puertos y los detiene; conoce el significado del socket pero duda en la diferencia entre `0.0.0.0` y `127.0.0.1`. | Domina la tupla de red; audita puertos con soltura técnica mediante `ss` y `lsof`; justifica la vinculación segura de interfaces en loopback. |
| **Sintaxis Declarativa y Modularidad (RA2.b, RA2.c)** [Ponderación: 25%] | Ubica directivas fuera de contexto (`events` dentro de `server`); manipula ficheros sin enlaces simbólicos; reinicia servicios sin validar sintaxis. | Configura los sitios modulares en `sites-available`/`sites-enabled`; corrige la sintaxis usando `nginx -t`, pero muestra fallos de herencia entre bloques. | Implementa modularidad desacoplada estricta; estructura jerarquías limpias (`http` > `server` > `location`); audita directivas compiladas con `nginx -T`. |
| **Resolución de Rutas y File System (RA2.c, RA2.h)** [Ponderación: 20%] | Confunde `root` con `alias`; no deduce la ruta física resultante, provocando persistencia en errores 404/403 sin diagnóstico. | Utiliza `alias` para directorios alternativos de forma funcional, pero omite barras de cierre o no justifica la fórmula de cálculo de rutas. | Resuelve y justifica matemáticamente la concatenación con `root` y sustitución con `alias`; aplica permisos acordes a `www-data` (`755/644`). |
| **Seguridad Perimetral y Zero-Downtime (RA2.d, RA2.f, RA2.j)** [Ponderación: 20%] | Ejecuta `restart` destructivo en caliente; deja versiones visibles (`server_tokens on`); expone ficheros sensibles (`.env`) en la red pública. | Aplica `systemctl reload` y oculta firmas básicas del servidor web; bloquea rutas sensibles pero devuelve códigos reveladores (403). | Demuestra la sustitución de workers con `SIGHUP`; implementa hardening estricto (OWASP A01/A05, código 404 camuflado); parametriza logs según RGPD. |
| **Rigor Metodológico y Documentación (RA1.i, RA2.g)** [Ponderación: 10%] | No sigue el checklist; entrega incompleta, sin capturas de evidencia técnica o sin justificación de las decisiones tomadas. | Entrega la mayoría de evidencias requeridas; memoria técnica correcta aunque centrada únicamente en el "cómo" y no en el "porqué". | Documentación profesional con formato Markdown/PDF impecable; incluye justificación técnica en cada paso, capturas claras y logs auditables. |
