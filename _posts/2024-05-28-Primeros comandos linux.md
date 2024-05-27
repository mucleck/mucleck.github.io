---
title: "Comandos basicos en Linux"

categories: [Linux]
tags: [Linux, Comandos]
---

## En este segundo post aprenderemos algunos de los comandos mas basicos para empezar a movernos en este SO

1. ls
    - El comando `ls` nos permite listar los contenidos del directorio actual en el que nos encontramos. Si ejecutamos este comando veremos lo siguiente:
    
      ![](/assets/img/0.png)

      Algunos atributos utiles que podemos añadir para tener mas informacion son los siguientes:
            
       1. `ls -l` Aporta mas detalles de los archivos listados

       ![](/assets/img/1.png)
       
       2. `ls -h` Muestra el tamaño de los archivos

       ![](/assets/img/2.png)

       3. `ls -s` Ordena los archivos 

       ![](/assets/img/3.png)

       4. `ls -r` Invierte el orden dado con el comando `s`

       ![](/assets/img/4.png)

            


2. pwd
    
    El comando `pwd` nos resultara util cuando querramos saber en que ruta nos encontramos. Si lo ejecutamos en el escritorio obtendremos lo siguiente:

    ![](/assets/img/5.png)

3. cd

    Con el comand `cd` podremos dirigrnos a una ruta deseada. Por ejemplo para acceder a la carpeta /etc usariamos el comando `cd /etc` y este nos devolveria lo siguiente:

    ![](/assets/img/6.png)

    Si usamos este comando sin darle ninguna direccion nos llevara a la ruta /home/usuario sin importar donde estemos

    ![](/assets/img/7.png)

    En cambio si queremos ir un paso atras en la direccion de la ruta solo tendremos que poner `cd ../` y esto nos llevara a la ruta anterior. Si usamos esto un par de veces veremos que llegamos a la ruta 0 del sistema. Aqui un ejemplo partiendo desde el escritorio:

    ![](/assets/img/8.png)

4. cp

    El comando `cp` se usa para copiar archivos y directorios. Es como el copiar y pegar de la terminal.

    Uso básico: `cp archivo_origen archivo_destino`
    Copiar un directorio: `cp -r directorio_origen directorio_destino`
    Atributos útiles:

    - `cp r`: Copia recursivamente todos los archivos y subdirectorios.
    - `cp v`: Modo detallado, muestra los archivos que se están copiando.
    - `cp i`: Interactivo, pide confirmación antes de sobrescribir.
    Ejemplo:

5. mv

6. mkdir

7. touch

8. echo

9. cat

10. nano

11. grep

12. tar