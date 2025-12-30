---
title: Fiverr Scam investigación
layout: post
image: 
    path: /assets/covers/fiverr.png
autor: AreiaNight
tags: [OSINT, Análisis de Phishing, Inteligencia de Amenazas, Ciberseguridade]
category: Pentesting
---

# Investigando una Operación Sofisticada de Phishing Multi-Dominio en Fiverr



## El background

Hola bandita, hoy les traigo una pequeña investigación que si están en nuestro grupo de discord ya sabes cual es. Mientras buscaba trabajo como freelancer en Fiverr, me empezaron a llegar mensajes (y aún los recibo, lo cual es molestísimo) de una campaña de phishing sofisticada y multicapa. En lugar de simplemente ignorarlo como alguien con dos dedos de frente lo haría, decidí investigar porque no tenía nada mejor que hacer y sinceramente estaba aburrida. Lo que comenzó como molestia por mensajes spam evolucionó en descubrir una operación de fraude a gran escala con infraestructura profesional, múltiples dominios y técnicas avanzadas de evasión.

Este writeup documenta mi la investigación que hice, hallazgos y la infraestructura de una campaña activa de phishing que parece haber generado cientos de millones de links únicos de rastreo dirigidos a freelancers.

Al final descubrí una red coordinada de phishing usando 5+ dominios, arquitectura de redirección multicapa, millones de IDs de rastreo y protecciones sofisticadas anti-análisis para robar credenciales de Fiverr e información de tarjetas de crédito.

---

## Descubrimiento Inicial

### El Anzuelo

Como freelancer activa en Fiverr, recibo regularmente mensajes de potenciales clientes. Sin embargo, comencé a notar un patrón de mensajes sospechosos que contenían links que redirigían a través de Linktr.ee y acortadores de URL que ni diosa sabe que existen. Los mensajes prometían oportunidades de trabajo legítimas pero llevaban a dominios que no eran Fiverr. Cabe aclarar que Fiverr tiene un botón de report para esto porque dichas estrategias van en contra de sus normas de usuario, así que les aconsejo que no hagan lo que yo hice y simplemente bloquen y reporten, las cuentan suelen bajarlas con las suficientes denuncias.

**Red flags**
- Mensajes no solicitados con "oportunidades" urgentes
- Links que pasan por Linktr.ee y acortadores de URL
- Solicitudes para "verificar" cuenta fuera de la plataforma de Fiverr
- Promesas de órdenes ya pagadas

---

## Investigación

### Herramientas Utilizadas

**Reconocimiento:**
- `whatweb` - Fingerprinting HTTP y la que más usé
- `whois` - Información de registro de dominios
- `dig`/`nslookup` - Enumeración DNS
- Shodan - Escaneo de puertos e identificación de servicios
- `curl` - Análisis de headers HTTP

**Plataformas de Análisis:**
- URLScan.io - Análisis seguro de URLs con sandboxing
- VirusTotal - Detección de malware/phishing
- SecurityTrails - Datos históricos de DNS
- crt.sh - Logs de transparencia de certificados SSL

**Medidas de Seguridad:**
- Todo el análisis se realizó en máquinas virtuales aisladas
- Nunca se proporcionaron credenciales reales
- Monitoreo de tráfico de red durante todo el proceso

---

## Dominios e ips 

### Portafolio de Dominios

A través de investigación sistemática, identifiqué **5 dominios principales** y evidencia de muchos más:

#### 1. fivmenu.com
**Dominio principal de phishing**

```
IPs: 104.21.75.174, 172.67.179.180
Backend: Express.js (Node.js)
Proxy: Cloudflare
Nombres de Cookies: connect.sid, verified
```

**Huella técnica (whatweb):**
```
[200 OK] 
Cookies[connect.sid,verified]
HTTPServer[cloudflare]
HttpOnly[connect.sid]
IP[172.67.179.180]
Title[Human Verification • Fiverr Style]
X-Powered-By[Express]
```

**Patrón de URL:** `https://fivmenu.com/[ID de 8-9 dígitos]`

**Ejemplos de IDs descubiertos:**
- 263079257
- 243396209
- 216345288

**Análisis:** El rango de 47 millones de IDs (216M a 263M) sugiere millones de links únicos de rastreo generados, indicando escala masiva. (Según cloude)

