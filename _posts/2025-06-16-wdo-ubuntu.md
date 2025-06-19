# WDO en Ubuntu WSL

## Clonación del repositorio

Clonamos el repositorio de WDO desde [GitHub](https://github.com/carles9000/ut.wdo.git):

```bash
git clone https://github.com/carles9000/ut.wdo.git
```

## Compilación

Ejecutamos el siguiente comando:

```bash
./go_wdo_linux.sh
```

En mi caso, obtuve el error "Permission denied", lo que indica que el archivo no tiene permisos de ejecución. Para solucionarlo, ejecutamos:

```bash
chmod +x go_wdo_linux.sh
```

Luego, intentamos nuevamente con:

```bash
./go_wdo_linux.sh
```

Si todo va bien, veremos el mensaje:

```
Lib WDO was created !!!
```

## Instalación de MySQL

Instalamos el servidor MySQL con:

```bash
sudo apt install mysql-server
```

Iniciamos el servicio:

```bash
sudo service mysql start
```

Realizamos la configuración básica de seguridad:

```bash
sudo mysql_secure_installation
```

Este comando te pedirá que configures una contraseña para el usuario root, elimines usuarios anónimos, deshabilites el inicio de sesión remoto para root y elimines la base de datos de prueba.

Para comprobar que MySQL se ha instalado correctamente, accedemos con:

```mysql
sudo mysql -u root -p
```

## Instalación de bibliotecas necesarias

Para compilar y enlazar aplicaciones con MySQL, instalamos el paquete correspondiente:

```bash
sudo apt install libmysqlclient-dev
```

## Configuración y prueba de conexión

Editamos el archivo `app.prg` del repositorio y configuramos el valor de la variable:

```harbour
HB_SetEnv( 'WDO_PATH_MYSQL', "/usr/lib/x86_64-linux-gnu/" )
```

La ruta `/usr/lib/x86_64-linux-gnu/` corresponde a la ubicación de la biblioteca de MySQL.

Antes de compilar el ejemplo, es importante seguir las indicaciones del archivo README del repositorio para crear la base de datos de prueba.

Si todo está listo, estando en el directorio `ut.wdo` compilamos el ejemplo con:

```bash
./go_sample_linux.sh
```

Finalmente, accedemos al navegador en el puerto 81 y verificamos que todo funcione correctamente.

Aquí comparto mi experiencia y los pasos que seguí, que me funcionaron. Espero que estas notas sean útiles para alguien más.
