---
layout: single
title: XSS ATACK - EL ARTE DEL HACKING WEB P1
date: 0204-01-14
classes: wide
categories:
  - Hacking_Web
tags:
  - Web
---
# DEFINICION DE XSS - CROSS SITE SCRIPTING

XSS es una vulnerabilidad de codigo sufrida cuando un atacante consigue de alguna forma inyectar una secuencia de comandos maliciosos dentro de un sitio web, provocando así el acceso no intencionado por parte del usuario victima a datos y privilegios que se encuentran almacenados en el navegador web al momento del ataque XSS al atacante. Este tipo de ataques surge comunmente por la confianza que brinda la pagina web a los datos que puede ingresar el usuario sin aplicar una correcta sanitizacion del codigo permitiendo que se ingrese cualquier tipo de valor sin restriccion alguna, posteriormente el input si contiene algun comentario en forma de `<SCRIPT>` es muy posible que sea interpretado por el servidor o navegador del usuario devolviendo valores mencionados anteriormente.

# TIPOS DE XSS 

Observando en varias páginas, se suelen mencionar 3 tipos de XSS más comunes. Diría que en 1 de cada 10, tal vez se menciona XSS mutable o algo parecido, pero al obtener el significado, se llega a relacionar con los siguientes tipos de XSS:

- REFLECTED XSS
- STORED XSS
- DOM-BASED XSS


## REFLECTED XSS

Este tipo de vulnerabilidad ocurre cuando el atacante envía código malicioso en la solicitud enviada a la web vulnerable, para que así se refleje en la respuesta de la misma. Al no ser un tipo de ataque persistente, con solo reenviar una solicitud al servidor web debería ser suficiente para que la carga maliciosa desaparezca.

Los 3 tipos de XSS parten desde que no se realiza una correcta sanitización de la entrada de texto proporcionada por el usuario. También puede que se llegue a realizar la correcta sanitización en muchos segmentos de la web, pero no se logren cubrir todas las brechas posibles que puedan existir. Esto se debe a que constantemente se descubren nuevas formas de acceso o existe más creatividad por parte del atacante.

Un ejemplo básico para XSS reflejado sería el siguiente, teniendo en cuenta que no se aplica ningún tipo de sanitización sobre la web.

	`http://webvulnerable.com/filescript.php?test=<script>evilFunction()</script>`

En teoria el comando inyectado en el link provocaria al momento de realizar un click sobre el link correspondiente como output tendriamos una ventana emergente.

![](/assets/images/FOTOS_xss_attack/Reflected-xss.png)


Nace a partir del siguiente valor ingresado y procesado: `<script>alert("Este es un Ejemplo de xss reflejado")</script>` 

# STORED XSS

Este tipo de XSS es casi idéntico al anteriormente mencionado, pero la carga maliciosa se guarda en el servidor, lo que provoca que sea mostrada cada vez que un usuario ingrese a la sección de la web vulnerable a XSS. Un ejemplo claro sería una página web que permita dejar algun tipo de comentario que sera mostrado a mas usuarios y no realice una sanitización correspondiente, tal vez mantenga desactualizado el firewall u algún otro factor que permita realizar este ataque. Entonces, no importa cuántas veces recargues la página; seguirá ejecutando el comando inyectado hasta que sea borrado del servidor.

Me enteré de esto mientras investigaba sobre XSS; resulta que TweetDeck, en su momento, fue sorprendido por un Stored  XSS . Básicamente, lo que hace el código de la imagen es retuitear el mismo script, provocando así un bucle constante y que todos realizaran esta ejecución solo al iniciar sesión en la aplicación.
![](/assets/images/FOTOS_xss_attack/xss-Stored.png)

# DOM XSS

En este concepto de XSS, primero daré un diminuto significado de DOM para entender de mejor forma qué es DOM XSS:

`DOM o Document Object Model es la estructura de un documento HTML, siendo el resultado de etiquetas anidadas una dentro de la anterior, formando de esta manera una estructura de árbol que permitirá acceder a ellas cuando sea necesario hacerlo, presentando archivos XML o HTML.`

Debemos tener en cuenta que el DOM es un conjunto que nos presentaría una página HTML, las cuales son estáticas por defecto, pero se incluye JavaScript para hacer que las páginas sean más dinámicas. Entonces, debemos entender que el código JavaScript es interpretado por el navegador del usuario, según las acciones que vaya realizando el mismo. En este punto, entendemos que DOM XSS se basa en la interpretación del navegador del usuario para poder tener un ataque exitoso. Si el código acepta el input de un usuario y lo transporta a una función sin una sanitización correcta, esto se volvería inseguro.

![](/assets/images/FOTOS_xss_attack/DOM-xss.png)


Los valores que ingresamos se van a la función, también llamada receptor: `document.write`. Tomando así el valor que nosotros proporcionamos y agregándolo a la página HTML, en este punto podríamos intentar ingresar un elemento `<script>`, lo cual podría dar como resultado un DOM XSS exitoso