#### 2. fiverrmenu.com
**Variante con doble "rr" - variación de typosquatting**

Descubierto como destino de redirección desde dominios desechables. Misma infraestructura que fivmenu.com pero con nombre modificado para evadir bloqueos.

#### 3. completeglg.digital
**Dominio secundario de phishing con enfoque diferente**

```
IP: 104.21.3.50
Backend: Express.js
Proxy: Cloudflare
Registrador: Mat Bao Corporation (Vietnam)
```

**Huella técnica:**
```
[200 OK]
Cookies[connect.sid]
Country[UNITED STATES][US]
HTTPServer[cloudflare]
IP[104.21.3.50]
Title[Human Verification]
X-Powered-By[Express]
```

**Patrones de URL:**
- `/res/manual-export` - Página de verificación de username
- `/f/[ID de 8 dígitos]` - Dashboard falso (ej., `/f/84916365`)

**Sofisticación:** Este dominio implementa validación de username, requiriendo el formato @username de Fiverr antes de proceder.

#### 4. atom2.digital
**Dominio redirector desechable**

```
Registrador: Mat Bao Corporation (Vietnam)
Propósito: Capa de redirección para proteger dominios principales
```

**Función:** Actúa como intermediario que redirige a fiverrmenu.com

**Descubrimiento:** Encontrado a través de mensaje spam con patrón `https://atom2.digital/project/status210`

**Hipótesis:** Parte de una serie (atom1, atom3, atom4...) usados como redirectores desechables. Cuando uno es bloqueado, rotan al siguiente.

#### 5. fiverr-to.offer-feedback.com
**Dominio más sofisticado - estructura de subdominio**

```
Estructura: Subdominio de offer-feedback.com
Protección: Validación avanzada de Host header
Anti-Análisis: Bloquea acceso directo por IP, previene escaneo nmap
```

**Sofisticación técnica:**
- Requiere header `Host:` válido para responder (devuelve 403 de lo contrario)
- Previene herramientas de escaneo automatizado
- Typosquatting de subdominio: "fiverr-to" imita URLs oficiales de Fiverr
- Dominio base "offer-feedback.com" suena legítimo

**Amenaza:** Podría generar subdominios infinitos para diferentes plataformas:
- `upwork-to.offer-feedback.com`
- `freelancer-to.offer-feedback.com`
- etc.

---

## Infraestructura

### Rangos de Direcciones IP

Todos los dominios resuelven a la red de Cloudflare, específicamente:

```
NetRange: 104.16.0.0 - 104.31.255.255
CIDR: 104.16.0.0/12
Organización: Cloudflare, Inc.

IPs específicas identificadas:
- 104.21.3.50    (completeglg.digital)
- 104.21.75.174  (fivmenu.com)
- 104.21.76.210  (dominios adicionales sospechosos)
- 172.67.179.180 (fivmenu.com secundaria)
```

**Patrón:** IPs en subredes secuenciales o cercanas sugieren configuración coordinada de infraestructura, probablemente por los mismos operadores registrando múltiples dominios a través de Cloudflare simultáneamente.

### Resultados de Escaneo de Puertos (Shodan)

```
Puerto 80 (HTTP):    ABIERTO
Puerto 443 (HTTPS):  ABIERTO
Todos los demás:     FORBIDDEN / BAD REQUEST
```

Acceso directo por IP: **403 FORBIDDEN** (requiere Host header válido)

**Evaluación:** Configuración de seguridad profesional. No es operación amateur.

### Registro de Dominios

**Patrón de Mat Bao Corporation:**
- completeglg.digital - Registrado vía Mat Bao (Vietnam)
- atom2.digital - Registrado vía Mat Bao (Vietnam)

```
Registrador: MAT BAO CORPORATION
Servidor WHOIS: whois.matbao.net
Website: http://www.matbao.net
Contacto de Abuso: abuse@matbao.net
```

**Elección estratégica:** Usar registrador vietnamita puede complicar esfuerzos de cierre debido a temas de jurisdicción internacional.

### Tecnología Backend

**Consistente en todos los dominios:**
- Framework: Express.js (Node.js)
- Proxy: Cloudflare (protección DDoS, enmascaramiento de IP)
- Gestión de Sesión: cookie `connect.sid` (default de Express)
- Rastreo Personalizado: cookie `verified` (solo fivmenu.com)

