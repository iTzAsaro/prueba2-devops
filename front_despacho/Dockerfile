# Etapa 1: build (compilación del frontend con dependencias de desarrollo)
FROM node:20.12.2-alpine3.19 AS build
WORKDIR /app

# Usuario no-root (principio de mínimo privilegio)
RUN addgroup -S app && adduser -S -G app -u 10001 app

# Instala dependencias de forma reproducible usando el lockfile
COPY package.json package-lock.json ./
RUN npm ci && npm cache clean --force


# Copia el código y compila el bundle estático (dist/)
COPY . .


# Variable de build para Vite (se “hornea” en el bundle durante la compilación)
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL


# Limpieza: elimina dependencias/cachés para reducir tamaño de la etapa (y por extensión, del contexto)
RUN npm run build && rm -rf node_modules /root/.npm /tmp/*


# Etapa 2: runtime (servidor estático mínimo para producción)
FROM nginx:1.27.0-alpine3.19 AS runtime
# Usuario no-root + endurecimiento de Nginx:
# - borra html por defecto
# - configura Nginx para escuchar en 8080 (puerto no privilegiado) y servir SPA
# - deshabilita server_tokens (menos información expuesta)
# - ajusta permisos para que Nginx corra como usuario sin privilegios


RUN addgroup -S app && adduser -S -G app -u 10001 app && \
    rm -rf /usr/share/nginx/html/* && \
    sed -i 's/^user .*/user app;/' /etc/nginx/nginx.conf && \
    rm -f /etc/nginx/conf.d/default.conf && \
    mkdir -p /etc/nginx/templates && \
    printf 'server {\n  listen 8080;\n  server_tokens off;\n  root /usr/share/nginx/html;\n  index index.html;\n  location /api/v1/ventas {\n    proxy_pass http://${VENTAS_UPSTREAM};\n    proxy_http_version 1.1;\n    proxy_set_header Host $host;\n    proxy_set_header X-Real-IP $remote_addr;\n    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;\n    proxy_set_header X-Forwarded-Proto $scheme;\n  }\n  location /api/v1/despachos {\n    proxy_pass http://${DESPACHOS_UPSTREAM};\n    proxy_http_version 1.1;\n    proxy_set_header Host $host;\n    proxy_set_header X-Real-IP $remote_addr;\n    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;\n    proxy_set_header X-Forwarded-Proto $scheme;\n  }\n  location /api/ {\n    proxy_pass http://${DESPACHOS_UPSTREAM};\n    proxy_http_version 1.1;\n    proxy_set_header Host $host;\n    proxy_set_header X-Real-IP $remote_addr;\n    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;\n    proxy_set_header X-Forwarded-Proto $scheme;\n  }\n  location / {\n    try_files $uri $uri/ /index.html;\n  }\n}\n' > /etc/nginx/templates/default.conf.template && \
    mkdir -p /var/cache/nginx /var/run && \
    chown -R app:app /var/cache/nginx /var/run /usr/share/nginx/html /etc/nginx/conf.d /etc/nginx/templates

ENV DESPACHOS_UPSTREAM=backend-despachos:8081
ENV VENTAS_UPSTREAM=backend-ventas:8080


# Copia solo el resultado compilado (dist) al runtime
COPY --from=build /app/dist /usr/share/nginx/html


# Ejecuta Nginx como usuario no-root
USER app
EXPOSE 8080


# Señal correcta para parar Nginx en contenedores
STOPSIGNAL SIGQUIT
CMD ["nginx","-g","daemon off;"]
