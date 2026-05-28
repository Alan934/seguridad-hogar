# Sistema de Seguridad Hogar

Sistema de videovigilancia inteligente con detección de personas, reconocimiento facial y notificaciones automáticas al celular. Corre 100% local con Docker.

## Stack

| Componente | Rol |
|---|---|
| **Frigate NVR** | Grabación continua, detección de objetos (YOLOv9), reconocimiento facial |
| **Home Assistant** | Automatizaciones, notificaciones push, dashboard |
| **Eclipse Mosquitto** | Broker MQTT que conecta Frigate con Home Assistant |

## Arquitectura

```
Cámaras IP (RTSP)
       │
       ▼
  ┌─────────┐     MQTT events      ┌────────────────┐
  │ Frigate │ ──────────────────── │ Home Assistant │ ──▶ Notificación Push
  │  (NVR)  │                      │ (Automatización│
  └─────────┘                      └────────────────┘
       │                                   │
       └──────────── Red Docker ───────────┘
                   (Mosquitto MQTT)
```

## Funcionalidades

- **Detección de personas** en 4 cámaras simultáneas a 5 FPS
- **Reconocimiento facial** con modelo local (sin APIs externas)
- **Zonas personalizadas** mapeadas al layout físico del hogar
- **Lógica de notificación diferenciada**:
  - Familiares reconocidos: alerta solo si cruzan el portón de la calle
  - Desconocidos: alerta en cualquier zona mapeada
- **Notificación push** con snapshot del evento al celular
- **Sin falsos positivos por re-entrada**: la automatización reacciona al cruce de fronteras de zona, no a la presencia continua
- **Búsqueda semántica** de grabaciones por descripción en lenguaje natural
- **Reconocimiento de patentes** (LPR) habilitado
- **Detección de aves** como categoría extra

## Estructura del proyecto

```
.
├── docker-compose.yml          # Orquestación de los 3 servicios
├── .env.example                # Variables de entorno requeridas
├── mosquitto.conf              # Configuración del broker MQTT
├── config/
│   └── config.yml              # Configuración de Frigate (cámaras, zonas, modelos)
└── config_ha/
    ├── configuration.yaml      # Configuración base de Home Assistant
    ├── automations.yaml        # Automatización de notificaciones inteligentes
    ├── scripts.yaml
    └── custom_components/
        └── frigate/            # Integración oficial Frigate para HA
```

## Configuración

### 1. Clonar y configurar variables

```bash
git clone https://github.com/TU_USUARIO/seguridad-hogar.git
cd seguridad-hogar
cp .env.example .env
```

Editar `.env` con las URLs RTSP de tus cámaras:

```env
RTSP_CAM1=rtsp://usuario:contraseña@IP_CAMARA:554/Streaming/Channels/101
RTSP_CAM2=rtsp://usuario:contraseña@IP_CAMARA:554/Streaming/Channels/201
RTSP_CAM3=rtsp://usuario:contraseña@IP_CAMARA:554/Streaming/Channels/301
RTSP_CAM4=rtsp://usuario:contraseña@IP_CAMARA:554/Streaming/Channels/401
```

### 2. Levantar los servicios

```bash
docker compose up -d
```

### 3. Acceder a las interfaces

| Servicio | URL |
|---|---|
| Frigate NVR | `http://localhost:5000` |
| Home Assistant | `http://localhost:8123` |
| RTSP re-stream | `rtsp://localhost:8554` |

## Automatización principal

La automatización `Seguridad: Notificaciones Inteligentes por Rostro` en [config_ha/automations.yaml](config_ha/automations.yaml) implementa la lógica central:

1. Escucha todos los eventos MQTT de Frigate en tiempo real
2. Filtra exclusivamente eventos de tipo `person`
3. Compara el estado **anterior vs actual** de zonas y rostro reconocido
4. Aplica reglas diferenciadas según si la persona es familiar o desconocida
5. Envía notificación push con snapshot directo del evento

```
Evento MQTT
    │
    ▼
¿Es persona?  ──No──▶ Ignorar
    │
   Sí
    │
    ▼
¿Cambió de zona o se reconoció el rostro?  ──No──▶ Ignorar
    │
   Sí
    │
    ▼
¿Es familiar?
  ├── Sí ──▶ ¿Está en portón calle?  ──No──▶ Ignorar
  │                │ Sí
  │                ▼
  │           Notificar "Familiar cruzando"
  │
  └── No ──▶ ¿Está en alguna zona mapeada?  ──No──▶ Ignorar
                      │ Sí
                      ▼
                Notificar "Desconocido en movimiento"
```

## Zonas configuradas

Las zonas se definen en `config/config.yml` según el layout físico del hogar:

- `zona_porton_calle` — entrada principal desde la calle
- `zona_hacia_puerta_trasera` — corredor hacia la parte trasera
- `zona_hacia_chancha` — lateral del terreno
- `zona_hacia_callejon` — acceso al callejón
- `zona_patio_general` — patio interior

## Requisitos

- Docker y Docker Compose
- GPU opcional (Frigate puede usar CPU, Coral TPU o GPU NVIDIA/AMD)
- Cámaras IP con stream RTSP
- App Home Assistant en el celular para notificaciones push

## Notas de seguridad

- Las credenciales RTSP van en `.env` (nunca en el repositorio)
- Frigate y Home Assistant corren en una red Docker aislada (`seguridad_network`)
- No se expone ningún puerto al exterior por defecto; todos son `localhost`