**Indicadores de calidad de código:**
- Diseño UI/UX profesional
- Validación de inputs (verificación de formato de username)
- Flujo de usuario multi-etapa
- Diseño responsive
- Implementación apropiada de HTTPS

---

## El ataque

### Etapa 1: Contacto Inicial y Distribución

**Vector:** Mensajes spam vía sistema de mensajería interno de Fiverr

**Arquitectura de distribución:**
```
1. Mensaje Spam en Fiver
2. Acortador de URL (bit.ly, tinyurl, etc.) O perfil Linktr.ee
3. Redirector Desechable (atom2.digital, etc.)
4. Dominio Principal de Phishing (fivmenu.com, completeglg.digital, etc.)
```

**Técnica de evasión:** Múltiples capas protegen los dominios principales de exposición directa y permiten rotación rápida de dominios intermediarios "quemados".

### Etapa 2: Recolección de Usernames

Aquí las páginas cambian un poco, algunas de éstas te piden tu username sin verificación a lo que puedes poner cosas como "RosaMelano69", sin embargo en otras **sí** hay verificación de usuarios, por lo que de alguna forma tiene acceso a una base de datos con todos los usuarios de Fiverr activos e inactivos. Quizá algún leak o simplemente tienen una forma de saber cuales son los nuevos usuarios (mi cuenta no tenía un una semana activa). 

**URL de ejemplo:** `https://completeglg.digital/res/manual-export`

**Contenido de la página:**
```
"¿Dónde encontrar tu @username de Fiverr?"

Ve a tu perfil de Fiverr — tu @username es visible 
justo debajo de tu foto, así:

[Muestra screenshot falso de perfil de Fiverr con @username resaltado]

Tu username de Fiverr siempre empieza con "@". Puedes 
incluir @ o no — solo evita espacios extra.

[Campo de entrada para @username]
[Botón: "Go to Profile"] [Botón: "Got it"]
```
### Etapa 3: Dashboard de Orden Falsa

**URL de ejemplo:** `https://completeglg.digital/f/84916365`

**La página muestra:**

```
[Header falso de Fiverr con logo, barra de búsqueda, notificaciones]

Orden: "Crearé arte único para dar vida a tus ideas usando ia"
Monto: €18

- Estado de pago: Pagado por el Comprador
- Estado de orden: Esperando tu aceptación
- Tiempo de procesamiento: Los fondos serán liberados a tu cuenta dentro de 15-60 minutos después de la confirmación
- Seguridad de fondos: Tu pago está retenido de forma segura hasta  que confirmes la orden

[Botón: "Proceed to Dashboard"]
```

**Manipulación psicológica:**
1. **Urgencia:** "15-60 minutos" crea presión de tiempo
2. **Autoridad:** Diseño profesional de interfaz de Fiverr
3. **Escasez:** Monto pequeño (€18) parece realista
4. **Reciprocidad:** "El cliente ya pagó" crea obligación
5. **Confianza:** Mensajes de seguridad, diseño profesional

### El robo de información

**Método:** Widget de chat en vivo aparece (etiquetado "Support Chat"). Todo el script inicial está en el código fuente de la página, sin embargo algunos chats puede o no tener por detrás a un humano o quizá una api de chatGTP, ya que no lograron identificar de inmediato que la ciudad que dije "Vergas" no era siquiera una ciudad verdadera. Tampoco captaban cosas como mis nombres tipo: "Elver Galarga" o "Rosa Melfierro". Por lo que o bien no hablan español o simplemente es una IA básica.

**Mensaje de ejemplo de la falsa "Emma del soporte al cliente":**

```
Gracias por usar nuestro servicio. Soy Emma del soporte al cliente.

El comprador ya completó el pago de la orden. Para recibir los fondos, por favor proporciona los detalles de tu tarjeta y completa una identificación segura de una sola vez. Esto es necesario para confirmar tu identidad y asegurar transacciones seguras.

Una vez completado, los pagos futuros serán procesados instantáneamente con un solo clic.

Si tienes alguna pregunta, no dudes en contactarnos.

Saludos cordiales,
Emma
Soporte al Cliente
```

