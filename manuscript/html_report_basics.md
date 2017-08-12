# Bases del informe HTML

En primer lugar, entender que PowerShell no se limita a crear informes en HTML. Pero me gusta el HTML porque es flexible, puede ser enviado fácilmente a través de correo electrónico, y se ve mucho mejor que un simple informe de texto sin ningún formato. Pero antes de sumergirnos en esto, necesitamos aclarar un poco cómo funciona el HTML.

Una página HTML es sólo un archivo de texto sin formato, con algo parecido a esto:

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"  "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<title>HTML TABLE</title>
</head><body>
<table>
<colgroup><col/><col/><col/><col/><col/></colgroup>
<tr><th>ComputerName</th><th>Drive</th><th>Free(GB)</th><th>Free(%)</th><th>Size(GB)</th></tr>
<tr><td>CLIENT</td><td>C:</td><td>49</td><td>82</td><td>60</td></tr>
</table>
</body></html>
```

Cuando se interpreta por un navegador, este archivo se representa en la pantalla que aparece en la ventana del navegador. Lo mismo se aplica a los clientes de correo electrónico capaces de mostrar contenido HTML. Mientras que usted, como persona, puede poner obviamente cualquier cosa en el archivo, necesita seguir las reglas que los browsers esperan para obtener la salida deseada.

Una de esas reglas es que cada archivo debe contener uno y un solo documento HTML. Es todo el contenido entre la etiqueta `<HTML>` y la etiqueta `</HTML>` (los nombres de las etiquetas no distinguen entre mayúsculas y minúsculas, y es común verlas en minúsculas como en el ejemplo anterior). Menciono esto porque una de las cosas más comunes que veré a la gente hacer con PowerShell se parecerá a esto:

```
Get-WmiObject -class Win32_OperatingSystem | ConvertTo-HTML | Out-File report.html
Get-WmiObject -class Win32_BIOS | ConvertTo-HTML | Out-File report.html -append
Get-WmiObject -class Win32_Service | ConvertTo-HTML | Out-File report.html -append 
```

"Aaarrrggh," dice mi colon cada vez que veo eso. Básicamente, está diciendo a PowerShell que cree tres documentos HTML completos y los coloque en un solo archivo. Mientras que algunos navegadores (Internet Explorer, por ejemplo) entenderán eso e intentarán mostrar algo, es simplemente incorrecto hacer esto. Una vez que empiece generar esta clase de informes, descubrirá rápidamente que este enfoque es doloroso. No es culpa de PowerShell; Simplemente no está siguiendo las reglas. ¡Por eso esta guía!

Se dará cuenta que el HTML consiste en muchas otras etiquetas, como: `<TABLE>, <TD>, <HEAD>`, y otras más. La mayoría de estas forman parejas, lo que significa que vienen en una etiqueta de apertura como `<TD>` y una etiqueta de cierre como `</TD>`. La etiqueta `<TD>` representa una celda de tabla, y todo entre esas etiquetas se considera el contenido de esa celda.

La sección `<HEAD>` es importante. Lo que hay dentro no es normalmente visible en el navegador. En su lugar, el navegador se centra en lo que hay en la sección `<BODY>`. La sección `<HEAD>` proporciona metadatos adicionales, como el título de la página (lo se muestra en la barra de título o pestaña de la ventana del navegador, no en la página), las hojas de estilo o las secuencias de comandos que se adjuntan a la página, y cosas así. Vamos a hacer algunas cosas impresionantes con la sección `<HEAD>`, confíe en mí.

También notará que este HTML es bastante "limpio", en contraposición, digamos, a la salida HTML de Microsoft Word. Este HTML no contiene información visual incrustada en él, como colores o fuentes. Eso es bueno, porque sigue las buenas prácticas de HTML de separar la información de formato de la estructura del documento. Será decepcionante al principio, porque sus páginas HTML parecerán algo aburridas. Pero vamos a mejorar eso, también.

In order to help the narrative in this book stay focused, I'm going to start with a single example. In that example, we're going to retrieve multiple bits of information about a remote computer, and format it all into a pretty, dynamic HTML report. Hopefully, you'll be able to focus on the techniques I'm showing you, and adapt those to your own specific needs.

In my example, I want the report to have five sections, each with the following information:

- Computer Information

- The computer's operating system version, build number, and service pack version.

- Hardware info: the amount of installed RAM and number of processes, along with the manufacturer and model. 

- An list of all processes running on the machine.

- A list of all services which are set to start automatically, but which aren't running.

- Information about all physical network adapters in the computer. Not IP addresses, necessarily - hardware information like MAC address.

I realize this isn't a universally-interesting set of information, but these sections will allow be to demonstrate some specific techniques. Again, I'm hoping that you can adapt these to your precise needs.
