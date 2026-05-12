# django-stripe-skill

> **Skill para Claude / OpenCode** — conocimiento experto sobre sistemas de pago con Django, dj-stripe y Stripe. Diseñado para que los agentes IA construyan integraciones de pago robustas, idempotentes y preparadas para producción.

<br>

<p align="center">
  <em>Stripe + Django + dj-stripe — desde la arquitectura inicial hasta el Go-Live en producción</em>
</p>

<br>

---

## Aviso

**Este es un skill comunitario no-oficial.** No está afiliado, respaldado ni mantenido por Stripe, Inc. ni por los maintainers de [dj-stripe](https://github.com/dj-stripe/dj-stripe). Es una contribución independiente para el ecosistema de agentes IA.

---

## ¿Qué es este proyecto?

Este repositorio contiene un **skill de agente IA** que permite a Claude, OpenCode y agentes compatibles diseñar, construir, depurar y mantener sistemas de pago con Django + dj-stripe de forma precisa y alineada con las mejores prácticas de Stripe.

Cubre **dj-stripe 2.x** con Django >= 5.1, Python >= 3.11 y PostgreSQL >= 12.

---

## ¿Qué es dj-stripe?

[dj-stripe](https://github.com/dj-stripe/dj-stripe) es la librería estándar para integrar Stripe con Django. Sincroniza automáticamente los objetos de Stripe (Customers, Charges, Subscriptions, Invoices, PaymentIntents...) en tu base de datos local y despacha eventos de webhook mediante signal receivers de Django.

- **Escribe** en la API de Stripe (crear customers, charges, subscriptions)
- **Lee** de tu base de datos local (consultas rápidas sin latencia de API)
- **Sincroniza** vía webhooks (Stripe empuja los cambios a tu app)

---

## Estructura del skill

El skill utiliza una arquitectura de **divulgación progresiva** en tres niveles:

### Nivel 1: Metadata (siempre en contexto)
- `SKILL.md` — nombre, descripción y compatibilidad

### Nivel 2: Cuerpo del skill (cuando se activa)
- `SKILL.md` — reglas críticas, referencias rápidas, errores comunes, red flags

### Nivel 3: Recursos empaquetados (bajo demanda)
📁 `references/` — 11 archivos de referencia especializados:

| Archivo | Contenido |
|---|---|
| `architecture.md` | Estructura de proyecto, patrones de diseño, capa de servicios, Stripe como source of truth |
| `installation.md` | Instalación, configuración de settings, INSTALLED_APPS, URLs, API keys, base de datos |
| `models.md` | Catálogo completo de modelos dj-stripe, campos, relaciones, stripe_data JSONField |
| `webhooks.md` | Arquitectura de webhooks, handlers idempotentes, `@djstripe_receiver`, procesamiento de errores, webhook testing |
| `subscriptions.md` | Gestión de suscripciones, ciclo de vida, facturación recurrente, proration, cancelación |
| `payments.md` | Checkout Sessions, PaymentIntents, SetupIntents, Payment Methods, flujos de pago completos |
| `sync-and-data.md` | Sincronización inicial y continua, comandos de gestión, consultas sobre stripe_data |
| `security.md` | API keys, claves restringidas, PCI compliance, verificación de firmas de webhook, encriptación |
| `testing.md` | Testing unitario, integración, webhooks, fixtures, CI/CD, pytest con Stripe test mode |
| `migrations.md` | Guía de upgrade entre versiones de dj-stripe, matriz de compatibilidad, pasos secuenciales |
| `stripe-api.md` | API de Stripe, versionado, Connect, productos y precios, patrones de integración directa |

---

## Cómo usar este skill

### En Claude Code / OpenCode

Coloca este repositorio en el directorio de skills de tu agente:

```bash
git clone https://github.com/ijmadalena/django-stripe-skill.git ~/.agents/skills/django-stripe-skill
```

El skill se activa automáticamente cuando el agente detecta referencias a Stripe, dj-stripe, pagos, suscripciones, webhooks de pago, o cualquiera de los triggers definidos en la descripción del skill.

### Instalación desde skills-registry (si está publicado)

```bash
# En Claude Code
/install ijmadalena/django-stripe-skill
```

---

## Qué cubre el skill

- **Arquitectura y diseño** — estructura de proyecto, patrones, capa de servicios
- **Instalación y configuración** — desde `pip install` hasta `STRIPE_LIVE_MODE = True`
- **Modelos y relaciones** — catálogo completo, `stripe_data` JSONField, FK linkage con User
- **Webhooks** — handlers idempotentes, verificación de firmas, reintentos, idempotency keys
- **Suscripciones** — ciclo de vida completo, trial, proration, cancelación, invoices
- **Pagos** — Checkout Sessions (hosted), Payment Intents (custom UI), Setup Intents
- **Sincronización de datos** — sync inicial, sync por modelo, reprocesamiento de eventos fallidos
- **Seguridad** — restricted API keys, PCI compliance, webhook signatures, key rotation
- **Testing** — unitario, integración, webhooks, fixtures, CI/CD
- **Migraciones** — upgrade paths secuenciales, breaking changes por versión
- **Stripe API directa** — versionado, Connect, productos/precios, patrones de integración
- **Errores comunes** — red flags detectados en testing de baseline con agentes IA

---

## No cubierto por este skill

- **Stripe.js / Elements** — este skill se enfoca en el backend Django; la implementación del frontend JavaScript queda fuera del scope
- **Stripe Sigma / Data Pipeline** — herramientas de analytics de Stripe
- **Stripe Terminal** — pagos presenciales con hardware
- **Stripe Atlas** — constitución de empresas

---

## Desarrollo del skill

Este skill fue desarrollado siguiendo la metodología **Test-Driven Development para documentación** del [Skill Creator](https://github.com/anthropics/skills):

1. Definición de escenarios de presión con subagentes
2. Baseline testing — documentar fallos sin el skill presente
3. Escritura del skill abordando los fallos específicos
4. Re-verificación — los agentes cumplen con las reglas
5. Refactor — cierre de loopholes identificados en testing

### Dependencias del proyecto

Este skill no tiene dependencias en tiempo de ejecución — es pura documentación. El proyecto real que lo use necesitará:

- Python >= 3.11
- Django >= 5.1
- dj-stripe >= 2.9
- PostgreSQL >= 12

---

## Licencia

GNU AGPL v3.0 — véase [LICENSE](LICENSE)

---

## Créditos

- [dj-stripe](https://github.com/dj-stripe/dj-stripe) mantenido por la comunidad dj-stripe
- [Stripe](https://stripe.com/docs/api) — documentación oficial de la API de Stripe
- Este skill es una contribución comunitaria independiente, **no-oficial**, mantenida para el ecosistema de agentes IA