**Lo que solicitan:**
- Número completo de tarjeta de crédito
- CVV
- Fecha de expiración
- Nombre del titular
- A veces: dirección de facturación, SSN, "verificación" adicional

**Realidad:** 
- No existe ningún pago
- No hay orden real
- "Emma" es bot o estafador
- Los detalles de tarjeta van directo a los atacantes
- La víctima pierde dinero, potencialmente su identidad

---

## Tácticas de Ingeniería Social

### Técnicas de Manipulación Psicológica

**1. Sesgo de Autoridad**
- Suplanta plataforma confiable (Fiverr)
- Agente de "soporte al cliente" proporciona falsa autoridad
- Diseño profesional crea legitimidad

**2. Presión de Tiempo**
- "Ya pagado"
- Plazo de "15-60 minutos"
- "Esperando tu aceptación"

**3. Incentivo Financiero**
- Monto pequeño, creíble (€18)
- "El pago está asegurado"
- "Pagos futuros serán instantáneos"

**4. Construcción de Confianza**
- Gramática y ortografía profesional
- Mensajes detallados de seguridad
- Tono amigable ("no dudes en contactarnos")
- Consistencia visual con Fiverr real

**5. Pie en la Puerta**
- Comienza con username (petición pequeña)
- Progresa a "ver dashboard" (inofensivo)
- Finalmente solicita detalles de tarjeta (gran petición)
- Cada paso construye compromiso

### Por Qué Funciona

**Selección de objetivo:** Los freelancers que buscan trabajo activamente:
- Esperan consultas de clientes
- Están motivados por necesidad financiera
- Están acostumbrados a comunicaciones de la plataforma
- Pueden no escrutar cada mensaje cuidadosamente

**Calidad de ejecución:** Depende del atacante, pero en su mayoría es bastante pátetico. Links obviamente dudosos, screenshots con "pruebas" junto con un qr para evadir las moderaciones. Sinceramente, si caes en esto es porque eres muy nuevo en internet o jamás te habías topado con un phishing o el concpeto de este. Quizá es mi propio sesgo hablando, pero realmente considero que podría ser más légitimo. 

---

## Escala de la Operación

### Indicadores Cuantitativos

**Secuencias de ID observadas:**
```
Rangos de fivmenu.com: 216,345,288 a 263,079,257
Rango: ~47 millones de IDs únicos

completeglg.digital: 84,916,365 (IDs de 8 dígitos)
atom2.digital: status210 (formato diferente)
```

**Estimaciones conservadoras:**
- 47,000,000 links únicos de rastreo generados
- Incluso con tasa de éxito del 0.1%: 47,000 víctimas potenciales
- A €18 promedio de fraude: €846,000 en robo
- La realidad probablemente es mayor con montos variables solicitados

### Inversión en Infraestructura

**Costos incurridos por operadores:**
- Múltiples registros de dominios ($10-50 cada uno anualmente)
- Características Pro de Cloudflare (opcional, $20-200/mes)
- Desarrollo Express.js (código personalizado)
- Integración de sistema de chat (LiveChat, Intercom, o personalizado)
- Diseño gráfico (imágenes de órdenes falsas, mockups de UI)
- Operación y mantenimiento continuo

**Conclusión:** Esto NO es un amateur solitario. Organización criminal profesional con financiamiento y experiencia técnica.

### Escala de Distribución

**Evidencia de campaña masiva:**
- Mensajes spam continuos en Fiverr (observados durante semanas)
- Múltiples perfiles de Linktr.ee creados repetidamente
- Cuentas siendo baneadas y recreadas constantemente
- Múltiples acortadores de URL rotados
- Múltiples dominios desechables para capa de redirección

**Línea de tiempo operacional:** Probablemente activa por meses o años, dado los rangos de ID y complejidad de infraestructura.

---

## Mecanismos de Evasión y Persistencia

### Arquitectura Multi-Capa

La operación usa una arquitectura sofisticada de tres capas diseñada para resiliencia:

