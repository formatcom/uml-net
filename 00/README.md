- REF: [https://www.kernel.org/doc/html/latest/virt/uml/user_mode_linux_howto_v2.html](https://www.kernel.org/doc/html/latest/virt/uml/user_mode_linux_howto_v2.html) 
- REF: [https://www.kernel.org/doc/html/v5.9/virt/uml/user_mode_linux.html](https://www.kernel.org/doc/html/v5.9/virt/uml/user_mode_linux.html) 
- REF: [https://www.gnu.org/software/libc/manual/html_node/Asynchronous-I_002fO-Signals.html](https://www.gnu.org/software/libc/manual/html_node/Asynchronous-I_002fO-Signals.html) 
- REF: [http://user-mode-linux.sourceforge.net/old/skas.html](http://user-mode-linux.sourceforge.net/old/skas.html) 

***


### Kernel User Mode Linux [ UML ]

User Mode Linux es la primera plataforma de virtualización de código abierto (primera fecha de lanzamiento 1991) y la segunda plataforma de virtualización para una PC x86.

No hay un solo dispositivo real a la vista. Es 100% artificial. Todos los dispositivos UML son conceptos abstractos que se asignan a algo proporcionado por el host: archivos, sockets, tuberías, etc.

El kernel de UML es solo un proceso que se ejecuta en Linux, al igual que cualquier otro programa. Puede ser ejecutado por un usuario sin privilegios y no requiere nada en términos de características especiales de la CPU.

UML es estrictamente uniprocesador en la actualidad. Si desea ejecutar una aplicación que necesita muchas CPU para funcionar, claramente es la elección incorrecta.

~~~

            +----------------+
            | Process 2 | ...|
+-----------+----------------+
| Process 1 | User-Mode Linux|
+----------------------------+
|       Linux Kernel         |
+----------------------------+
|         Hardware           |
+----------------------------+
~~~

### ¿Por qué querría Linux en modo de usuario?

- Si el kernel de Linux en modo de usuario falla, su kernel de host aún está bien. No se acelera de ninguna manera (vhost, kvm, etc.) y no intenta acceder a ningún dispositivo directamente. De hecho, es un proceso como cualquier otro.

- Puede ejecutar un kernel en modo de usuario como usuario no root (es posible que deba disponer los permisos adecuados para algunos dispositivos).

- Puede ejecutar una máquina virtual muy pequeña con una huella mínima para una tarea específica (por ejemplo, 32 M o menos).

- Puede obtener un rendimiento extremadamente alto para cualquier cosa que sea una “tarea específica del kernel” como reenvío, cortafuegos, etc. sin dejar de estar aislado del kernel del host.

- Puedes jugar con los conceptos del kernel sin romper cosas.

- No está obligado a "emular" hardware, por lo que puede probar conceptos extraños y maravillosos que son muy difíciles de soportar cuando se emula hardware real como el viaje en el tiempo y hacer que el reloj de su sistema dependa de lo que hace UML (muy útil para cosas como pruebas) .

- Es divertido.

### ¿Donde lo usaremos nosotros?
Lo usaremos para crear muchas maquinas virtuales con muy pocos recursos y asi poder estudiar de una manera elegante.

![](/home/lowlevel/repos/uml-net/00/uml0.png) 

### Construyendo una instancia de UML

~~~bash
host $ curl -LO https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.13.7.tar.gz
host $ tar xvf linux-5.13.7.tar.gz
host $ cd linux-5.13.7.tar.gz
~~~

Instalar algunas dependencias, la maquina **host** utilizada es **Fedora**.

~~~bash
host $ sudo dnf install ncurses-devel gcc gcc-c++ make dkms pkgconfig 
~~~

Configurar el kernel, si no tienes mucha idea de este paso, con ejecutar el siguiente comando y guardarlo sin tocar nada te valdria.

NOTA:  si lo dejas por defecto el kernel no soportara ipv6 ya que este no viene habilitado por defecto y debe ser configurado manualmente.

~~~bash
host $ make menuconfig ARCH=um SUBARCH=x86_64
~~~

Compilar el kernel

~~~bash
host $  make -j2 linux ARCH=um SUBARCH=x86_64    # -j numero de cores que se utilizan para compilar el kernel
~~~

Instalar (opcional)
~~~bash
host $ sudo ln -s $(pwd)/vmlinux   /usr/bin/vmlinux
~~~

Descargar el rootfs de este repositorio: **alpine.3.14.rootfs.ext4**, como su nombre lo indica es un alpine v3.14.

***

Modulos del kernel (drivers), por simplicidad para la practica, asumiremos que todos los modulos los ha configurado por defecto o estan compilados de manera estatica, en el caso contrario sigua las intrucciones siguientes: [https://github.com/formatcom/uml](https://github.com/formatcom/uml).

En el enlace anterior tambien se encuentra  como se creo el rootfs.

***

Cositas tecnicas =3 

###  Seguimiento de UML

Cuando se ejecuta, UML consta de un hilo principal del núcleo y varios hilos auxiliares.

- El que tiene el número PID más bajo y utiliza la mayor parte de la CPU suele ser el hilo del **kernel**.
- El subproceso del espacio de usuario de UML.
- El subproceso de E/S asíncrono del controlador UBD [UML Block Device].
- El subproceso auxiliar SIGIO.

***

### Ejecutar una maquina virtual simple

~~~bash
host $ vmlinux umid=uml1 hostname=uml1 ubda=alpine.3.14.rootfs.ext4
~~~
| nombre | descripción |
|--|--|
|  umid| id de la máquina virtual. |
|  hostname |  nombre de la maquina. |
|  udba | root filesystem del s.o.|

***

| usuario | password |
|--|--|
|  root | sin password|

### Apagar la maquina virtual

~~~bash
uml1 $ halt
~~~

### Compartir sistemas de archivos entre máquinas virtuales
No intente compartir sistemas de archivos simplemente iniciando dos UML desde el mismo archivo. Eso es lo mismo que arrancar dos máquinas físicas desde un disco compartido. Dará como resultado la corrupción del sistema de archivos.

#### Usar dispositivos de bloques en capas

Para compartir  un filesystem entre dos o mas maquinas virtuales utilizar copy-on-write (COW) layering capability de el ubd block driver. El controlador admite la superposición de un dispositivo privado de lectura y escritura sobre un dispositivo compartido de solo lectura. Las escrituras de una máquina se almacenan en el dispositivo privado, mientras que las lecturas provienen de cualquier dispositivo.

#### Ejecutar dos máquinas virtuales usando el mismo rootfs

máquina virtual **uml1**
~~~bash
host $ vmlinux umid=uml1 hostname=uml1 ubda=uml1.cow,alpine.3.14.rootfs.ext4
~~~

máquina virtual  **uml2**
~~~bash
host $ vmlinux umid=uml2 hostname=uml2 ubda=uml2.cow,alpine.3.14.rootfs.ext4
~~~

donde **ubda=umlx.cow,alpine.3.14.rootfs.ext4** genera el archivo privado **umlx.cow** por maquina virtual y comparten el rootfs **alpine.3.14.rootfs.ext4**. Es importante no dejar espacio despues de la coma como se mostro en el ejemplo.

***

Con esto terminamos la primera parte y cualquier duda me la pueden dejar en los pull request o por mensajes privados.
