# 📚 Karakeep Docker - Gestor de Marcapáginas Inteligente con IA

[![GitHub Stars](https://img.shields.io/github/stars/karakeep-app/karakeep?style=for-the-badge&logo=github)](https://github.com/karakeep-app/karakeep)
[![Docker Pulls](https://img.shields.io/docker/pulls/karakeep/karakeep?style=for-the-badge&logo=docker)](https://hub.docker.com/r/karakeep/karakeep)
[![License](https://img.shields.io/github/license/karakeep-app/karakeep?style=for-the-badge)](https://github.com/karakeep-app/karakeep/blob/main/LICENSE)
[![Discord](https://img.shields.io/discord/115000000000000000?style=for-the-badge&logo=discord&label=Discord)](https://discord.gg/NrgeYywsFh)

## 📋 Descripción general

**Karakeep** (anteriormente Hoarder) es una aplicación de código abierto diseñada para autohospedarse que permite guardar, organizar y buscar todo tipo de contenido: enlaces, notas, imágenes y archivos PDF. Su diferencial principal es la integración de modelos de lenguaje (LLM) para etiquetado y resumen automático, con soporte para modelos locales mediante Ollama, además de búsqueda full-text y semántica impulsada por Meilisearch.

El proyecto está licenciado bajo **AGPL-3.0**, desarrollado en **TypeScript** con **Next.js (App Router)**, **Drizzle ORM**, **NextAuth**, **tRPC** y **Puppeteer**, y cuenta con clientes nativos para iOS, Android y extensiones para Chrome, Firefox y Safari.

> 🎥 **Vídeo tutorial**: [Cómo instalar Karakeep en Docker](https://www.youtube.com/@GENBYTE) en el canal GENBYTE

## ✨ Características principales

- **Guardado universal de contenido**: Almacena enlaces, notas simples, imágenes y PDFs con obtención automática de título, descripción e imagen
- **Organización y colaboración**: Marcadores agrupados en listas compartibles para trabajo en equipo
- **Búsqueda full-text y semántica**: Indexación completa mediante Meilisearch para búsquedas por texto y semánticas
- **Etiquetado y resumen automático con IA**: LLM (OpenAI o modelos locales vía Ollama) para generar etiquetas y resúmenes
- **Agentes LLM y CLI**: Compatible con OpenClaw, Hermes, CLI oficial y skills para automatización avanzada
- **Motor de reglas personalizable**: Reglas para gestión automática del contenido según criterios configurables
- **OCR en imágenes**: Extracción de texto de imágenes para hacerlo buscable
- **Clientes multiplataforma**: Extensiones Chrome/Firefox/Safari + apps nativas iOS/Android con lectura offline
- **Automatización por RSS**: Importación automática desde feeds RSS configurados
- **API REST y clientes múltiples**: API expuesta para integraciones y desarrollo propio
- **Soporte multiidioma**: Interfaz traducida gestionada mediante Weblate
- **Resaltado y anotaciones**: Marcado y guardado de fragmentos destacados
- **Archivado completo de páginas**: Monolith para copias completas contra link rot
- **Archivado de vídeo**: Descarga y almacenamiento automático mediante yt-dlp
- **Acciones en lote**: Operaciones masivas sobre múltiples elementos
- **Autenticación SSO**: Integración con proveedores de identidad mediante Single Sign-On
- **Modo oscuro**: Interfaz adaptable a preferencias de tema
- **Importadores**: Migración desde Chrome, Pocket, Linkwarden, Omnivore, Tab Session Manager
- **Sincronización navegadores**: Sincroniza marcadores mediante Floccus

## 📋 Requisitos del sistema

- **Docker** y **Docker Compose** (método recomendado)
- **Node.js 20+** (para desarrollo local, según `.nvmrc`)
- **pnpm 11.2.1** (gestor de paquetes del monorepo)
- **Base de datos PostgreSQL** (gestionada por Drizzle ORM)
- **Meilisearch** para motor de búsqueda
- **Opcional**: Ollama para modelos LLM locales
- **Opcional**: GPU para acelerar inferencia local y OCR (mínimo 8 GB RAM para modelos 7B)

## 🐳 Instalación

### Clonar repositorio y configurar entorno

```bash
git clone https://github.com/karakeep-app/karakeep.git
cd karakeep
cp .env.sample .env
# Edite .env con sus configuraciones (ver sección Configuración)
```

### Docker Compose completo (recomendado)

La configuración oficial incluye todos los servicios: aplicación web, workers, PostgreSQL, Meilisearch y opcionalmente Ollama.

```yaml
# docker-compose.yml (versión simplificada - consulte docs.karakeep.app para completa)
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: karakeep
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: karakeep
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U karakeep"]
      interval: 10s
      timeout: 5s
      retries: 5

  meilisearch:
    image: getmeili/meilisearch:v1.11
    environment:
      MEILI_MASTER_KEY: ${MEILISEARCH_API_KEY}
    volumes:
      - meilisearch_data:/meili_data
    ports:
      - "7700:7700"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:7700/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    image: karakeep/karakeep:latest
    depends_on:
      postgres:
        condition: service_healthy
      meilisearch:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://karakeep:${POSTGRES_PASSWORD}@postgres:5432/karakeep
      MEILISEARCH_HOST: http://meilisearch:7700
      MEILISEARCH_API_KEY: ${MEILISEARCH_API_KEY}
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      NEXTAUTH_URL: ${NEXTAUTH_URL}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      OLLAMA_BASE_URL: ${OLLAMA_BASE_URL}
    ports:
      - "3000:3000"
    volumes:
      - karakeep_data:/app/data

  workers:
    image: karakeep/karakeep:latest
    command: pnpm workers
    depends_on:
      postgres:
        condition: service_healthy
      meilisearch:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://karakeep:${POSTGRES_PASSWORD}@postgres:5432/karakeep
      MEILISEARCH_HOST: http://meilisearch:7700
      MEILISEARCH_API_KEY: ${MEILISEARCH_API_KEY}
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      OLLAMA_BASE_URL: ${OLLAMA_BASE_URL}
    volumes:
      - karakeep_data:/app/data

  # Opcional: Ollama para modelos locales
  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

volumes:
  postgres_data:
  meilisearch_data:
  karakeep_data:
  ollama_data:
```

### Levantar servicios

```bash
docker compose up -d
```

## ⚙️ Configuración

1. **Base de datos**: Configure `DATABASE_URL` con conexión PostgreSQL (usuario, contraseña, host, puerto, base de datos)
2. **Meilisearch**: Defina `MEILISEARCH_HOST` (ej. `http://meilisearch:7700`) y `MEILISEARCH_API_KEY` (clave maestra segura)
3. **Autenticación**: Genere `NEXTAUTH_SECRET` (clave larga aleatoria, `openssl rand -base64 32`) y establezca `NEXTAUTH_URL` (URL pública HTTPS)
4. **IA - OpenAI**: Añada `OPENAI_API_KEY` si usa modelos de OpenAI
5. **IA - Ollama local**: Configure `OLLAMA_BASE_URL` (ej. `http://ollama:11434`) y opcional `DEFAULT_OLLAMA_MODEL` (ej. `llama3.1:8b`)
6. **Almacenamiento**: Configure variables de almacenamiento de archivos según documentación oficial
7. **Correo**: Configure SMTP para notificaciones y recuperación de contraseña
8. **Características opcionales**: Active/desactive funcionalidades según necesidades (OCR, archivado vídeo, etc.)

## 🚀 Primeros pasos

1. **Clonar y configurar**: Ejecute los comandos de instalación y edite `.env` con sus valores
2. **Levantar contenedores**: `docker compose up -d` y verifique logs con `docker compose logs -f`
3. **Acceder a la interfaz**: Abra `http://localhost:3000` (o su `NEXTAUTH_URL`)
4. **Crear usuario administrador**: El primer usuario registrado se convierte en admin automáticamente
5. **Configurar IA (opcional)**: En ajustes, seleccione proveedor LLM (OpenAI u Ollama) y modelo deseado
6. **Instalar extensiones**: Añada extensión de navegador y configure URL de su instancia
7. **Configurar apps móviles**: Descargue app iOS/Android e introduzca URL de su instancia
8. **Importar datos**: Use importadores desde Chrome, Pocket, Linkwarden, Omnivore o Floccus

## 💡 Casos de uso

- **Gestión personal de conocimiento** tipo "read-it-later" con archivado permanente
- **Investigación y recopilación** de referencias con etiquetado automático por IA
- **Equipos colaborativos** que comparten colecciones de enlaces y documentos
- **Archivado legal/cumplimiento** de páginas web y vídeos completos
- **Sustitución autohospedada** de Pocket, Raindrop.io o MyMind con control total de datos
- **Integración en flujos de trabajo** con agentes LLM mediante CLI y API
- **Sincronización unificada** de marcadores entre navegadores y dispositivos móviles

## 🔒 Acceso remoto seguro

Para exponer Karakeep en internet use un **proxy inverso** (Nginx, Traefik, Caddy) que termine TLS y reenvíe al puerto 3000.

### Ejemplo con Caddy

```caddyfile
karakeep.tu-dominio.com {
    reverse_proxy web:3000
    header {
        X-Forwarded-Proto "https"
        X-Forwarded-Host {host}
    }
}
```

**Importante**: Configure `NEXTAUTH_URL` con la URL pública HTTPS y asegúrese de que el proxy pase cabeceras `X-Forwarded-*` para que NextAuth funcione correctamente.

## 🛠️ Gestión y mantenimiento

### Actualización de versión

```bash
docker compose pull
docker compose up -d
# Ejecutar migraciones si la versión lo requiere
docker compose exec web pnpm db:migrate
```

> Revise el [changelog en GitHub Releases](https://github.com/karakeep-app/karakeep/releases) para cambios breaking.

### Copia de seguridad

```bash
# Backup base de datos PostgreSQL
docker compose exec postgres pg_dump -U karakeep karakeep > backup_$(date +%F).sql

# Backup volúmenes (Meilisearch, almacenamiento)
docker run --rm -v karakeep_meilisearch_data:/data -v $(pwd):/backup alpine tar czf /backup/meilisearch_$(date +%F).tar.gz -C /data .
docker run --rm -v karakeep_karakeep_data:/data -v $(pwd):/backup alpine tar czf /backup/karakeep_data_$(date +%F).tar.gz -C /data .
```

> Versionar también archivo `.env` y configuración de proxy inverso.

### Limpieza y reindexado

```bash
# Ver logs de workers (procesan colas: fetching, OCR, IA)
docker compose logs -f workers

# Reindexar Meilisearch (ejemplo vía API)
curl -X POST 'http://localhost:7700/indexes/bookmarks/documents/fetch' \
  -H 'Authorization: Bearer masterKey'
```

## 📝 Licencia

Este proyecto está licenciado bajo **AGPL-3.0** - ver archivo [LICENSE](https://github.com/karakeep-app/karakeep/blob/main/LICENSE) para detalles.

---

> 📖 **Artículo original**: [Cómo instalar Karakeep en Docker: Gestor de Marcapáginas Inteligente con IA](https://genbyte.blogspot.com/2026/10/karakeep-gestor-de-marcapaginas.html)  
> 🎬 **Canal YouTube**: [GENBYTE](https://www.youtube.com/@GENBYTE)  
> 💬 **Comunidad Discord**: [discord.gg/NrgeYywsFh](https://discord.gg/NrgeYywsFh)  
> ☕ **Apoya el canal**: [Ko-fi](https://ko-fi.com/genbyte) | [Newsletter](https://genbyte.blogspot.com/)