```
Capa 1: DISTRIBUCIÓN (Fácilmente reemplazable)
1. Cuentas spam de Fiverr (creadas/baneadas continuamente)
2. Perfiles Linktr.ee (desechables)
3. Acortadores de URL (servicios rotados)

Capa 2: REDIRECTORES (Escudos desechables)
1. atom1.digital
2. atom2.digital  ← Confirmado
3. atom3.digital
4. atom[N].digital  (hipótesis: muchos más existen)

Capa 3: INFRAESTRUCTURA PRINCIPAL (Protegida)
1. fivmenu.com
2. fiverrmenu.com
3. completeglg.digital
4. fiverr-to.offer-feedback.com
```

**Estrategia:** Cuando la Capa 1 o 2 es bloqueada, los operadores simplemente rotan al siguiente recurso desechable mientras la Capa 3 (costosa, infraestructura core) permanece protegida y operacional.

### Protecciones Anti-Análisis

**1. Proxy de Cloudflare**
- IP real del servidor oculta
- Protección DDoS
- Limitación de tasa
- Detección de bots

**2. Validación de Host Header**
- Acceso directo por IP devuelve 403 Forbidden
- Requiere header `Host:` válido para responder
- Derrota escáneres automatizados básicos

**3. Avanzado (fiverr-to.offer-feedback.com)**
- Bloquea escaneo de puertos `nmap`
- Solo responde a requests HTTP formateados apropiadamente
- Implementa WAF (Web Application Firewall)

**4. Rastreo Basado en Cookies**
- Cookie `verified` rastrea progreso del usuario
- `connect.sid` mantiene estado de sesión
- Permite a operadores rastrear viaje de la víctima

### Estrategia de Dominios

**Variaciones de typosquatting:**
```
fivmenu.com      → Sustitución simple
fiverrmenu.com   → Variación de letra doble
completeglg      → Imitación de autoridad (GLG es firma real de consultoría)
fiverr-to.*      → Imitación de componente de ruta
offer-feedback   → Nombre genérico que suena legítimo
```

**Explotación de subdominios:**
```
fiverr-to.offer-feedback.com  ← Descubierto
[plataforma]-to.offer-feedback.com  ← Objetivos futuros potenciales:
  - upwork-to.offer-feedback.com
  - freelancer-to.offer-feedback.com
  - guru-to.offer-feedback.com
```

Un solo dominio base puede generar variantes de ataque infinitas para diferentes plataformas.

---

## Evolución de la Sofisticación

### Generación 1: Phishing Básico (fivmenu.com)

**Características:**
- Typosquatting de dominio simple
- Protección básica de Cloudflare
- Rastreo secuencial de IDs
- Acceso directo posible

**Stack tecnológico:**
- Backend Express.js
- Cookies estándar
- HTML/CSS/JS simple

### Generación 2: Intermedio (completeglg.digital, atom2.digital)

**Mejoras:**
- Validación de username
- Arquitectura de redirección
- Múltiples dominios
- Registrador vietnamita (complejidad de jurisdicción)

**Características agregadas:**
- Verificación de formato de entrada
- Flujo multi-etapa
- Redirectores desechables

### Generación 3: Avanzado (fiverr-to.offer-feedback.com)

**Salto de sofisticación:**
- Estructura de subdominio
- Validación de Host header
- Protecciones anti-escaneo
- Implementación de WAF
- Dominio base genérico para uso multi-plataforma

**Evaluación:** Los operadores están aprendiendo de intentos de cierre y evolucionando sus técnicas. Esto indica:
- Monitoreo activo de su infraestructura
- Adaptación a medidas de seguridad
- Planeación operacional a largo plazo
- Adversario profesional, no amateur oportunista

---

## Indicadores Técnicos de Compromiso (IOCs)

### Dominios
```
fivmenu.com
fiverrmenu.com
completeglg.digital
atom2.digital
fiverr-to.offer-feedback.com
offer-feedback.com (dominio base)
```

### Direcciones IP
```
104.21.3.50
104.21.75.174
104.21.76.210
172.67.179.180
```

### Patrones de URL
```
/[número de 8-9 dígitos]       (ej., /263079257)
/res/manual-export
/f/[número de 8 dígitos]        (ej., /f/84916365)
/project/status[número]         (ej., /project/status210)
```

### Firmas Técnicas
```
Headers HTTP:
X-Powered-By: Express
Server: cloudflare

Cookies:
connect.sid (todos los dominios)
verified (solo fivmenu.com)

Títulos:
"Human Verification • Fiverr Style"
"Human Verification"
"Fiverr - Freelance Services"
```

