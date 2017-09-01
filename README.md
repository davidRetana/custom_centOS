# custom_centOS
Modificación de una imagen de CentOS para albergar un cluster Hadoop

27/12/2016
version de SO: centOS 6.7

#######################################################################################
############################## CONFIGURACION DE RED ###################################
#######################################################################################
> ifconfig -a => localizamos la eth0 en nuestro caso
> dhclient eth0 => Solicitamos una direccion IP al adaptador de red eth0
> ping google.com => probamos si funciona la conexion

# Esto que hemos hecho ha sido pedirle una IP al servidor DHCP y este nos ha asignado
# una dinamicamente. Si queremos hacerlo fijo debemos tocar los siguientes ficheros:
 ==> /etc/sysconfig/network-scripts/ifcfg-eth0
     Modificar la linea ONBOOT=YES
     Añadir las siguientes lineas
       IPADDR=10.164.77.163 (la IP que nos haya dado en ifconfig)
       BOOTPRO=static
       NETMASK=255.255.255.0

 ==> /etc/sysconfig/network
     Modificar las siguientes lineas: 
       HOSTNAME=localhost (el nombre de la maquina)

 ==> /etc/resolv.conf
     No hemos modificado nada y funciona correctamente

Ahora reiniciamos la configuracion de red
/etc/init.d/network restart

#####################################################################################
################################ USUARIOS DEL SISTEMA ###############################
#####################################################################################
> useradd -d /home/training -m -s /bin/bash training  ==> añadimos el usuario training
> passwd training  ==> le damos una contraseña al usuario training
> Añadir al usuario training al archivo /etc/sudoers (permitirle lanzar comandos como sudo)
> visudo  ==> modificamos el archivo /etc/sudoers
   añadir la linea ==> training  ALL=(ALL)  ALL

#####################################################################################
######### CONFIGURACION DE LA MAQUINA PARA ALBERGAR UN HOST HADOOP ##################
#####################################################################################

# Deshabilitar el firewall
> service iptables stop ==> paramos el servicio
> chkconfig iptables off  ==> para que cuando reiniciemos no se vuelva a activar

# Deshabilitar selinux
> /usr/sbin/getenforce  ==> comporbar si esta habilitado
> vi /etc/selinux/config
    modificar la linea => SELINUX=disabled
> reboot  ==> reiniciamos la maquina para que haga efecto

# Desabilitar el swappiness
> vi /etc/sysctl.conf
    añadir la linea => vw.swappiness=1

# Deshabilitar THP (Transparent Huge Page)
> vi /etc/rc.local (añadimos el siguiente contenido)

   if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
      echo never > /sys/kernel/mm/transparent_hugepage/enabled
   fi
   if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
      echo never > /sys/kernel/mm/transparent_hugepage/defrag
   fi
> echo never > /sys/kernel/mm/transparent_hugepage/defrag  ==> lo desactivamos

# Instalar cliente NTP para la sincronizacion de relojes
> yum install ntp ntpdate ntp-doc  ==> Instalamos los paquetes necesarios
> chkconfig ntpd on  ==> lo hacemos permanente aunque reiniciemos la maquina
> ntpdate es.pool.ntp.org  ==> cogemos la hora del servidor (en España)
> /etc/init.d/ntpd start  ==> arrancamos el servicio
> ntpstat  ==> comprobamos que funciona correctamente

# Descargamos el repositorio de Cloudera (CDH)
> yum update
> yum install wget
> OPCIONAL => yum install man ==> manual de los comandos en la propia maquina
> wget https://archive.cloudera.com/cdh5/one-click-install/redhat/6/x86_64/cloudera-cdh-5-0.x86_64.rpm
    ==> nos descargamos el repositorio oficial de cloudera
> yum --nogpgcheck localinstall cloudera-cdh-5-0.x86_64.rpm  ==> añadimos cloudera a los repositorios de yum
> vi /etc/yum.repos.d/cloudera-cdh5.repo  ==> modificamos la version de CHD a la deseada
     baseurl=http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/X.X.X (5.8.2)
> yum clean all  ==> limpiamos la cache de yum
> rm -f cloudera-cdh-5-0.x86_64.rpm  ==> eliminamos el archivo descargado

# Descargamos el repositorio del Cloudera Manager
> wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo
> vi cloudera-manager.repo  ==> abrimos el archivo y modificamos la siguiente linea:
        baseurl=https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5.8.2/
> mv cloudera-manager.repo /etc/yum.repos.d/  ==> movemos el archivo a la ruta de los repositorios

#??????????????????????????????????????????????????
> vi /etc/gdm/custom.conf  ==> añadir las siguientes lineas
    [daemon]
     AutomaticLoginEnable=true
     AutomaticLogin=training

# Instalacion de Java 7 (JDK 7)
> wget -O - --no-cookies --no-check-certificate --header "Cookie: \
  gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" \
  "http://download.oracle.com/otn-pub/java/jdk/7u79-b15/jdk-7u79-linux-x64.rpm" > \
  jdk-7u79-linux-x64.rpm  ==>  nos descargamos el archivo de instalacion de Java
> yum localinstall --assumeyes jdk-7u79-linux-x64.rpm  ==> instalamos el jdk de Java
> java -version  ==> comprobamos que la instalacion se ha realizado correctamente
> rm -f jdk-7u79-linux-x64.rpm  ==> eliminamos el archivo

