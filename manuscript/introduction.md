# Creating HTML Reports in PowerShell

Por Don Jones

Diseño de portada por Nathan Vonnahme

---

Aprenda a utilizar correctamente ConvertTo-HTML para producir informes HTML de varias secciones y bien formados, pero luego vaya más allá con un módulo EnhancedHTML personalizado. Produaca informes hermosos, codificados por colores, dinámicos y con multi-secciones de forma fácil y rápida. Escrito por Don Jones.

---

Esta guía se publica bajo la licencia Creative Commons Attribution-NoDerivs 3.0 Unported. Los autores le animan a redistribuir este archivo lo más ampliamente posible, pero le solicitan que no modifique el documento original.

**Descargar el código**  El módulo EnhancedHTML2 mencionado en este libro puede encontrarse en [PowerShell Gallery](https://www.powershellgallery.com/packages/EnhancedHTML2/). Esa página incluye instrucciones de descarga. PowerShellGet es necesario, y se puede obtener de PowerShellGallery.com

**¿Ha sido útil este libro?** El (los) autor (es) le pide (n) que haga una donación deducible de impuestos (en los EE.UU., consulte sus leyes si vive en otro lugar) de cualquier cantidad a [The DevOps Collective](https://devopscollective.org/donate/) para apoyar su trabajo.

**Revise las actualizaciones!** Nuestros ebooks se actualizan a menudo con contenido nuevo y corregido. Los hacemos disponibles de tres maneras::

* Nuestra rama principal [GitHub organization](https://github.com/devops-collective-inc), con un repositorio para cada libro. Visite https://github.com/devops-collective-inc/
* Nuestra [GitBook page](https://www.gitbook.com/@devopscollective), donde puede navegar por los libros en línea, o descargarlos en formato PDF, EPUB o MOBI. Utilizando el lector en línea, puede saltar a capítulos específicos. Visite https://www.gitbook.com/@devopscollective
* En [LeanPub](https://leanpub.com/u/devopscollective), donde se pueden descargar como PDF, EPUB, o MOBI (login requerido), y "comprar" los libros haciendo una donación a DevOps. También puede elegir recibir notificaciones de actualizaciones. Visite https://leanpub.com/u/devopscollective

GitBook y LeanPub generan la salida del formato PDF ligeramente diferente, por lo que puede elegir el que prefiera. LeanPub también le puede notificar cada vez que liberamos alguna actualización. Nuestro repositorio de GitHub es el principal; los repositorios en otros sitios suelen ser sólo espejos utilizados para el proceso de publicación. GitBook normalmente contendrá nuestra última versión, incluyendo algunos bits no terminados; LeanPub siempre contiene la más reciente "publicación liberada" de cualquier libro.