### Indicadores de Ingeniería Social
```
Frases en contenido:
"¿Dónde encontrar tu @username de Fiverr?"
"El cliente ha pagado por tu servicio"
"proporciona los detalles de tu tarjeta y completa una identificación segura de una sola vez"
"Los fondos serán liberados dentro de 15-60 minutos"
"Soy [Nombre] del soporte al cliente"
```

### Patrones de Infraestructura
```
Registrador: Mat Bao Corporation
WHOIS: whois.matbao.net
ASN: AS13335 (Cloudflare)
Tecnología: Express.js (Node.js)
```

---

## Recomendaciones Defensivas

### Para Freelancers (Víctimas Potenciales)

**Banderas rojas a vigilar:**
1. Mensajes no solicitados con links a través de Linktr.ee o acortadores de URL
2. Solicitudes para "verificar" tu cuenta fuera de la plataforma
3. Afirmaciones de órdenes ya pagadas antes de hacer cualquier trabajo
4. Urgencia y presión de tiempo ("15-60 minutos")
5. Solicitudes de tarjeta de crédito para "verificación" para recibir pagos
6. URLs que no coinciden exactamente con el dominio de la plataforma

**Mejores prácticas:**
- Verifica todas las comunicaciones a través de canales oficiales de la plataforma
- Nunca proporciones detalles de tarjeta de crédito para "recibir" un pago
- Sé escéptica de ofertas demasiado buenas para ser verdad
- Verifica URLs cuidadosamente antes de hacer clic
- Reporta mensajes sospechosos inmediatamente
- Usa 2FA en todas las cuentas de freelance

**Si eres atacada:**
- NO proporciones ninguna información personal
- NO hagas clic en links ni proporciones credenciales
- Reporta a la plataforma (Fiverr, Upwork, etc.)
- Reporta al servicio de acortador de URL (si aplica)
- Reporta a abuso de Cloudflare
- Considera reportar al FBI IC3 (si estás en EE.UU.)

### Para Operadores de Plataformas (Fiverr, etc.)

**Mecanismos de detección:**
1. Monitorear links salientes en mensajes (Linktr.ee, acortadores)
2. Bloquear dominios IOC conocidos a nivel de mensaje
3. Implementar limitación de tasa en mensajería de cuentas nuevas
4. Agregar advertencias cuando se detectan links externos
5. Educar usuarios con tips de seguridad in-app

**Procedimientos de respuesta:**
1. Suspensión inmediata de cuenta en spam confirmado
2. Compartir IOCs con otras plataformas
3. Coordinar con Cloudflare para cierres
4. Reportar a registradores de dominios
5. Trabajar con fuerzas del orden

### Para Investigadores de Seguridad

**Técnicas de investigación usadas:**
```bash
# Fingerprinting HTTP
whatweb [target_url]

# Información de registro de dominio
whois [dominio]

# Enumeración DNS
dig [dominio] ANY
nslookup [dominio]

# Información de IP
whois [dirección_ip]

# Escaneo de puertos (cuando no está bloqueado)
nmap -sV -p 80,443 [ip]

# Análisis de headers
curl -I [url]
curl -IL [url]  # Seguir redirects

# Análisis seguro de URL
# Usar: urlscan.io, virustotal.com
```

**Protocolos de seguridad:**
- Siempre usar VMs aisladas para análisis
- Nunca usar credenciales reales para pruebas
- Documentar todo con capturas de pantalla
- Monitorear tráfico de red
- Usar URLScan.io para reconocimiento seguro

**Canales de reporte:**
- Cloudflare Abuse: https://www.cloudflare.com/abuse
- Registradores de dominios: abuse@[dominio-registrador]
- Google Safe Browsing: https://safebrowsing.google.com/safebrowsing/report_phish/
- Equipos de abuso específicos de plataformas

---

## Esfuerzos de Reporte y Cierre

### Acciones Tomadas

**Reportado a:**
1. Fiverr Trust & Safety (múltiples cuentas spam)
2. Equipo de Abuso de Linktr.ee (perfiles distribuidores)
3. Cloudflare Abuse (todos los dominios)
4. Mat Bao Corporation (completeglg.digital, atom2.digital)
5. Google Safe Browsing (todos los dominios)

