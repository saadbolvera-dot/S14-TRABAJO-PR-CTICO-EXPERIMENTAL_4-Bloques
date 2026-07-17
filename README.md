# Gestor de Tareas

Sistema web de gestión de tareas — proyecto académico. Backend REST API con Node.js, Express,
MongoDB y autenticación JWT + OAuth con Google, y frontend en React con Vite y Tailwind CSS.

## Estructura

- `server/` — API REST (Express + MongoDB + JWT)
- `client/` — Frontend React (Vite + Tailwind)
- `docs/superpowers/specs/` — documentos de diseño
- `docs/superpowers/plans/` — planes de implementación

## Backend — cómo correrlo

1. `cd server`
2. `npm install`
3. Copiar `.env.example` a `.env` y completar `MONGO_URI` (tu cluster de MongoDB Atlas) y `JWT_SECRET`.
   Para el login con Google completar además `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`,
   `GOOGLE_CALLBACK_URL` y `CLIENT_URL` (ver sección [OAuth con Google](#autenticación-con-google-oauth)).
4. `npm start` (o `npm run dev` para reinicio automático al guardar cambios)
5. La API queda disponible en `http://localhost:5000/api`

### Datos de ejemplo (seed)

`npm run seed` crea (o recrea) un usuario de demostración con tareas de ejemplo, sin tocar otras
cuentas ya existentes en la base de datos:

- Usuario: `demo@gestortareas.com` / `demo123456`
- 8 tareas de ejemplo repartidas entre los tres estados y las tres prioridades

## Endpoints

### Auth (`/api/auth`)
| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| POST | `/register` | No | Registra un usuario, devuelve JWT |
| POST | `/login` | No | Inicia sesión, devuelve JWT |
| GET | `/me` | Sí | Devuelve el usuario autenticado |
| GET | `/google` | No | Redirige a Google para iniciar el flujo OAuth |
| GET | `/google/callback` | No | Callback de Google; genera un JWT y redirige al frontend |

### Tasks (`/api/tasks`) — todas requieren `Authorization: Bearer <token>`
| Método | Ruta | Descripción |
|---|---|---|
| GET | `/` | Lista las tareas del usuario (filtros opcionales `?status=` y `?priority=`) |
| POST | `/` | Crea una tarea |
| GET | `/:id` | Obtiene una tarea propia |
| PUT | `/:id` | Actualiza una tarea propia |
| DELETE | `/:id` | Elimina una tarea propia |

## Frontend — cómo correrlo

1. `cd client`
2. `npm install`
3. Copiar `.env.example` a `.env` (por defecto ya apunta a `http://localhost:5000/api`).
4. `npm run dev`
5. Abrir `http://localhost:5173` (requiere que el backend también esté corriendo)

### Páginas

| Ruta | Descripción |
|---|---|
| `/login` | Inicio de sesión (email/contraseña o botón "Continuar con Google") |
| `/register` | Registro de usuario |
| `/oauth-callback` | Recibe el token tras volver de Google y completa el login |
| `/` | Dashboard: contadores y gráfica de distribución por estado, filtros por estado/prioridad, listado de tareas, crear/editar tareas en un modal |

## Postman

Importar `server/postman/GestorTareas.postman_collection.json` en Postman. Ejecutar primero
"Login" (o "Register") para que el token se guarde automáticamente en la variable de colección
`token`; el resto de los requests lo usan automáticamente.

El login con Google (`/api/auth/google`) es un flujo basado en redirecciones de navegador, por lo
que no se puede probar desde Postman: se prueba abriendo esa URL directamente en el navegador.

## Autenticación

- El logout es solo del lado del cliente: se descarta el token guardado, no hay endpoint de
  servidor para esto.
- El token JWT expira a las 24 horas.
- Ambos métodos de login (email/contraseña y Google) emiten el mismo tipo de JWT, así que el resto
  de la API (rutas de tareas, middleware `auth`) no distingue cómo se autenticó el usuario.

## Autenticación con Google (OAuth)

1. En [Google Cloud Console](https://console.cloud.google.com/apis/credentials) crear credenciales
   OAuth 2.0 de tipo "Aplicación web".
2. Agregar como "Orígenes de JavaScript autorizados" la URL del frontend (ej.
   `http://localhost:5173`) y como "URI de redireccionamiento autorizados" la URL del callback del
   backend (ej. `http://localhost:5000/api/auth/google/callback`).
3. Copiar el Client ID y Client Secret al `.env` del servidor (`GOOGLE_CLIENT_ID`,
   `GOOGLE_CLIENT_SECRET`), y completar `GOOGLE_CALLBACK_URL` y `CLIENT_URL` (URL del frontend a la
   que se redirige tras el login).
4. Si el login falla durante el callback, la app ahora redirige al frontend en
   `${CLIENT_URL}/login` en lugar de `/login` del backend.
5. Flujo: el botón "Continuar con Google" del login redirige a `GET /api/auth/google` → Google →
   `GET /api/auth/google/callback` → el backend crea/vincula el usuario (por `googleId` o por
   email si ya existía una cuenta local con ese email), genera un JWT y redirige a
   `${CLIENT_URL}/oauth-callback?token=...` → el frontend toma el token, llama a `/api/auth/me` y
   completa la sesión igual que con el login tradicional.
6. En producción, actualizar los orígenes/URIs autorizados en Google Cloud Console y las variables
   `GOOGLE_CALLBACK_URL`/`CLIENT_URL` con las URLs desplegadas.

## Despliegue

### Backend en Render

1. Crear un nuevo "Web Service" en [Render](https://render.com) apuntando a este repositorio
   (usa `server/render.yaml` como Blueprint, o configurar manualmente: Root Directory `server`,
   Build Command `npm install`, Start Command `npm start`).
2. Configurar las variables de entorno del servicio con los mismos valores que `server/.env.example`
   (`MONGO_URI`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `CLIENT_ORIGIN`, `CLIENT_URL`, `GOOGLE_CLIENT_ID`,
   `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`).
3. `CLIENT_ORIGIN` debe ser la URL del frontend en Vercel (para que el CORS lo permita) y
   `GOOGLE_CALLBACK_URL` debe apuntar a `https://<tu-servicio>.onrender.com/api/auth/google/callback`.

También se incluye `server/Procfile` por si se prefiere desplegar en Heroku en lugar de Render.

### Frontend en Vercel

1. Crear un nuevo proyecto en [Vercel](https://vercel.com) apuntando a este repositorio con Root
   Directory `client` (Vercel detecta Vite automáticamente: Build Command `npm run build`, Output
   Directory `dist`).
2. Configurar la variable de entorno `VITE_API_URL` con la URL del backend desplegado (ej.
   `https://<tu-servicio>.onrender.com/api`).
3. `client/vercel.json` ya incluye el rewrite necesario para que las rutas de React Router (como
   `/oauth-callback`) funcionen correctamente al recargar la página.

Tras desplegar ambos, actualizar en el backend `CLIENT_ORIGIN`/`CLIENT_URL` con la URL final de
Vercel y en Google Cloud Console los orígenes/redirect URIs, y volver a desplegar.
