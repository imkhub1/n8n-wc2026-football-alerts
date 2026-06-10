# ⚽ WC2026 Alertas Diarias — WhatsApp

> Notificaciones automáticas por **WhatsApp** sobre partidos de fútbol internacional de selecciones del **Mundial FIFA 2026** — construido con **n8n**, **Twilio** y la API REST de **API-Football**, desplegado en producción sobre **Railway**.

🌐 [Read in English](README.md)

<div align="center">

[![n8n](https://img.shields.io/badge/built%20with-n8n-EA4B71?logo=n8n&logoColor=white)](https://n8n.io)
[![Twilio](https://img.shields.io/badge/WhatsApp-Twilio-F22F46?logo=twilio&logoColor=white)](https://www.twilio.com/whatsapp)
[![Deployed on Railway](https://img.shields.io/badge/deployed-Railway-0B0D0E?logo=railway&logoColor=white)](https://railway.app)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

</div>

## Resumen

Cada día este workflow envía dos resúmenes automáticos por WhatsApp a una lista de destinatarios:

| Hora (CR) | Qué envía |
| :--- | :--- |
| **07:00 — Mañana** | Vista previa de los **partidos de hoy** con horas de inicio (local) |
| **23:00 — Noche** | **Resultados finales** con goleadores, minutos y etiquetas de penal/autogol |

Ambos resúmenes se filtran para incluir solo lo importante: selecciones **clasificadas al Mundial 2026**, en las competiciones de **Mundial** y **Amistosos Internacionales** — categorías juveniles excluidas.

> **Caso de uso:** un grupo de amigos y familia en Costa Rica que quiere un resumen diario limpio, sin spam, del fútbol relevante para el Mundial, directo en WhatsApp — sin app, sin feed, sin ruido.

## Cómo funciona

![Canvas del workflow de n8n — ambos pipelines y sticky notes](docs/screenshots/workflow-canvas.png)

```mermaid
flowchart LR
    subgraph Manana["Pipeline Mañana — 07:00 CR"]
        A1["Schedule Trigger\ncron 0 7 * * *"] --> A2["Partidos de Hoy\nAPI-Football /fixtures"]
        A2 --> A3["Partidos día UTC siguiente\n(solape de zona horaria)"]
        A3 --> A4["Formatear Resumen\nfiltrar + armar mensaje"]
    end
    subgraph Noche["Pipeline Noche — 23:00 CR"]
        B1["Schedule Trigger\ncron 0 23 * * *"] --> B2["Resultados de Hoy"]
        B2 --> B3["Resultados día UTC siguiente"]
        B3 --> B4["Formatear Resultados\nfiltrar + goleadores + dividir"]
    end
    A4 --> S["Enviar WhatsApp\nTwilio"]
    B4 --> S
```

### Decisiones de diseño clave

- **Límites de día correctos por zona horaria.** Costa Rica es `UTC-6`. Un partido a las 8 pm CR cae en el día UTC *siguiente*, así que cada pipeline consulta **ambos** días (hoy y mañana en UTC), deduplica y recorta todo de vuelta al día calendario de Costa Rica con Luxon. Esto evita el clásico bug de "partidos de la noche que desaparecen".
- **Filtrado de negocio en código.** Solo ligas `1` (Mundial) y `10` (Amistosos); solo selecciones del Mundial 2026; equipos juveniles (`U17`, `U20`, `U23`…) excluidos con regex.
- **Resultados enriquecidos.** El resumen nocturno llama al endpoint `/fixtures/events` por cada partido finalizado para listar goleadores con minuto, más etiquetas `(P)` penal y `(OG)` autogol, acreditando correctamente los autogoles al rival.
- **División segura para WhatsApp.** Los días con muchos resultados se dividen en mensajes de ≤ 1500 caracteres para que el proveedor no trunque nada.
- **Envío masivo (fan-out).** Un único mensaje formateado se entrega a todos los destinatarios en una sola ejecución.

## Stack

| Área | Tecnología / Técnica |
| :--- | :--- |
| Orquestación | **n8n** (Schedule, HTTP Request, Code, Twilio) |
| Mensajería | **API de WhatsApp de Twilio** |
| Fuente de datos | **API-Football** (`v3.football.api-sports.io`) |
| Programación | Expresiones cron (`0 7 * * *`, `0 23 * * *`) |
| Lógica | Code nodes en JavaScript · fechas con Luxon · filtros regex · dedup |
| Despliegue | **Railway** (localhost → producción en la nube) |
| Secretos | Variables de entorno + Credenciales de n8n (fuera del código) |

## Requisitos previos

Antes de importar, asegúrate de tener:

1. Una instancia de **n8n** en ejecución (self-hosted, Railway o n8n Cloud).
2. Una API key de **API-Football** — regístrate en [api-football.com](https://www.api-football.com/).
3. Una cuenta de **Twilio** con el remitente de WhatsApp habilitado (el [Sandbox](https://www.twilio.com/console/sms/whatsapp/learn) sirve para pruebas).
4. Números de WhatsApp de los destinatarios en formato **E.164** (ej. `+50688889999`).

## Primeros pasos

### 1. Importar el workflow

```bash
git clone https://github.com/imkhub1/n8n-wc2026-football-alerts.git
cd n8n-wc2026-football-alerts
```

Abre el editor de n8n → menú superior derecho → **Import from File**, luego selecciona [`workflow/wc2026-football-alerts.json`](workflow/wc2026-football-alerts.json).

### 2. Configurar credenciales

Este repositorio se publica con todos los secretos reales eliminados. Reemplaza los siguientes placeholders:

| Placeholder en el JSON | Qué es | Dónde poner el valor real |
| :--- | :--- | :--- |
| `YOUR_API_FOOTBALL_KEY` | API key de API-Football | Header `x-apisports-key` en los nodos HTTP (o una credencial **Header Auth** de n8n) |
| `+10000000001 … 5` | Números de WhatsApp destino | El array `recipients` dentro de los dos nodos **Code** |
| Credencial de Twilio | Account SID / Auth Token | **Credentials → Twilio API** en n8n (mapeada por nombre `Twilio account`) |

Copia [`.env.example`](.env.example) a `.env` y completa tus valores:

```bash
cp .env.example .env
```

> [!TIP]
> En lugar de incrustar la API key directamente en los nodos HTTP, crea una credencial **Header Auth** en n8n y referencíala — así la key nunca queda en el JSON exportado.

### 3. Activar

Activa el workflow. Los dos triggers programados empezarán a dispararse a las 07:00 y 23:00 en tu zona horaria configurada.

## Despliegue: localhost → Railway

Este proyecto comenzó en **localhost** para desarrollo y luego fue promovido a un **despliegue de producción en [Railway](https://railway.app)** para correr 24/7.

### ¿Por qué Railway?

- Despliegue de n8n con **volumen persistente** y **PostgreSQL** administrado.
- Un **dominio HTTPS público** desde el inicio — necesario para el editor, los webhooks y los callbacks OAuth de n8n.
- Gestión simple de variables de entorno para los secretos.
- Económico y siempre activo, ideal para una automatización personal en un cron diario.

### Local vs. producción

| Aspecto | Localhost (dev) | Railway (producción) |
| :--- | :--- | :--- |
| Base de datos | Archivo SQLite | PostgreSQL (administrado, persistente) |
| Acceso público | `localhost:5678` | Dominio HTTPS público (`*.up.railway.app`) |
| Webhooks | no accesibles | `WEBHOOK_URL` apuntando al dominio público |
| Secretos | `.env` en mi máquina | Variables de entorno de Railway |
| Cifrado de credenciales | clave local | `N8N_ENCRYPTION_KEY` estable como env var |
| Zona horaria | default del sistema | `GENERIC_TIMEZONE=America/Costa_Rica` |

### Variables de entorno principales

```env
N8N_HOST=tu-instancia.up.railway.app
N8N_PROTOCOL=https
WEBHOOK_URL=https://tu-instancia.up.railway.app/
GENERIC_TIMEZONE=America/Costa_Rica

DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=...
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=...

N8N_ENCRYPTION_KEY=<clave-aleatoria-estable-32+-chars>
```

> [!WARNING]
> Mantén `N8N_ENCRYPTION_KEY` **estable** entre redeploys — si cambia, n8n ya no puede descifrar las credenciales guardadas.

Ver [`.env.example`](.env.example) para la lista completa y anotada.

## Estructura del repositorio

```
.
├── workflow/
│   └── wc2026-football-alerts.json   # Workflow de n8n importable (sin secretos)
├── docs/
│   ├── screenshots/
│   │   └── workflow-canvas.png       # Captura del canvas de n8n
│   └── README.md                     # Guía de capturas de pantalla
├── .env.example                      # Variables de entorno anotadas
├── .gitignore
├── LICENSE
├── README.md                         # Versión en inglés
└── README.es.md                      # Estás aquí
```

## Autor

Construido por [@imkhub1](https://github.com/imkhub1). Los issues y PRs son bienvenidos.