**Evidencia proporcionada:**
- Análisis técnico completo
- Capturas de pantalla del flujo de ataque
- Outputs de WhatWeb
- Datos WHOIS
- Listas de IOC
- Documentación de metodología de ataque

**Línea de tiempo esperada:**
- Fiverr: Horas (cuentas baneadas rápidamente)
- Linktr.ee: 24-48 horas
- Cloudflare: 2-7 días
- Registradores: 1-2 semanas
- Google Safe Browsing: 24-72 horas

### Desafíos en el Cierre

**Problemas jurisdiccionales:**
- Operadores probablemente fuera de jurisdicción EE.UU./UE
- Registrador vietnamita (Mat Bao) complica cooperación internacional
- Cloudflare con sede en EE.UU. pero solo proxy, no host

**Obstáculos técnicos:**
- IPs reales de servidor ocultas detrás de Cloudflare
- Múltiples dominios de respaldo listos para activar
- Capa de redirect desechable absorbe bloqueos iniciales
- Pueden registrar rápidamente nuevos dominios

**Resiliencia operacional:**
- Arquitectura multi-capa diseñada para persistencia
- Motivación financiera asegura operación continua
- Bajo costo para reemplazar recursos bloqueados
- Alto retorno de inversión de fraudes exitosos

**Resultado realista:**
- Dominios principales pueden ser cerrados
- Operadores probablemente se re-establecerán con nuevos dominios
- Juego continuo del gato y el ratón
- Educación y concienciación más efectivas que cierre puro

---

## Lecciones Aprendidas

### Insights Técnicos

1. **Cloudflare como espada de doble filo:**
   - Protege sitios legítimos de ataques
   - También protege atacantes de investigación y cierre
   - Ocultamiento de IP hace atribución casi imposible

2. **Arquitectura multi-capa es efectiva:**
   - Capas externas desechables absorben detección/bloqueo
   - Infraestructura core permanece protegida
   - Estrategia simple pero efectiva para persistencia

3. **Express.js es popular para fraude:**
   - Rápido de desarrollar
   - Ecosistema grande (paquetes npm)
   - Fácil de desplegar
   - Suficientemente común para no destacar

4. **El registro de dominio importa:**
   - Elección de registrador afecta complejidad de cierre
   - Protección de privacidad oculta identidad del operador
   - Registradores internacionales agregan barreras jurisdiccionales

### Metodología de Investigación

**Lo que funcionó:**
- Máquinas virtuales para análisis seguro
- WhatWeb para fingerprinting rápido
- URLScan.io para reconocimiento comprehensivo
- Documentación sistemática durante todo el proceso
- Seguir la cadena de redirección

**Lo que fue desafiante:**
- Bloqueo a nivel de IP previno algo de análisis
- Enmascaramiento de Cloudflare complicó atribución
- Requirió unir múltiples puntos de datos
- Investigación manual intensiva en tiempo

**Habilidades requeridas:**
- Fundamentos de OSINT
- Conocimiento de protocolos DNS/HTTP
- Conciencia de ingeniería social
- Paciencia y enfoque sistemático
- Disciplina de documentación

### Implicaciones Más Amplias

**Para la comunidad de ciberseguridad:**
1. Las operaciones de phishing son cada vez más sofisticadas
2. Indicadores tradicionales (gramática pobre, dominios falsos obvios) ya no son confiables
3. Necesidad de mejor compartir información entre plataformas
4. La educación del usuario sigue siendo capa de defensa crítica

**Para plataformas de freelance:**
1. Mensajería in-plataforma es vector de ataque
2. Usuarios no siempre reconocen riesgos de links externos
3. Necesidad de mejor análisis de links en tiempo real
4. Advertencias a usuarios necesitan ser más prominentes

**Para investigadores:**
1. Investigación del mundo real proporciona insights que laboratorios no pueden
2. Documentar y compartir hallazgos beneficia a la comunidad
3. Prácticas de investigación éticas son críticas
4. Protocolos de seguridad (VMs, etc.) son no negociables

---

## Conclusión

Lo que comenzó como un "vamos a ver a estos tipos" y mi propio aburriemiento evolucionó en descubrir una operación sofisticada de phishing con millones de links. La infraestructura demuestra desarrollo profesional, planeación estratégica y resiliencia operacional poco común en campañas típicas de phishing.