# Instalacion de MySQL
> yum install mysql-server  ==> Instalamos el servidor de MySQL
> /sbin/service mysqld start  ==> Arrancamos el demonio
> /usr/bin/mysql_secure_installation  ==> Lanzamos el proceso de instalacion
    Enter current password for root (enter for none): (Lo dejamos en blanco)
    Set root password? [Y/n] (damos a Y)
    New password:
    Re-enter new password:
    Remove anonymous users? [Y/n] Y
    Disallow root login remotely? [Y/n] n
    Remove test database and access to it? [Y/n] n
    Reload privilege tables now? [Y/n] Y
> sudo chkconfig mysqld on  ==> Establecemos el arranque automático al iniciar la maquina
> mysql -uroot -ptraining  ==> comprobamos que todo funciona correctamente
> vi /etc/my.cnf  ==> modificamos el fichero y le dejamos asi:
        [mysqld]
        transaction-isolation = READ-COMMITTED
        datadir=/var/lib/mysql
        socket=/var/lib/mysql/mysql.sock
        user=mysql
        # Disabling symbolic-links is recommended to prevent assorted security risks
        #symbolic-links=0

        key_buffer = 16M
        key_buffer_size = 32M
        max_allowed_packet = 32M
        thread_stack = 256K
        thread_cache_size = 64
        query_cache_limit = 8M
        query_cache_size = 64M
        query_cache_type = 1

        max_connections = 550
        #expire_logs_days = 10
        #max_binlog_size = 100M

        #log_bin should be on a disk with enough free space. Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your system
        #and chown the specified folder to the mysql user.
        log_bin=/var/lib/mysql/mysql_binary_log

        # For MySQL version 5.1.8 or later. Comment out binlog_format for older versions                                                                        .
        binlog_format = mixed

        read_buffer_size = 2M
        read_rnd_buffer_size = 16M
        sort_buffer_size = 8M
        join_buffer_size = 8M

        # InnoDB settings
        default-storage-engine = InnoDB
        innodb_file_per_table = 1
        innodb_flush_log_at_trx_commit  = 2
        innodb_log_buffer_size = 64M
        innodb_buffer_pool_size = 2G
        innodb_thread_concurrency = 8
        innodb_flush_method = O_DIRECT
        innodb_log_file_size = 512M


        [mysqld_safe]
        log-error=/var/log/mysqld.log
        pid-file=/var/run/mysqld/mysqld.pid

        sql_mode=STRICT_ALL_TABLES
> /sbin/service mysqld restart
POSIBLES ERRORES AL REINICIAR Y TROUBLESHOOTING  (se ven en /var/log/mysqld.log)
==> Fatal Error: cannot allocate the memory for the buffer pool
     solucion ==> rebajar la memoria en /etc/my.cnf
                  innodb_buffer_pool_size=2G
==> Si por el contrario, consigue arrancar el buffer pool, debemos eliminar los siguientes archivos
     solucion ==> rm -rf /var/lib/mysql/ib_logfile0 /var/lib/mysql/ib_logfile1
> /sbin/service mysqld restart  ==> reiniciamos el servcio
> mysql -uroot -ptraining
  > show engines  ==> ver que InnoDB esta como motor por defecto

# Descargamos el driver JDBC de MySQL
http://www.mysql.com/downloads/connector/j/5.1.html  ==> lo debemos descargar desde la pagina y luego copiarlo
a la maquina con pscp o MobaXterm, copiarlo a /root
> yum install zip unzip  ==> para poder comprimir y descomprimir archivos
> unzip mysql-connector-java-5.1.40.zip  ==> descomprimimos el archivo
> mkdir /usr/share/java  ==> creamos el directorio
> mv mysql-connector-java-5.1.40/mysql-connector-java-5.1.40-bin.jar /usr/share/java/  ==> movemos el jar al directorio
> rm -rf mysql-connector-java-5.1.40 mysql-connector-java-5.1.40.zip  ==> eliminamos los archivos para que no ocupen
> mv /usr/share/java/mysql-connector-java-5.1.40-bin.jar /usr/share/java/mysql-connector-java.jar ==> 
                                          Modificamos el nombre del driver JDBC y le quitamos la versión -- crear un link


# Preparacion de las bases de datos en MySQL para albergar el cluster Hadoop
> mysql -uroot -ptraining
        create database amon DEFAULT CHARACTER SET utf8;
        grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'amon_password';

        create database rman DEFAULT CHARACTER SET utf8;
        grant all on rman.* TO 'rman'@'%' IDENTIFIED BY 'rman_password';

        create database metastore DEFAULT CHARACTER SET utf8;
        grant all on metastore.* TO 'hive'@'%' IDENTIFIED BY 'hive_password';

        create database sentry DEFAULT CHARACTER SET utf8;
        grant all on sentry.* TO 'sentry'@'%' IDENTIFIED BY 'sentry_password';

        create database nav DEFAULT CHARACTER SET utf8;
        grant all on nav.* TO 'nav'@'%' IDENTIFIED BY 'nav_password';

        create database navms DEFAULT CHARACTER SET utf8;
        grant all on navms.* TO 'navms'@'%' IDENTIFIED BY 'navms_password';

# Hacer esto solo en el nodo del cloudera manager server
        create database cloudera DEFAULT CHARACTER SET utf8;
        grant all on cloudera.* TO 'root'@'%' IDENTIFIED BY 'training';
