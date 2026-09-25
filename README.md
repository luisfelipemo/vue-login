# Nube · Frontend Vue

Interfaz de inicio de sesión en Vue 3 + Vite. El backend independiente está en `../vue-login-backend`.

## Desarrollo local

1. En `../vue-login-backend`, ejecuta `npm install`, configura `.env` a partir de `.env.example` y corre `npm run dev`.
2. En este proyecto, ejecuta `npm install` y `npm run dev`.

Vite reenvía `/api` a `http://localhost:3000` durante el desarrollo. Para producción u otro host, crea `.env` desde `.env.example` y configura `VITE_API_URL` con la URL base de la API, por ejemplo `https://api.ejemplo.com/api`.