**Hallazgos clave:**
- **5+ dominios** identificados en 3 "familias" distintas
- **4 direcciones IP confirmadas** (todas proxied por Cloudflare)
- **47+ millones de links de rastreo** generados (estimación conservadora)
- **3 generaciones de sofisticación** mostrando evolución operacional
- **Arquitectura multi-capa** diseñada para persistencia y evasión
- **Ingeniería social de grado profesional** con alta efectividad psicológica

**Evaluación de amenaza:** Esto representa una operación criminal organizada, no un estafador oportunista. La inversión en infraestructura, la escala de targeting, y la sofisticación de técnicas indican un grupo bien financiado con experiencia técnica y planeación operacional a largo plazo.

**Impacto:** Número desconocido de víctimas, pero potencialmente miles de cuentas comprometidas y pérdidas financieras significativas. La operación permanece activa al momento de escribir esto.

**Para investigadores:** Investigaciones del mundo real como esta proporcionan inteligencia valiosa de amenazas. Compartir hallazgos ayuda a la comunidad a defenderse contra estas amenazas en evolución. Siempre prioriza seguridad y prácticas éticas.

El panorama de phishing continúa evolucionando. Los operadores aprenden, se adaptan y mejoran sus técnicas. Nuestras defensas deben evolucionar en consecuencia—no solo controles técnicos, sino conciencia del usuario y cooperación entre industrias.

---

## Línea de Tiempo de la Investigación

- **Semana 1:** Mensajes spam iniciales recibidos e ignorados
- **Semana 2:** Reconocimiento de patrón - múltiples mensajes similares
- **Día 1:** Decisión de investigar, primer análisis en VM
- **Día 1:** Descubrimiento de fivmenu.com, escaneo inicial con WhatWeb
- **Día 1:** Encontrado completeglg.digital a través de spam diferente
- **Día 1:** Análisis WhatWeb revela infraestructura idéntica
- **Día 1:** Análisis de IP muestra red de Cloudflare
- **Día 1:** Descubierta página de verificación de username en completeglg
- **Día 1:** Alcanzado dashboard falso con orden de €18
- **Día 1:** Chat de "Emma" reveló intento de recolección de tarjetas
- **Día 1:** Escaneo Shodan identificó configuración de puertos
- **Día 2:** Nuevo spam reveló dominio redirect atom2.digital
- **Día 2:** Descubierta redirección a fiverrmenu.com (nueva variante)
- **Día 2:** WHOIS reveló conexión con Mat Bao Corporation
- **Día 2:** Último descubrimiento: fiverr-to.offer-feedback.com con protecciones avanzadas
- **Día 2:** Compilado paquete de evidencia para reportes
- **Día 3:** Writeup y divulgación pública

---

## Recursos y Referencias

### Herramientas Utilizadas
- WhatWeb: https://github.com/urbanadventurer/WhatWeb
- Shodan: https://www.shodan.io/
- URLScan.io: https://urlscan.io/
- VirusTotal: https://www.virustotal.com/
- SecurityTrails: https://securitytrails.com/
- crt.sh: https://crt.sh/

### Canales de Reporte
- Cloudflare Abuse: https://www.cloudflare.com/abuse
- Google Safe Browsing: https://safebrowsing.google.com/safebrowsing/report_phish/
- FBI IC3: https://www.ic3.gov/
- ICANN Compliance: https://www.icann.org/compliance/complaint

### Lectura Adicional
- MITRE ATT&CK: Técnicas de Phishing
- OWASP: Cheat Sheet de Prevención de Phishing
- Reportes del Anti-Phishing Working Group

---

## Aviso Legal

Esta investigación se realizó con fines educativos y de investigación. Todo el análisis se realizó en entornos aislados sin proporcionar credenciales reales a sitios maliciosos. El objetivo es aumentar la concienciación y proporcionar inteligencia de amenazas accionable para ayudar a proteger a víctimas potenciales.

Si eres víctima de esta estafa, por favor:
1. Contacta a tu banco inmediatamente para congelar tarjetas
2. Reporta a la plataforma de freelance
3. Presenta un reporte policial
4. Considera reportar al FBI IC3 (si estás en EE.UU.)
