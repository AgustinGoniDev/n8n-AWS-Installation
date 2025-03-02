# Configuración de AWS para Docker y Nginx & Certificado con Certbot con n8n.

### Requisitos previos:
- Cuenta de AWS
- Creación de instancia EC2 (Si recién te registras podes tenerla por 1 año gratis)
- Creación de archivo .pem para claves SSH (En caso de no tenerlo)
- Tener un dominio (Aca te dejo como tener uno gratis, https://www.noip.com/es-MX)

### Paso 1: Conectarse a la instancia de AWS EC2 vía SSH
```bash
ssh -i yourkey.pem ec2-user@publicip
```

### Paso 2: Actualizar la instancia e instalar Docker
```bash
sudo apt update -y
sudo apt install -y docker
```

### Paso 3: Iniciar y habilitar el servicio Docker
```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### Paso 4: Agregar el usuario al grupo Docker para acceso sin root
```bash
sudo usermod -aG docker ec2-user
```

### Paso 5: Salir y volver a ingresar para aplicar los cambios
```bash
exit
```

### Paso 6: Conectarse a la instancia de AWS EC2 vía SSH
```bash
ssh -i yourkey.pem ec2-user@publicip
```

### Paso 7: Instalar Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Paso 8: Ejecutar el contenedor Docker de N8N
```bash
sudo docker run -d --restart unless-stopped -it \
--name n8n \
-p 5678:5678 \
-e N8N_HOST="your-domain-name" \
-e WEBHOOK_TUNNEL_URL="https://your-domain-name/" \
-e WEBHOOK_URL="https://your-domain-name/" \
-v ~/.n8n:/root/.n8n \
n8nio/n8n
```

### Paso 9: Instalar y configurar Nginx
```bash
sudo dnf install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Paso 10: Configurar Nginx como Proxy Inverso para N8N
```bash
sudo nano /etc/nginx/conf.d/n8n.conf
```
Agregar el siguiente contenido en `n8n.conf`:
```nginx
server {
    listen 80;
    server_name your-domain-name;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;

        proxy_set_header Connection 'Upgrade';
        proxy_set_header Upgrade $http_upgrade;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Guardar y salir usando:
```bash
CTRL+O, ENTER, CTRL+X
```

### Paso 11: Probar la configuración de Nginx y reiniciar el servicio
```bash
sudo nginx -t
sudo systemctl restart nginx
```

### Paso 12: Configurar el certificado SSL con Certbot
```bash
sudo dnf install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain-name
sudo systemctl restart nginx
