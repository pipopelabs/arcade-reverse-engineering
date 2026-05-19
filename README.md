# arcade-reverse-engineering
Reverse engineering and security research of the Arcade Jobs Android application, focused on OAuth flows, deep links, Firebase infrastructure, Expo Router, exported components, and realtime communication surfaces.


<div align="center">

```
██████╗ ██╗██████╗  ██████╗ ██████╗ ███████╗██╗      █████╗ ██████╗ ███████╗
██╔══██╗██║██╔══██╗██╔═══██╗██╔══██╗██╔════╝██║     ██╔══██╗██╔══██╗██╔════╝
██████╔╝██║██████╔╝██║   ██║██████╔╝█████╗  ██║     ███████║██████╔╝███████╗
██╔═══╝ ██║██╔═══╝ ██║   ██║██╔═══╝ ██╔══╝  ██║     ██╔══██║██╔══██╗╚════██║
██║     ██║██║     ╚██████╔╝██║     ███████╗███████╗██║  ██║██████╔╝███████║
╚═╝     ╚═╝╚═╝      ╚═════╝ ╚═╝     ╚══════╝╚══════╝╚═╝  ╚═╝╚═════╝ ╚══════╝
PipopeLabs no se hace responsable del uso de esta informacion lo realizado aqui
es exclusivamente para fines educativos y de investigacion.
```

### `security research // mobile // vol.01`

