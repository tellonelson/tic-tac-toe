# **Tutorial: Despliegue Automático de React Tic-Tac-Toe en AWS EC2 con CI/CD usando GitHub Actions**

## **Objetivo**
Aprender a:
1. Configurar un servidor EC2 en AWS con una IP elástica fija y Nginx para servir una aplicación React.
2. Implementar un workflow en GitHub Actions que compile, despliegue y actualice la aplicación automáticamente.

---

## **1. Configuración del Servidor AWS EC2**

### **Paso 1: Crear una instancia EC2**
1. Accede a la **Consola de AWS** y navega a **EC2 > Instances > Launch Instances**.
2. Configura lo siguiente:
   - **AMI**: Ubuntu Server 22.04.
   - **Instance type**: `t2.micro` (gratis en el nivel gratuito de AWS).
   - **Key pair**: Si no tienes un par de claves creado, configúralo en esta sección, aunque no lo usaremos directamente.
   - **Security Group**: Asegúrate de que permita tráfico entrante en el puerto `80` (HTTP).
3. Haz clic en **Launch Instance** y espera a que se inicie.

---

### **Paso 2: Asignar una IP elástica fija**
1. En la consola de AWS, ve a **Elastic IPs > Allocate Elastic IP Address**.
2. Asigna una nueva IP elástica.
3. Asocia la IP a la instancia EC2 creada:
   - Ve a **Actions > Associate Elastic IP Address**.
   - Selecciona tu instancia EC2 y guarda los cambios.

Esta IP será la dirección fija para tu servidor y se usará en la configuración del workflow.

---

### **Paso 3: Conectar al servidor EC2 con EC2 Instance Connect**
1. Desde la consola de AWS, selecciona tu instancia EC2.
2. Haz clic en **Connect > EC2 Instance Connect**.
3. Usa el terminal en el navegador para conectarte directamente al servidor.

---

### **Paso 4: Configurar el servidor**
#### Actualizar paquetes e instalar Nginx:
```bash
sudo apt update
sudo apt install nginx -y
```

#### Crear el directorio para la aplicación:
```bash
sudo mkdir -p /var/www/tic-tac-toe
sudo chown -R $USER:$USER /var/www/tic-tac-toe
```

#### Configurar Nginx:
Edita el archivo de configuración para servir la aplicación React:
```bash
sudo nano /etc/nginx/sites-available/tic-tac-toe
```

Agrega lo siguiente:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/tic-tac-toe;
    index index.html;

    location / {
        try_files $uri /index.html;
    }
}
```

Habilita la configuración de Nginx y reinicia el servicio:
```bash
sudo ln -s /etc/nginx/sites-available/tic-tac-toe /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

## **2. Configuración del Workflow de GitHub Actions**

El workflow tiene dos jobs:
1. **Build**: Compila la aplicación React.
2. **Deploy**: Sube los archivos generados al servidor EC2 y reinicia Nginx.

---

### **Archivo del Workflow**

Crea el archivo **`.github/workflows/deploy.yml`** con el siguiente contenido:

```yaml
name: CI/CD Despliegue React a EC2

on:
  push:
    branches:
      - main

jobs:
  build:
    name: Compilar React App
    runs-on: ubuntu-latest

    steps:
      - name: Checkout del repositorio
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Instalar dependencias
        run: npm install

      - name: Compilar aplicación
        run: npm run build

      - name: Subir artefacto
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: dist

  deploy:
    name: Desplegar en EC2
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Descargar artefacto
        uses: actions/download-artifact@v4
        with:
          name: build-artifact
          path: .

      - name: Copiar archivos al servidor EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "./*"
          target: "/var/www/tic-tac-toe"

      - name: Reiniciar Nginx
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo systemctl restart nginx
```

---

### **Explicación del Workflow**

#### **Job: `build`**
1. **`actions/checkout`**:
   - Clona el repositorio en el runner.
2. **`actions/setup-node`**:
   - Configura la versión 20 de Node.js.
3. **`npm install`**:
   - Instala las dependencias del proyecto.
4. **`npm run build`**:
   - Compila la aplicación React, generando archivos estáticos en la carpeta `dist`.
5. **`actions/upload-artifact`**:
   - Sube los archivos generados como un artefacto llamado `build-artifact`.

---

#### **Job: `deploy`**
1. **`actions/download-artifact`**:
   - Descarga el artefacto `build-artifact` generado por el job `build`.
2. **`appleboy/scp-action`**:
   - Copia los archivos desde el runner al directorio `/var/www/tic-tac-toe` en el servidor EC2.
3. **`appleboy/ssh-action`**:
   - Ejecuta un comando SSH en el servidor EC2 para reiniciar Nginx y aplicar los cambios.

---

## **3. Configuración de GitHub Secrets**

En **Settings > Secrets and Variables > Actions** de tu repositorio, agrega los siguientes secretos:

- **`EC2_HOST`**: IP elástica de tu servidor EC2.
- **`EC2_SSH_KEY`**: Clave privada para conectarte al servidor EC2.
- **`EC2_USER`**: `ubuntu` (usuario predeterminado para Ubuntu Server).

---

## **4. Flujo de Despliegue**

1. Haz cambios en el código y realiza un **push** a la rama `main`.
2. El workflow de GitHub Actions:
   - Compila la aplicación React.
   - Sube los archivos a tu servidor EC2.
   - Reinicia Nginx para que sirva la aplicación actualizada.
3. Accede a la **IP elástica** de tu servidor en el navegador para verificar el despliegue.

---

## **Resultado**

Cada vez que hagas un **push** a la rama `main`, tu aplicación React se desplegará automáticamente en tu servidor EC2, accesible desde la **IP elástica fija**.