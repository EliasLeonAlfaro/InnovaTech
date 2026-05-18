# ==========================================
# ETAPA 1: Construcción (Build)
# ==========================================
FROM node:18-alpine AS build
WORKDIR /app

# Copiar archivos de dependencias para aprovechar la caché de Docker
COPY package*.json ./
RUN npm ci

# Copiar el resto del código y compilar
COPY . .
RUN npm run build

# ==========================================
# ETAPA 2: Producción (Entorno Seguro con Nginx)
# ==========================================
FROM nginx:1.25-alpine
WORKDIR /usr/share/nginx/html

# RÚBRICA (IE1, IE6): Crear un usuario y grupo "no-root" por seguridad
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Copiar los archivos estáticos compilados desde la etapa anterior
COPY --from=build /app/dist .

# Configurar permisos para que Nginx pueda correr sin privilegios de root
RUN chown -R appuser:appgroup /usr/share/nginx/html && \
    chown -R appuser:appgroup /var/cache/nginx && \
    chown -R appuser:appgroup /var/log/nginx && \
    chown -R appuser:appgroup /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R appuser:appgroup /var/run/nginx.pid

# Cambiar al usuario seguro antes de ejecutar
USER appuser

# Puerto expuesto estándar HTTP
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]