![Status](https://img.shields.io/badge/estado-en_progreso-orange?style=flat-square)
![Platform](https://img.shields.io/badge/plataforma-Android-brightgreen?style=flat-square&logo=android)
![Type](https://img.shields.io/badge/tipo-investigación_académica-blue?style=flat-square)
![Team](https://img.shields.io/badge/equipo-pipopelabs-black?style=flat-square)
![Disclosure](https://img.shields.io/badge/responsible_disclosure-aplicada-green?style=flat-square)

</div>

---

# Arcade: El inicio de todo
## Análisis de superficie de ataque en aplicaciones Android modernas

> **Resumen.** Este documento presenta los resultados de la primera etapa de una investigación académica sobre la arquitectura de seguridad de aplicaciones Android híbridas. A través de análisis estático, revisión de manifiestos y observación dinámica controlada, se identificaron patrones arquitectónicos que representan áreas de riesgo potencial en aplicaciones que combinan tecnologías nativas con frameworks JavaScript, autenticación OAuth y servicios cloud. Ningún sistema fue vulnerado, ningún dato fue accedido, y los hallazgos documentados son de naturaleza descriptiva y preventiva.

---

## Índice

1. [Introducción](#1-introducción)
2. [Metodología](#2-metodología)
3. [Stack tecnológico identificado](#3-stack-tecnológico-identificado)
4. [Hallazgos por área de análisis](#4-hallazgos-por-área-de-análisis)
5. [Superficie de ataque documentada](#5-superficie-de-ataque-documentada)
6. [Conclusiones](#6-conclusiones)
7. [Responsible Disclosure & Ética](#7-responsible-disclosure--ética)
8. [Sobre el equipo](#8-sobre-el-equipo)

---

## 1. Introducción

Las aplicaciones móviles modernas han dejado de ser sistemas monolíticos para convertirse en ecosistemas distribuidos que combinan código nativo, lógica JavaScript, servicios cloud y protocolos de autenticación federada. Esta complejidad, si bien habilita experiencias de usuario sofisticadas, introduce capas adicionales de superficie de ataque que no siempre son evaluadas en conjunto.

El objetivo de esta investigación fue analizar una aplicación Android real —**Arcade Jobs**, disponible públicamente en Play Store— con el propósito de documentar su arquitectura, comprender cómo gestiona autenticación y sesiones, e identificar áreas donde patrones de diseño conocidos podrían derivar en vulnerabilidades si no son implementados con los controles adecuados.

Esta es la **Etapa 1** de una investigación en curso. Su alcance es descriptivo: no se buscó comprometer ningún sistema, acceder a datos de usuarios, ni explotar ningún hallazgo identificado.

---

## 2. Metodología

La investigación siguió un proceso estructurado en fases progresivas, combinando análisis estático con observación dinámica controlada.

### 2.1 Análisis estático

La primera fase consistió en la obtención y decompilación del paquete de aplicación distribuido públicamente. Se procesaron sus componentes para reconstruir la estructura de clases, recursos, configuración de manifiestos y bibliotecas incluidas. Esta fase no implicó modificación alguna del sistema objetivo.

### 2.2 Revisión arquitectónica

A partir del análisis estático, se documentaron los componentes expuestos públicamente, los esquemas de comunicación entre capas, los mecanismos de autenticación declarados y la infraestructura cloud referenciada en el código.

### 2.3 Observación dinámica

La fase dinámica se limitó a observar el comportamiento de la aplicación ante entradas legítimas variadas, monitorear componentes activos y confirmar qué partes del sistema procesaban determinados tipos de interacción. No se realizaron ataques, inyecciones maliciosas ni accesos no autorizados a servicios backend.

---

## 3. Stack tecnológico identificado

El análisis reveló una arquitectura híbrida que combina múltiples capas tecnológicas:

| Capa | Tecnologías identificadas |
|------|--------------------------|
| Framework de aplicación | Expo · React Native |
| Autenticación | Firebase Auth · OAuth 2.0 |
| Persistencia cloud | Firestore |
| Configuración remota | Firebase Remote Config |
| Almacenamiento local seguro | SecureStore |
| Comunicación | HTTP/S · WebSockets |
| Navegación | Expo Router · React Navigation |

Este perfil tecnológico es relevante porque implica que la lógica de la aplicación no reside únicamente en el binario nativo Android, sino que se distribuye entre código JavaScript cargado dinámicamente, servicios cloud externos y mecanismos de autenticación federada. Cada una de estas capas introduce su propio modelo de seguridad, y la ausencia de un análisis integral puede dejar puntos ciegos en la postura de seguridad general.

---

## 4. Hallazgos por área de análisis

### 4.1 Configuración de componentes Android

El manifiesto de la aplicación declaró múltiples componentes accesibles desde contextos externos, incluyendo actividades con filtros de intent de categoría `BROWSABLE` y esquemas de deep link personalizados. Esta configuración es técnicamente necesaria para implementar flujos OAuth en mobile, pero requiere controles de validación rigurosos en la capa de recepción.

Se identificó que la actividad principal actúa como punto de entrada centralizado para estos flujos, un patrón que, si no implementa verificación de origen y validación de parámetros, puede ser susceptible a manipulación por aplicaciones de terceros instaladas en el mismo dispositivo.

### 4.2 Flujo de autenticación OAuth

La aplicación implementa autenticación OAuth con redirección hacia un esquema personalizado. Este patrón es estándar en aplicaciones móviles, pero su seguridad depende completamente de la rigurosidad con que se validen los parámetros recibidos en el callback antes de procesarlos.

Durante la observación se constató que el flujo de retorno es procesado por la actividad principal sin evidencia visible de mecanismos de rechazo ante parámetros inesperados. La investigación no llegó a determinar si existe validación en capas internas de la aplicación o en el servidor, por lo que este punto queda como área de análisis pendiente.

### 4.3 Infraestructura TLS

El dominio principal de la aplicación presentó una configuración TLS atípica: el certificado expuesto no correspondía al dominio consultado, generando errores de verificación de hostname. En producción, este tipo de configuración puede indicar infraestructura mal segmentada, uso de balanceadores intermedios sin configuración adecuada, o servicios en transición no completamente migrados.

Aunque no representa una vulnerabilidad directamente explotable en el contexto de esta investigación, evidencia un área que merece revisión desde el equipo de infraestructura.

### 4.4 Integración con Firebase

El código reveló integración con múltiples servicios Firebase: autenticación, base de datos en tiempo real, configuración remota y almacenamiento. La seguridad de estos servicios depende en gran medida de las reglas de acceso configuradas en la consola de Firebase —aspectos que no son visibles desde el análisis del cliente— y del manejo correcto de tokens en el dispositivo.

Se identificaron referencias a mecanismos de persistencia de sesión y tokens de acceso, cuya robustez no fue posible evaluar completamente en esta etapa.

### 4.5 Comunicación en tiempo real

La aplicación incluye infraestructura de comunicación persistente mediante WebSockets, lo que sugiere uso de eventos en tiempo real, notificaciones o sincronización de datos. Es un hallazgo relevante porque los mecanismos de autenticación de canales persistentes suelen seguir modelos distintos a los de las APIs REST, y con frecuencia reciben menos atención en auditorías de seguridad convencionales.

### 4.6 Navegación dinámica

El uso de Expo Router implica que parte de las rutas y pantallas de la aplicación existe únicamente en el bundle JavaScript, sin representación en el manifiesto nativo. Esto dificulta la enumeración completa de la superficie de la aplicación desde análisis estático convencional, y es un factor que debe considerarse en cualquier evaluación de seguridad de apps construidas con este stack.

---

## 5. Superficie de ataque documentada

A partir de los hallazgos anteriores, se describe la superficie de ataque identificada. Esta descripción es de naturaleza arquitectónica — no constituye ni pretende ser una guía de reproducción, sino una caracterización de las áreas que requieren controles de seguridad adicionales.

| Área | Observación | Prioridad de revisión |
|------|-------------|----------------------|
| Validación de callbacks OAuth | Los parámetros recibidos en deep links requieren verificación de integridad y origen | Alta |
| Control de componentes exportados | Las actividades accesibles externamente deben verificar el contexto del llamante | Alta |
| Configuración TLS en dominio principal | Hostname mismatch detectado en certificados de producción | Media |
| Reglas de acceso en Firebase | No evaluables desde el cliente; requieren revisión interna por el equipo | Media |
| Autenticación de canales WebSocket | Modelo de auth independiente del flujo REST; requiere auditoría específica | Media |
| Enumeración de rutas JavaScript | Rutas dinámicas no visibles en el manifiesto nativo | Baja |

---

## 6. Conclusiones

Esta investigación permitió documentar la complejidad arquitectónica de una aplicación Android moderna que combina múltiples tecnologías y capas de abstracción. El hallazgo central no es una vulnerabilidad específica, sino la constatación de que **la superficie de ataque de este tipo de aplicaciones es más amplia y distribuida de lo que sugiere un análisis superficial**.

Las áreas de mayor atención identificadas son el flujo de autenticación OAuth y el manejo de deep links, donde la seguridad depende de validaciones que no son visibles desde el análisis estático del cliente y que requieren revisión del código del servidor para ser evaluadas completamente.

Esta es la primera etapa de una investigación en curso. Las fases siguientes se centrarán en el análisis de la lógica JavaScript del bundle, el comportamiento de los servicios Firebase y los mecanismos de autenticación en canales WebSocket.

---

## 7. Responsible Disclosure & Ética

Esta investigación fue realizada bajo un marco ético estricto:

- ✅ El análisis se limitó a recursos públicamente distribuidos (APK disponible en Play Store)
- ✅ No se accedió a datos de usuarios, bases de datos ni servicios backend
- ✅ No se explotó ninguna de las áreas de riesgo identificadas
- ✅ Los hallazgos son descriptivos y no incluyen información que facilite su reproducción maliciosa
- ✅ Este reporte fue elaborado con intención de contribuir a la seguridad del ecosistema móvil

Si eres parte del equipo de Arcade Jobs: este trabajo existe para señalar, no para atacar. Estamos disponibles para cualquier proceso de divulgación coordinada a través de los canales que el equipo considere apropiados.

---

## 8. Sobre el equipo

<div align="center">

```
╔══════════════════════════════════════╗
║          p i p o p e l a b s        ║
║      security research team          ║
║                                      ║
║  investigación · análisis · reporte  ║
╚══════════════════════════════════════╝
```

*Esta fue la primera etapa de una investigación mucho más grande.*  
*Y probablemente, apenas el inicio de todo.*

</div>

---

<div align="center">
<sub>pipopelabs · mobile security research · vol.01 · etapa 1 de N · responsible disclosure applied</sub>
</div>
