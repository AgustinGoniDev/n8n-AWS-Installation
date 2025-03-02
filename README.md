# Guía de Configuración AWS con Docker, Nginx y SSL para n8n


## Paso 1: Crear la instancia EC2 en AWS

- Ingresar a AWS

- Buscá "EC2" en el buscador y entrá al servicio.

- Lanzar una nueva instancia

### Crear una nueva instancia en AWS

- Hacé clic en "Launch instance" (Iniciar instancia).

- Nombre: Poné algo como n8n-server.

- Imagen de Amazon Machine (AMI): Elegí Ubuntu 22.04 LTS (versión recomendada por estabilidad y soporte).

- Tipo de instancia: Si es para pruebas, podés elegir t2.micro (gratis con Free Tier).

- Par de claves (Key Pair): Creá una nueva o usá una existente (vas a necesitarla para conectarte por SSH).

### Grupo de seguridad:

- Permití conexiones SSH (22) desde tu IP (o un rango seguro).

- Permití tráfico en los puertos 80 (HTTP) y 443 (HTTPS) desde cualquier origen (0.0.0.0/0).

- Permití el puerto 5678 (n8n) si lo vas a acceder directamente.

- Revisá la configuración y dale a Launch Instance.

- Esperá a que la instancia inicie y copiá su IP pública.
  

## Paso 2: Conectarse a la instancia de AWS EC2 vía SSH

```bash
ssh -i yourkey.pem ec2-user@publicip
```

## Paso 3: Instalar Docker y Docker Compose
### 1- Actualizar paquetes
```bash
sudo apt update && sudo apt upgrade -y
```
### 2- Instalar dependencias
```bash
sudo apt install -y ca-certificates curl gnupg
```
### 3- Agregar repositorio de Docker
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```
### 4- Instalar Docker y Docker Compose
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
### 5- Agregar tu usuario a Docker para no necesitar "sudo"
```bash
sudo usermod -aG docker $USER
```
### 6- Reiniciar para aplicar los cambios
```bash
exit
```
### 7- Volver a conectarte por SSH


## Paso 4: Configurar n8n con Docker
### 1- Crear una carpeta para n8n
```bash
mkdir -p ~/n8n && cd ~/n8n
```
### 2- Crear un archivo docker-compose.yml
```bash
nano docker-compose.yml
```
Pegá esto dentro:
```yaml

services:
  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
        - N8N_HOST=n8n.tudominio.com
        - N8N_PROTOCOL=https
        - WEBHOOK_URL=https://n8n.tudominio.com/
    volumes:
      - ~/.n8n:/home/node/.n8n
```
Guarda con: 
```bash
Ctrl + X, Y. Enter
```
### 3- Levantar n8n 
```bash
docker compose up -d
```

## Paso 5: Configurar Nginx como Proxy Reverso con SSL
Se utiliza Nginx y Let's Encrypt para que tu entorno de n8n sea accesibl con dominio y HTTPS
### 1- Instalar Nginx y Certbot
```bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 2- Crear un archivo de configuración para n8n
```bash
sudo nano /etc/nginx/sites-available/n8n
```

Pegá esto (reemplazá n8n.tudominio.com por tu dominio real):
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

        # Headers for WebSocket support
        proxy_set_header Connection 'Upgrade';
        proxy_set_header Upgrade $http_upgrade;

        # Additional headers for forwarding client info
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Guarda con: 
```bash
CTRL+O, ENTER, CTRL+X
```

### 3- Habilitar la configuración y reiniciar Nginx
```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 4- Generar el certificado SSL con Let's Encrypt
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain-name
sudo systemctl restart nginx
```

## Paso 6: Configurar Renovación Automática de SSL

Certbot renueva los certificados automáticamente, pero podés verificar con:
```bash
sudo certbot renew --dry-run
```
## Y listo, accediendo a tu dominio deberias ver la pantalla de registro de n8n.

# En caso de que no te conecte:
Para encontrar el error debes ver los logs de tu contenedor

### 1- Ver si el contenedor esta corriendo
```bash
docker ps
```
Si no esta corriendo, levantarlo con
```bash
docker compose up -d
```
### 2- Probar si n8n responde dentro de servidor
```bash
curl -I http://localhost:5678
```
Si responde con un HTTP/1.1 200 OK, significa que n8n está funcionando bien. Si no responde, el problema está en el contenedor.

## Si arroja un error como:
```bash
curl: (7) Failed to connect to localhost port 5678 after 0 ms: Couldn't connect to server
```
### 3- Obtener ID del contenedor de n8n con "docker ps"

### 4- Ver los logs del contenedor
```bash
docker logs -f <ID_DEL_CONTENEDOR>
```

# Posible caso de error:
Si el error que vemos es algo como:
```bash
EACCES: permission denied, open '/home/node/.n8n/config
```
## Segui estos pasos en tu servidor para darle acceso a n8n
### 1- Da permisos de lectura/escritura
```bash
sudo chown -R 1000:1000 ~/.n8n
sudo chmod -R 775 ~/.n8n
```
### 2- Volve a levantar n8n
```bash
docker-compose down
docker-compose up -d
```



