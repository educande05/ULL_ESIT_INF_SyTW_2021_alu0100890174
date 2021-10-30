![ULL](imagenes/Logo_Universidad_LaLaguna.png)

# Práctica 2 - Despliegue de una aplicación MEAN en el IaaS de la

## 1. Introducción:

En el siguiente informe documentremos la práctica número 2 dedicada a la instalación de un sistema servidor MEAN que sirva contenido Web a través del uso de un Proxy HTTP, NodeJS y una base de datos MongoDB distribuidos en diferentes servidores.

![Esquema servidor MEAN](https://github.com/educande05/ULL_ESIT_INF_SyTW_2021_alu0100890174/blob/main/practica_2/img/Captura%20de%20pantalla%202021-10-30%20105857.png)

Para ello seguiremos una serie de pasos concretos para la instalación de cada máquina servidora.

## 2. Objetivos:

1. Continuar el trabajo realizado en la práctica 1. [Informe 1](https://github.com/educande05/ULL_ESIT_INF_SyTW_2021_alu0100890174/blob/main/practica_1/Informe.md)
2. Configurar la MV que hace la función de Proxy HTTP. [instalación](https://linuxize.com/post/how-to-install-nginx-on-debian-9/)
3. Configuración del servidor backend NodeJS. [instalación](https://linuxconfig.org/how-to-install-nodejs-on-debian-9-stretch-linux)
4. Configuración del servidor de base de datos con MongoDB. [instalación](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/)
5. Configuración del servidor de base de datos como sistema de ficheros compartido(NFS). [instalación](https://wiki.debian.org/NFSServerSetup)

## 3. Desarrollo:

### Configuración de la MV Proxy HTTP

Primero instalaremos Nginx para que nuestro servidor actue como un proxy.

```bash
sudo apt update
sudo apt install nginx
```

Instalamos curl para verificar la conectividad de la dirección IP.

```bash
sudo apt install curl
curl -I 127.0.0.1
```

Abrimos los puertos del firewall.

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

Configuramos el servidor como Reverse Proxy (el servidor recibe las peticiones de la web y este las reenvia al backend). Para ello tenemos que configurar un archivo conf que refiera al Backend. Editamos ```/etc/nginx/sites-available/default``` añadiendo:

```
location / {
  proxy_pass http://172.16.133.4:3000; #IP Backend
}
```

Reseteamos Nginx
```bash
sudo service nginx configtest
sudo service nginx restart
```


### Configuración de la MV Backend con NodeJS

Instalamos Curl.

```bash
sudo apt update
sudo apt install curl
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
```

Instalamos NodeJS

```bash
sudo apt install nodejs
```

Instalamos npm

```bash
sudo apt install build-essential libssl-dev
```

Abrimos el puerto 3000 para que exista conexión con el proxy.

```bash
sudo iptables -A INPUT -p tcp --dport 3000 -j ACCEPT
```

Para probar que funciona correctamente usaremos una aplicación de ejemplo de express.

```bash
mkdir app
cd app
npm init -y 
npm install express
node app.js
```

Como podemos comprobar nuestra app de prueba corre perfectamente a través del proxy:

![imagen aplicación de prueba](https://github.com/educande05/ULL_ESIT_INF_SyTW_2021_alu0100890174/blob/main/practica_2/img/Captura%20de%20pantalla%202021-10-30%20133044.png)


### Configuración de la MV Base de datos con MongoDB

Instalamos MongoDB

```bash
sudo apt-get install gnupg
wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
echo "deb http://repo.mongodb.org/apt/debian stretch/mongodb-org/5.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```

Activamos el servicio de mongoDB para que se active automáticamente al iniciar el servidor.

```bash
sudo systemctl enable mongod
sudo systemctl start mongod
sudo systemctl daemon-reload
sudo systemctl status mongod
```

Configuramos el usuario administrador para MongoDB

```bash
mongosh
$\ use admin
$\ db.createUser({user: "Edu", pwd: "clave", roles: ["root"]});
$\ quit()
```

Configuramos los ajustes del fichero ```/etc/mongod.conf```. Los ajustes cambiados son los siguientes.

![Imagen configuracion mongod.conf](https://github.com/educande05/ULL_ESIT_INF_SyTW_2021_alu0100890174/blob/main/practica_2/img/Captura%20de%20pantalla%202021-10-30%20135332.png)

Reiniciamos el servidor MongoDB

```bash
sudo systemctl restart mongod
```

Ahora podemos acceder a la base de datos con el usuario registrado como Root:

```
mongosh admin -u Edu -p "pass"
```

### Configuración de la MV Base de datos como sistema de ficheros compartidos (NFS)


Para empezar debemos instalar NFS, herramienta que nos permitirá compartir un sistema de archivos entre la Base de datos y el Backend.

```bash
sudo apt-get install nfs-kernel-server
sudo apt-get install nfs-common
```

debemos realizar una serie de cambios en la configuración de NFS realizando cambios en los siguientes archivos:

* nfs-common
```bash
sudo vi /etc/default/nfs-common
```

```
NEED_STATD=”no”
NEED_IDMAPD=”yes”
```

* nfs-kernel-server
```bash
sudo vi /etc/default/nfs-kernel-server
```

```
RPCNFSDOPTS="-N 2 -N 3"
RPCMOUNTDOPTS="--manage-gids -N 2 -N 3"
```

* /etc/exports
```bash
sudo vi /etc/exports
```

```
/home/usuario/public 172.16.134.4/24(rw,no_root_squash,no_subtree_check,sync)
```

ahora aplicamos los cambios realizados en la configuración

```bash
sudo exportfs -a
sudo /etc/init.d/nfs-kernel-server reload
```

Por último, quitamos algunos detalles que se establecen por defecto pero
que no necesitan la versión 4.

```bash
sudo systemctl mask rpcbind.service
sudo systemctl mask rpcbind.socket
```

Reactivamos nfs.

```bash
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
sudo systemctl status nfs-server
```


### Configuración de la MV Backend como sistema de ficheros compartidos (NFS)


Instalamos nfs:

```bash
sudo apt-get install nfs-common
```

Montamos la carpeta compartida:

```bash
sudo mount 172.16.134.4:/home/usuario/public /home/usuario/public
```

Para hacer los cambios persistentes.

```bash
sudo vi /etc/fstab
```

Añadimos

```
172.16.134.3:/home/usuario/public /home/usuario/public nfs auto 0 0
```



