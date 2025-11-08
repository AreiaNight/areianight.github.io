---
title: Anonforce
layout: post
image: 
    path: /assets/covers/tryhackme.png
autor: AreiaNight
tags: [anonforce, writeup, tryhackme]
category: Pentesting
---

## INFORMACIÓN SOBRE LA MÁQUINA
- **Nombre:** [Anonforce](https://tryhackme.com/room/bsidesgtanonforce)
- **OS:** Linux
- **Flag:** User y Root
- **Objetivo:** Root
- **Dificultad:** Fácil
- **Conocimientos usados:** FTP, hash cracking. 

```
¡Hola a todos! Vengo a traerles otra máquina que es excelente para los nuevos que quieren empezar a juguetear con el pentesitng. Así que, sin más, vamos a empezar con la nueva práctica. 
````

## Anonforce

Primero intenté ingresara a la página web, pero para mi sorpresa no había nada ahí lo que me decía algunas cosas, que esta no sería una máquina que se centre en web (cosa que agradezco, porque me gusta más jugar con ftps y ssh directamente en vez de mandar una reverse shell). 

![](/assets/post/Anon/1.png)

Así que para empezar, voy a confirmar si la máquina está activa, para eso, vamos a usar el `ping` que nos permite mandar paquetes de bytes. El resultado fue interesante, se transmitieron 5 paquetes, 5 paquetes fueron recibidos y ninguno fue perdido. Así que la máquina está activa y conectada a la red, pero no por el puerto 80 que es el comunmente usado para el HTTP. 

![](/assets/post/Anon/2.png)

Ya con estas dos cosas, puedo pasar a mandar un `nmap` para ver que puertos están abiertos. Después del reconocimiento, nmpa me arrojó que habían dos puertos abiertos: el puerto 21 que es para ftp y el 22 que es un puerto ssh. 

![](/assets/post/Anon/3.png)

El ftp o File Transfer Protocol (Protocolo de Transferencia de Archivos) es un protocolo para la transferencia de información entre sistemas conectados a una red TCP (siglas en inglés de Transmission Control Protocol, o sea, Protocolo de Control de Transmisión) que se base en la arquitectura cliente-servidor. 

Una característica del protocolo ftp es que nos permite conenctarnos de forma anónima solo sí en las configuraciones está activado, así que vamos a intentar hacerlo y para mi sorpresa estaba activo, de esta forma tenemos cierto acceso a los archivos dentro de la máquina. 

![](/assets/post/Anon/4.png)

Usando los comandos básicos de Linux podemos navegar hasta el directorio del usuario, que en este caso es *melodias* y justo ahí tenemos la primera flag. Lanzamos un `get` para poder descargarnos el file y con `cat` leémos lo que dice y listo. Ahora falta root. 

![](/assets/post/Anon/5.png)

Mientras exploraba el directorio raíz, me di cuenta que había un directorio llamado 'notread' y cuando alguien dice que no haga algo, eso significa que debo hacerlo. Así que entramos a dicho directorio que por los permisos podía hacerlo. 

![](/assets/post/Anon/6.png)

Dentro de este directorio hay dos archivos, aquí pasó algo extraño, al inicio no me dejaba bajar uno de los files, pero después de un rato me lo permitió, así que no se desesperen si es que les pasa lo mismo. 

![](/assets/post/Anon/7.png)

Con los archivos ya en mi máquina, not que tiene terminaciones curiosas que son pgp y asc. Primero que nada, vamos a ver que son estas dos terminaciones. El PGP (Pretty Good Privacy) es un sistema de cifrado que combina métodos de criptografía de clave pública y clave privada para proteger la información digital, especialmente correos electrónicos y archivos confidenciales. Por otro lado el file asc puede utilizarse para el cifrado o para la comunicación segura. En este caso, el documento contiene un texto formado exclusivamente por caracteres ASCII que es la llave para desencriptar el PGP. 

Con esto listo, hay que desencriptarlo y para mi desgracia deberé usar John para hacerlo. Tengo mis malas experiencias con la herramienta, jamás la hago funcionar como debería, así que solo me queda aguantarme y tratar de hacerla funcionar. Pero primero debos saber la clave para desencriptar el archivo pgp, así que llamaos al John con el diccionario de rockyou para poder saber cual es esta palabra clave y así tener acceso al pgp. 

```
# Extraer el hash para crackear con John
gpg2john private.asc > hash.txt

# Crackear con rockyou
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![](/assets/post/Anon/8.png)

Vemos que la contraseña es xbox360, así que ahora solo hay que meter la palabra clave en pgp, así que simplemente lo meteos y ya tenemos el file desencriptado. Lanzo un cat para poder leer que hay en el file y enseguida reconozco qué es, es un archivo shadows y esto es súper mega importante, ya que contiene los hashes para decifrar las contraseñas del file passwd de linux. 

![](/assets/post/Anon/9.png)

Ahora simplemente me meto de nuevo a la máquina con ftp, para mi fortuna el archivo de passwd tiene los persmisos necesarios para que me lo pueda traer a mi máquina, así que lo hago y con los archivos shadows y passwd en mi máquina, toca desencriptar y crackear las contraseñas. Así que para eso volvemos a usar John. Aquí tuve un muuuuy mal rato tratando de hacerlo funcionar (como siempre). Así que después de varios intentos, una lloradita y un helado después, por fin logré hacer todo esto funcionar. Pero yo sé que quieres saber qué hice, así que vamos paso a paso. 

Primero hay que "unshadow" el archivo shadow, que es darle formato porque por si solos no podemos hacer mucho. Para eso usamos el comando: 

```
unshadow shadows > unshadowerd.txt
```

Ya con esto, podemos empezar con mi pesadilla personal. John. 

![](/assets/post/Anon/10.png)

Después de dos horas tratando de que John trabajara decentemente (no es broma, no tengo idea de porque John me falla tanto), al fin podemos ver lo que yo necesitaba, la contraseña de root. 

![](/assets/post/Cheese/11.png)

Ahora solo tengo acceder a root por ssh usando las credenciales que hemos adquirido. Logramos hacer el logging y solo nos queda navegar hasta su directorio para extraer la flag correspondiente. 

![](/assets/post/Cheese/13.png)

¡Y listo! ¡Hemos completado la máquna! La verdad no esperaba que tuviese que usar John para este tipo de cosas, sinceramente lo evito justo por los probleas que luego me causa, pero es una buena práctica de vez en cuando. Ahora si, ¡nos vemos!