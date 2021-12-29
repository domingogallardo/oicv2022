
# Instalación evaluador olimpiada informática (CMS) #

## Referencias ##

Documentación CMS (Contest Management System): https://cms.readthedocs.io

## Instalación ##

- Ubuntu 18.04

- Creamos usuario con permiso de sudo y cambiamos a ese usuario para
hacer desde ahí la instalación (voy a crear un usuario con mi nombre, 'domingo').
  
```
# adduser domingo
# usermod -aG sudo domingo 
# su - domingo
```

- Instalamos los paquetes requeridos:

```
$ sudo apt-get install build-essential openjdk-8-jdk-headless fp-compiler \
    postgresql postgresql-client python3.6 cppreference-doc-en-html \
    cgroup-lite libcap-dev zip
$ sudo apt-get install python3.6-dev libpq-dev libcups2-dev libyaml-dev \
    libffi-dev python3-pip
```

- Descargamos CMS 1.4.rc1.

La Olimpiada Española tiene un [repositorio Git](https://github.com/olimpiada-informatica/cms) con algunas 
modificaciones en la aplicación de Ranking. Para descargar esta versión:

```
$ git clone --recursive https://github.com/olimpiada-informatica/cms.git
```

La versión original se puede descargar de la siguiente forma:

```
$ wget https://github.com/cms-dev/cms/releases/download/v1.4.rc1/v1.4.rc1.tar.gz
$ tar xvfz v1.4.rc1.tar.gz
```

- Preparamos la instalación (se crea el grupo `cmsuser` y se añade el
  usuario actual a ese grupo):
  
```
$ sudo python3 prerequisites.py install
// salimos del shell y volvemos a entrar para que se incorpore el
// usuario al grupo 
$ exit
# su - domingo
$ groups
domingo sudo cmsuser
```

- Instalamos CMS globalmente con pip:

```
$ cd cms
$ sudo pip3 install -r requirements.txt
$ sudo python3 setup.py install
```

- Configuramos la base de datos Postgres:

```
$ sudo su - postgres
$ createuser --username=postgres --pwprompt cmsuser (pide definir password, yo he usado el mismo nombre cmsuser)
$ createdb --username=postgres --owner=cmsuser cmsdb
$ psql --username=postgres --dbname=cmsdb --command='ALTER SCHEMA public OWNER TO cmsuser'
$ psql --username=postgres --dbname=cmsdb --command='GRANT SELECT ON pg_largeobject TO cmsuser'
$ exit
```

- Añadimos la contraseña a la configuración de la conexión con la base de datos en `/usr/local/etc/cms.conf`:

```
"database": "postgresql+psycopg2://cmsuser:cmsuser@localhost:5432/cmsdb",
```

- Inicializamos el esquema de base de datos con la utilidad `cmsInitDB`:

```
$ cmsInitDB
2021-10-15 10:22:51,002 - INFO [<unknown>] Using configuration file /usr/local/etc/cms.conf.
```

- Creamos un usuario admin:

```
$ cmsAddAdmin domingo
// Se muestra la contraseña por la consola:
2021-10-15 10:28:02,850 - INFO [<unknown>] Using configuration file /usr/local/etc/cms.conf.
2021-10-15 10:28:03,281 - INFO [<unknown>] Creating the admin on the database.
2021-10-15 10:28:03,583 - INFO [<unknown>] Admin with complete access added. Login with username domingo and password aenrrc
```

## Crear una competición ##

Para ejecutar CMS necesitamos añadir una competición en la base de
datos. 

- Lanzamos el servidor de administración

```
$ cmsAdminWebServer
```

- Nos conectamos a puerto 8889 (<http://localhost:8889>) y nos
  logeamos con el usuario y contraseña anterior.

Desde este administrador tendremos que crear todos los elementos de la
competición (usuarios, preguntas, anuncios, ...). En la documentación
también se dice que se puede exportar e importar una competición, pero
todavía no lo he probado.

- Desde el menú principal creamos una competición, una tarea y un
  usuario con valores por defecto.

- Añadimos la tarea y el usuario a la competición.

- Podemos cerrar el servidor de administración

## Puesta en marcha ##

- Lanzamos los siguientes servicios (usamos distintas terminales
  para cada servicio):

```
$ cmsLogService (logging)
$ cmsResourceService -a (nos pide el número de competición y lanza
  todos los servicios)
Contests available:
  1  -  ID: 1  -  Name: Competicion1  -  Description: Competicion1 (default)
Insert the row number next to the contest you want to load (not the id): 1
$ cmsRankingWebServer
```

El servidor en el que se ve el ranking está accesible por defecto en
el puerto 8890 y el servidor al que se conectan los participantes en
el puerto 8888.

- Ranking: <http://localhost:8890/Ranking.html>
- Participantes: <http://localhost:8888>

## Puesta en marcha (en background) ##

Para poner en marcha los procesos mínimos de forma que queden en background
cuando salimos del terminal:

```
$ nohup cmsResourceService -a <<< '1' &
$ nohup cmsRankingWebServer &
```

## Reinicio del ranking ##

El servicio `cmsRankingWebServer` muestra los datos guardados en el 
directorio `/var/local/lib/cms/ranking`. Si queremos reiniciar este servidor
y resetear sus datos podemos eliminar este directorio. Debemos después reiniciar
el servicio y también el `cmsProxyService`:

```
$ ps aux | grep cmsRankingWebServer
domingo   1182  0.0 /usr/bin/python3 /usr/local/bin/cmsRankingWebServer
$ kill -9 1182
$ cd /var/local/lib/cms/
$ rm -rf ranking
$ nohup cmsRankingWebServer &
$ ps aux | grep cmsProxyService 
domingo   7584  0.7 /usr/bin/python3 /usr/local/bin/cmsProxyService 0 -c 1
$ kill -9 7584
```
Al estar en marcha el `cmsResourceService`, éste reinicia el servicio de proxy
y se vuelven a volcar todos los datos de la competición actual en el directorio
`/var/local/lib/cms/ranking`.

## Importar usuarios ##

Comando:

```
$ cmsImportUser -L italy_yaml -c 1 --all
```

El fichero con los usuarios se debe llamar `contest.yaml` y tiene el siguiente formato:

```
nome_breve: testcontest
nome: "Test contest"

problemi:
  - testtask

# start: Monday, January 1, 2018 12:00:00 PM (GMT)
inizio: 1514808000
# end:  Friday, January 1, 2100 12:00:00 PM (GMT)
fine: 4102488000

utenti:
- username: 'u1'
  password: 'aaa'
- username: 'u2'
  password: 'bbb'
- username: 'u3'
  password: 'ccc'
- username: 'u4'
  password: 'ddd'
- username: 'u5'
  password: 'eee'