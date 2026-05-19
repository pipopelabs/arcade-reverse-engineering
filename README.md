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
```

### `security research // mobile // vol.01`

![Status](https://img.shields.io/badge/estado-en_progreso-orange?style=flat-square&logo=statuspage)
![Platform](https://img.shields.io/badge/plataforma-Android-brightgreen?style=flat-square&logo=android)
![Research](https://img.shields.io/badge/tipo-ingeniería_inversa-red?style=flat-square)
![Team](https://img.shields.io/badge/equipo-pipopelabs-black?style=flat-square)

</div>

---

# Arcade: El inicio de todo

> *"Lo que comenzó como una extracción básica de APK terminó convirtiéndose en una investigación mucho más profunda sobre OAuth, deep links, infraestructura cloud, Firebase y manejo de sesiones."*

Este repositorio documenta la **Etapa 1** de una investigación de seguridad ofensiva sobre **Arcade Jobs**, una aplicación Android moderna distribuida en Play Store. El objetivo no fue romper la aplicación de forma inmediata, sino mapear su arquitectura, comprender su superficie de ataque real, e identificar vectores que podrían convertirse en hallazgos de alto impacto.

---

## Índice

- [Contexto](#contexto)
- [Stack tecnológico identificado](#stack-tecnológico-identificado)
- [Fases de investigación](#fases-de-investigación)
- [Hallazgos principales](#hallazgos-principales)
- [Vectores de riesgo identificados](#vectores-de-riesgo-identificados)
- [Responsible Disclosure](#responsible-disclosure)
- [Equipo](#equipo)

---

## Contexto

La aplicación objetivo fue **Arcade Jobs**, desarrollada con tecnologías modernas como Expo, React Native y Firebase. Las preguntas iniciales que guiaron la investigación:

- ¿Cómo está construida internamente?
- ¿Cómo maneja la autenticación?
- ¿Cómo se comunica con sus servicios backend?
- ¿Qué infraestructura cloud utiliza?
- ¿Qué componentes pueden convertirse en vectores reales de ataque?

La investigación evolucionó desde un análisis estático básico hasta una exploración profunda de **OAuth**, **deep links**, **WebSockets**, **Firebase** y **navegación dinámica con Expo Router**.

---

## Stack tecnológico identificado

| Categoría | Tecnología |
|-----------|------------|
| Framework | Expo · React Native |
| Autenticación | Firebase Auth · OAuth 2.0 |
| Base de datos | Firestore |
| Configuración remota | Firebase Remote Config |
| Almacenamiento seguro | SecureStore |
| Networking | OkHttp · WebSockets |
| Navegación | Expo Router · React Navigation |
| Análisis estático | JADX |
| Extracción | ADB |

---

## Fases de investigación

### `FASE_01` — Extracción del APK

Se utilizó **ADB** para identificar el package name real y localizar los APKs instalados en el dispositivo. La aplicación utilizaba **APK Splits**: base, ARM64, recursos regionales y recursos gráficos. Todos los archivos fueron extraídos hacia el entorno Linux de análisis.

```bash
# Identificar package name
adb shell pm list packages | grep arcade

# Localizar APKs instalados
adb shell pm path <package.name>

# Extraer APK al entorno local
adb pull /data/app/<path>/base.apk ./
```

---

### `FASE_02` — Reversing estático con JADX

El APK fue procesado con **JADX** para reconstruir código Java/Kotlin, recursos, manifest y bibliotecas empaquetadas. Esta fase reveló el stack completo y cambió el enfoque: no era una app Android tradicional, sino una **aplicación híbrida** donde gran parte de la lógica vive en JavaScript y servicios cloud externos.

---

### `FASE_03` — Análisis del AndroidManifest

El manifest reveló el primer hallazgo importante:

- Múltiples **componentes exportados públicamente**
- **Deep links personalizados** con scheme `arcadiajobs://`
- Handlers OAuth con categoría `BROWSABLE`
- Actividades accesibles desde otras apps y desde el navegador

El scheme personalizado `arcadiajobs://` indicaba que la app manejaba callbacks de autenticación y redirects OAuth directamente — arquitectura extremadamente sensible ante errores de validación.

---

### `FASE_04` — Infraestructura y TLS

El dominio principal presentó anomalías graves:

- Certificados TLS **inválidos**
- **Hostname mismatch** — el certificado no coincide con el dominio
- Certificados **self-signed** en producción
- Respuestas HTTP inconsistentes

Esto evidencia una operación insegura, posible separación no documentada entre frontend y backend, o servicios parcialmente abandonados.

---

### `FASE_05` — Validación dinámica de deep links

Se enviaron intents directamente hacia la aplicación simulando callbacks OAuth, códigos de autenticación, tokens falsos y parámetros `state` manipulados.

**Resultado:** La aplicación aceptó todos los intents sin errores, sin rechazos y sin validaciones aparentes. Esto confirmó que el attack surface es **real y activo**.

```bash
# Ejemplo de intent simulado hacia el handler OAuth
adb shell am start \
  -a android.intent.action.VIEW \
  -d "arcadiajobs://auth/callback?code=FAKE_CODE&state=INJECTED"
```

---

### `FASE_06` — Firebase, Expo Router y WebSockets

- **Firebase**: Referencias a Auth, Firestore, Remote Config, Push Tokens y mecanismos de persistencia — dependencia crítica del backend cloud.
- **Expo Router**: Rutas que existen únicamente en JavaScript, cargadas dinámicamente, fuera del binario nativo Android.
- **WebSockets**: Infraestructura realtime con `WebSocketReader`, `WebSocketWriter`, validación TLS y compresión — las apps suelen validar bien REST pero fallan en autenticación WebSocket.

---

## Hallazgos principales

```
┌─────────────────────────────────────────────────────────────────┐
│  HALLAZGO                         │  SEVERIDAD   │  ESTADO      │
├───────────────────────────────────┼──────────────┼──────────────┤
│  Deep links sin validación        │  🔴 Alta     │  Confirmado  │
│  scheme arcadiajobs:// expuesto   │  🔴 Alta     │  Confirmado  │
│  MainActivity como único handler  │  🔴 Alta     │  Confirmado  │
│  TLS inválido / hostname mismatch │  🟠 Media    │  Confirmado  │
│  Firebase en superficie lateral   │  🟠 Media    │  En análisis │
│  WebSocket auth sin verificar     │  🔵 Baja     │  Pendiente   │
│  Rutas ocultas en Expo Router     │  🔵 Baja     │  Pendiente   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Vectores de riesgo identificados

Los siguientes escenarios fueron identificados como potenciales vectores de ataque durante esta etapa. Ninguno fue explotado completamente — representan hipótesis fundamentadas para investigación futura:

| Vector | Descripción |
|--------|-------------|
| **OAuth Callback Abuse** | Inyección de callbacks OAuth via deep links sin validación de origin |
| **Intent Hijacking** | Intercepción de intents por apps maliciosas registradas con el mismo scheme |
| **Session Confusion** | Comportamiento `singleTask` en MainActivity puede mezclar estados de sesión |
| **Token Injection** | Parámetros de autenticación aceptados sin verificación de integridad |
| **Open Redirects** | Redirects OAuth no validados contra una whitelist de destinos |
| **Deep Link Manipulation** | Manipulación de rutas internas via URIs arbitrarias |
| **Hidden Route Discovery** | Descubrimiento de rutas internas no documentadas en Expo Router |
| **Firebase Misconfiguration** | Reglas de Firestore o Auth potencialmente mal configuradas |
| **Weak Session Handling** | Gestión de tokens en SecureStore sin validación adicional |
| **WebSocket Auth Issues** | Autenticación de canal WebSocket independiente de la REST API |

---

## Responsible Disclosure

> Esta investigación es de carácter educativo y fue realizada con fines de análisis de seguridad. **Ninguna vulnerabilidad fue explotada activamente** ni se accedió a datos de usuarios reales. Los hallazgos documentados son vectores potenciales identificados durante análisis estático y dinámico controlado.
>
> Si eres el equipo de Arcade Jobs y llegaste hasta aquí: este reporte existe para ayudar, no para dañar. Estamos disponibles para coordinar cualquier proceso de responsible disclosure.

---

## Equipo

<div align="center">

```
╔══════════════════════════════╗
║        p i p o p e l a b s  ║
║   security research team     ║
╚══════════════════════════════╝
```

*Esta fue la primera etapa de una investigación mucho más grande.*  
*Y probablemente, apenas el inicio de todo.*

</div>

---

<div align="center">
<sub>pipopelabs · mobile security research · vol.01 · etapa 1 de N</sub>
</div>
