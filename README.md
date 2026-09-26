# Nube · Frontend Vue

Interfaz de inicio de sesión en Vue 3 + Vite. El backend independiente está en `../vue-login-backend`.

## Desarrollo local

1. En `../vue-login-backend`, ejecuta `npm install`, configura `.env` a partir de `.env.example` y corre `npm run dev`.
2. En este proyecto, ejecuta `npm install` y `npm run dev`.

Vite reenvía `/api` a `http://localhost:3000` durante el desarrollo. Ese proxy no se incluye en `dist`.

## EC2 pública y backend privado

Mantén `VITE_API_URL=/api` en `.env`. El navegador llama al mismo origen que sirve Vue y el servidor público reenvía las peticiones al backend privado `10.20.1.162:3000`. No uses la IP privada como URL de Vite: el navegador del usuario no tiene acceso a esa red.

### Si el servidor del frontend es Nginx

1. Desde la EC2 pública, verifica conectividad:

   ```sh
   curl -i --connect-timeout 5 http://10.20.1.162:3000/api/health
   ```

2. Copia `deploy/nginx-api.conf` a `/etc/nginx/snippets/vue-login-api.conf` (crea el directorio si no existe). Añade esta línea **dentro del bloque `server` existente que sirve el frontend**:

   ```nginx
   include /etc/nginx/snippets/vue-login-api.conf;
   ```

   Si ya existe un `location /api/`, reemplázalo con el del archivo. Si el sitio usa HTTPS, incluye la configuración en su bloque HTTPS. Conserva el dominio, certificados y la ruta `root` existentes. Para navegación de Vue, el bloque de archivos estáticos puede usar:

   ```nginx
   location / {
       try_files $uri $uri/ /index.html;
   }
   ```

   `proxy_pass` no lleva barra final para conservar `/api/auth/login`, `/api/auth/register` y `/api/auth/me`.

3. Valida y recarga:

   ```sh
   sudo nginx -t && sudo systemctl reload nginx
   ```

4. Abre `/api/health` en el dominio o IP pública del frontend. Debe devolver JSON. Prueba después el inicio de sesión y el registro.

### Red y backend

- En el security group del backend, permite TCP 3000 desde el security group de la EC2 pública. Verifica que las subredes tengan rutas entre sí y que las reglas de salida y las ACL permitan la comunicación y el tráfico de retorno.
- La API debe estar iniciada y escuchar en una interfaz accesible desde la VPC, no solo en `127.0.0.1`. El backend actual usa `app.listen(port)`, sin restringirse a localhost. Comprueba en la EC2 privada con `ss -lntp | grep :3000`.
- `/api/health` también verifica PostgreSQL: una respuesta HTTP 500 requiere revisar la API/base de datos; un timeout apunta a conectividad y un rechazo de conexión a un servicio que no escucha o un firewall.
- No hace falta abrir el puerto 3000 a Internet. Con `/api` en el mismo origen, el navegador no necesita CORS entre el frontend y la IP privada.
- Un 502 en Nginx requiere revisar conectividad y `/var/log/nginx/error.log`. Si `/api/health` devuelve HTML, la petición está llegando al fallback de Vue: revisa el bloque `server` y el `location` activos.

Si cambias `VITE_API_URL`, ejecuta `npm run build` y vuelve a subir el contenido de `dist`; Vite incorpora esa variable al compilar. Si el `dist` desplegado ya usa `/api`, basta con configurar el proxy del servidor.
