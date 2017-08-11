# Construyendo el HTML

Voy a abandonar el CmdLet nativo de ConvertTo-HTML que he discutido hasta ahora, En lugar de eso, voy a pedirle que utilice el módulo EnhancedHTML2 que viene con este e-Book. Tenga en cuenta que, a partir de octubre de 2013, se trata de una nueva versión del módulo - es más sencillo que el módulo EnhancedHTML introducido con la edición original de este libro.

Comencemos con el script que utiliza el módulo. Se incluye con este libro como EnhancedHTML2-Demo.ps1, por lo que aquí voy a pegarlo aquí y luego agregare las explicaciones sobre lo que hace cada bit. Tenga en cuenta que no puedo controlar cómo se ve el código en un e-Reader, por lo que es probable que parezca un poco desordenado.

```
#requires -module EnhancedHTML2
<#
.SYNOPSIS
Generates an HTML-based system report for one or more computers.
Each computer specified will result in a separate HTML file; 
specify the -Path as a folder where you want the files written.
Note that existing files will be overwritten.

.PARAMETER ComputerName
One or more computer names or IP addresses to query.

.PARAMETER Path
The path of the folder where the files should be written.

.PARAMETER CssPath
The path and filename of the CSS template to use. 

.EXAMPLE
.\New-HTMLSystemReport -ComputerName ONE,TWO `
                       -Path C:\Reports\ 
#>
[CmdletBinding()]
param(
    [Parameter(Mandatory=$True,
               ValueFromPipeline=$True,
               ValueFromPipelineByPropertyName=$True)]
    [string[]]$ComputerName,

    [Parameter(Mandatory=$True)]
    [string]$Path
)
```

La sección anterior nos dice que se trata de un "script avanzado", lo que significa que utiliza el enlace de CmdLet de PowerShell. Puede especificar uno o más nombres de equipo para los que se genera el informe, y debe especificar una ruta de acceso de carpeta (no un nombre de archivo) para almacenar los reportes finales.

```
BEGIN {
    Remove-Module EnhancedHTML2
    Import-Module EnhancedHTML2
}
```

El bloque BEGIN podría ser eliminado dependiendo de la versión de PowerShell que esté utilizando. Utilizo esta demostración para probar el módulo, así que es importante que descargue cualquier versión antigua de la memoria (si ha cargado el módulo anteriormente) y vuelva a cargar la versión revisada. De hecho, PowerShell v3 y posterior no requerirá la importación si el módulo está correctamente ubicado en `\Documents\WindowsPowerShell\Modules\EnhancedHTML2`.

```
PROCESS {

$style = @"
<style>
body {
    color:#333333;
    font-family:Calibri,Tahoma;
    font-size: 10pt;
}

h1 {
    text-align:center;
}

h2 {
    border-top:1px solid #666666;
}

th {
    font-weight:bold;
    color:#eeeeee;
    background-color:#333333;
    cursor:pointer;
}

.odd  { background-color:#ffffff; }

.even { background-color:#dddddd; }

.paginate_enabled_next, .paginate_enabled_previous {
    cursor:pointer; 
    border:1px solid #222222; 
    background-color:#dddddd; 
    padding:2px; 
    margin:4px;
    border-radius:2px;
}

.paginate_disabled_previous, .paginate_disabled_next {
    color:#666666; 
    cursor:pointer;
    background-color:#dddddd; 
    padding:2px; 
    margin:4px;
    border-radius:2px;
}

.dataTables_info { margin-bottom:4px; }

.sectionheader { cursor:pointer; }

.sectionheader:hover { color:red; }

.grid { width:100% }

.red {
    color:red;
    font-weight:bold;
} 
</style>
"@
```

Eso se llama hoja de estilos en cascada, o CSS. Hay algunas cosas interesantes para sacar destacar:

He colocado toda la sección `<style></ style>` en una cadena [here-string de PowerShell]( https://goo.gl/exzNGQ), y almacenado en la variable $style. Hará que sea fácil referirse a esto en adelante.

Tenga en cuenta que he definido el estilo de varias etiquetas HTML, como H1, H2, BODY y TH. Esas definiciones de estilo listan el nombre de la etiqueta sin un signo anterior de período o hash. Se definen los elementos de estilo que interesan, como el tamaño de la fuente, la alineación del texto, etc. Etiquetas como H1 y H2 ya tienen estilos predefinidos establecidos por su navegador, como su tamaño de fuente. Cualquier cosa que ponga en el CSS reemplazará los valores predeterminados del navegador.

Los estilos también heredan. Todo el cuerpo de la página HTML está contenido dentro de las etiquetas `<BODY></ BODY>`, por lo que cualquier cosa que asigne a la etiqueta BODY en CSS también se aplicará a todo lo que contenga la página. Mi cuerpo establece una familia de fuentes y un color de fuente. Las etiquetas H1 y H2 usarán la misma fuente y color.

También verá las definiciones de estilo precedidas por un punto. Esos se llaman estilos de clase. Son clase de plantillas reutilizables del estilo que se pueden aplicar a cualquier elemento dentro de la página. Los ".paginate" son realmente utilizados por el JavaScript que uso para crear tablas dinámicas. No me gustó la forma en que los botones Prev / Next se veían fuera de la caja, así que modifiqué mi CSS para aplicar estilos diferentes.

Preste mucha atención a .odd, .even, y .red en el CSS. Vera que los utilizo poco a poco.

```
function Get-InfoOS {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$True)][string]$ComputerName
    )
    $os = Get-WmiObject -class Win32_OperatingSystem -ComputerName $ComputerName
    $props = @{'OSVersion'=$os.version
               'SPVersion'=$os.servicepackmajorversion;
               'OSBuild'=$os.buildnumber}
    New-Object -TypeName PSObject -Property $props
}

function Get-InfoCompSystem {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$True)][string]$ComputerName
    )
    $cs = Get-WmiObject -class Win32_ComputerSystem -ComputerName $ComputerName
    $props = @{'Model'=$cs.model;
               'Manufacturer'=$cs.manufacturer;
               'RAM (GB)'="{0:N2}" -f ($cs.totalphysicalmemory / 1GB);
               'Sockets'=$cs.numberofprocessors;
               'Cores'=$cs.numberoflogicalprocessors}
    New-Object -TypeName PSObject -Property $props
}

function Get-InfoBadService {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$True)][string]$ComputerName
    )
    $svcs = Get-WmiObject -class Win32_Service -ComputerName $ComputerName `
           -Filter "StartMode='Auto' AND State<>'Running'"
    foreach ($svc in $svcs) {
        $props = @{'ServiceName'=$svc.name;
                   'LogonAccount'=$svc.startname;
                   'DisplayName'=$svc.displayname}
        New-Object -TypeName PSObject -Property $props
    }
}

function Get-InfoProc {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$True)][string]$ComputerName
    )
    $procs = Get-WmiObject -class Win32_Process -ComputerName $ComputerName
    foreach ($proc in $procs) { 
        $props = @{'ProcName'=$proc.name;
                   'Executable'=$proc.ExecutablePath}
        New-Object -TypeName PSObject -Property $props
    }
}

function Get-InfoNIC {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$True)][string]$ComputerName
    )
    $nics = Get-WmiObject -class Win32_NetworkAdapter -ComputerName $ComputerName `
           -Filter "PhysicalAdapter=True"
    foreach ($nic in $nics) {      
        $props = @{'NICName'=$nic.servicename;
                   'Speed'=$nic.speed / 1MB -as [int];
                   'Manufacturer'=$nic.manufacturer;
                   'MACAddress'=$nic.macaddress}
        New-Object -TypeName PSObject -Property $props
    }
}

function Get-InfoDisk {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$True)][string]$ComputerName
    )
    $drives = Get-WmiObject -class Win32_LogicalDisk -ComputerName $ComputerName `
           -Filter "DriveType=3"
    foreach ($drive in $drives) {      
        $props = @{'Drive'=$drive.DeviceID;
                   'Size'=$drive.size / 1GB -as [int];
                   'Free'="{0:N2}" -f ($drive.freespace / 1GB);
                   'FreePct'=$drive.freespace / $drive.size * 100 -as [int]}
        New-Object -TypeName PSObject -Property $props 
    }
}
```

Las seis funciones anteriores no hacen otra cosa que recuperar datos de una sola computadora (observe que su parámetro -ComputerName se define como `[string]`, aceptando un valor, en lugar de `[string[]]` que aceptaría múltiples). Si no logra entender cómo funciona esto... es probable que tenga que dar un paso atrás!

Para propósitos de formato, usted está viendo que se utiliza el carácter (back tick) (como en –ComputerName y $ComputerName). En PowerShell este carácter funciona como una especie de continuación de línea. Lo señalo porque puede ser fácil perderlo de vista.

```
foreach ($computer in $computername) {
    try {
        $everything_ok = $true
        Write-Verbose "Checking connectivity to $computer"
        Get-WmiObject -class Win32_BIOS -ComputerName $Computer -EA Stop | Out-Null
    } catch {
        Write-Warning "$computer failed"
        $everything_ok = $false
    }
```

Lo anterior es el inicio de mi script de demostración. Se están tomando los nombres de equipo que se pasaron al parámetro `-ComputerName`, procesándolos uno a la vez. Luego se hace una llamada a `Get-WmiObject` como una prueba - si esto falla, no quiero hacer nada con el nombre del equipo en absoluto. El resto de la secuencia de comandos sólo se ejecuta si esa llamada WMI tiene éxito.

```
 if ($everything_ok) {
        $filepath = Join-Path -Path $Path -ChildPath "$computer.html"
```

Recuerde que el otro parámetro de este script es `-Path`. Estoy utilizando `Join-Path` para combinar $Path con un nombre de archivo. Join-Path garantiza el número correcto de barras inversas, de modo que si `-Path` es "C:" o "C:" obtendré una ruta de archivo válida. El nombre de archivo será el nombre del equipo actual, seguido de la extensión .html.

```
        $params = @{'As'='List';
                    'PreContent'='<h2>OS</h2>'}
        $html_os = Get-InfoOS -ComputerName $computer |
                   ConvertTo-EnhancedHTMLFragment @params

```

Aquí está mi primer uso del módulo EnhancedHTML2: Con ConvertTo-EnhancedHTMLFragment. Observe lo que estoy haciendo:

1. Estoy usando un hashtable para definir los parámetros del comando, incluyendo ambos -As List y -PreContent '`<H2>OS</H2>`' como parámetros y sus valores. Esto especifica una salida de estilo de lista (frente a una tabla), precedida por el encabezado "OS" en el estilo H2. Vuelva a mirar el CSS y verá que he aplicado un borde superior a todo el elemento `<H2>`, lo que ayudará a separar visualmente las secciones de mi informe.

2. Estoy ejecutando mi comando Get-InfoOS, pasando el nombre del equipo actual. La salida se canaliza a...

3. ConvertTo-EnhancedHTMLFragment, ConvertTo-EnhancedHTMLFragment, donde se encuentra mi hashtable de parámetros. El resultado será una gran cadena de HTML, que se almacenará en $html\_os.

```
        $params = @{'As'='List';
                    'PreContent'='<h2>Computer System</h2>'}
        $html_cs = Get-InfoCompSystem -ComputerName $computer |
                   ConvertTo-EnhancedHTMLFragment @params 
```

Ese es un ejemplo similar, para la segunda sección de mi informe..

```
        $params = @{'As'='Table';
                    'PreContent'='<h2>&diams; Local Disks</h2>';
                    'EvenRowCssClass'='even';
                    'OddRowCssClass'='odd';
                    'MakeTableDynamic'=$true;
                    'TableCssClass'='grid';
                    'Properties'='Drive',
               @{n='Size(GB)';e={$_.Size}},
               @{n='Free(GB)';e={$_.Free};css={if ($_.FreePct -lt 80) { 'red' }}},
               @{n='Free(%)';e={$_.FreePct};css={if ($_.FreeePct -lt 80) { 'red' }}}}
        
        $html_dr = Get-InfoDisk -ComputerName $computer |
                   ConvertTo-EnhancedHTMLFragment @params
```

OK, ese es un ejemplo más complejo. Echemos un vistazo a los parámetros que estoy pasando a ConvertTo-EnhancedHTMLFragment:

- Como se está produciendo una tabla en lugar de una lista, la salida será en un diseño de tabla columnar (algo como lo que produciría Format-Table, pero en HTML).

- Para mi sección de encabezado, he añadido un símbolo de diamante utilizando la entidad HTML &diams; Creo que se ve bien. Eso es todo.

- Puesto que esto será una tabla, puedo especificar -EvenRowCssClass y -OddRowCssClass. Le doy los valores "even" y "odd", que son las dos clases (.even y .odd) que definí en mi CSS. De esta forma estoy creando el vínculo entre las filas de la tabla y mi CSS. Cualquier fila de la tabla "etiquetada" con la clase "odd" heredará el formato de ".odd" de mi CSS. No se debe incluir el punto al especificar los nombres de clase con estos parámetros. Sólo en el CSS se coloca el punto delante del nombre de la clase.

- `-MakeTableDynamic` se establece en $True, para que se aplique el JavaScript necesario y convertir la salida en una tabla que se pueda ordenar y paginar. Esto requerirá que el HTML final se vincule al archivo JavaScript necesario, pero cubriremos este punto cuando lleguemos allí.

- `-TableCssClass` es opcional, pero lo estoy usando para asignar la clase "grid". Una vez más, si observa el CSS, podrá observar que definí un estilo para ".grid", por lo que esta tabla heredará esas instrucciones de estilo.

- El último es el parámetro `-Properties`. Funciona muy parecido a los parámetros `-Properties` de `Select-Object` y `Format-Table`. El parámetro acepta una lista de propiedades separada por comas. El primero, Drive, ya está siendo producido por `Get-InfoDisk`. Los siguientes tres son especiales: son hashtables, creando columnas personalizadas como loa haría Format-Tabl. Dentro del hashtable, usted puede utilizar las siguientes claves:

  - n (o name, o l, o label) especifica el encabezado de columna. Estoy usando "Size(GB)," "Free(GB)", y "Free(%)" como encabezados de columna.
  
  - e (o expression) es un bloque de secuencia de comandos, que define lo que contendrá la celda de la tabla. Dentro de ella, puede utilizar $\_ para referirse al objeto de entrada. En este ejemplo, el objeto canalizado proviene de `Get-InfoDisk`, por lo que me refiero a las propiedades Size, Free y FreePct del objeto. 
  
  - css (o cssClass) es también un bloque de secuencia de comandos. Mientras que el resto de las claves funcionan igual que lo hacen con Select-Object o Format-Table, css (o cssClass) es exclusivo de ConvertTo-EnhancedHTMLFragment. Acepta un bloque de secuencia de comandos, que se espera que produzca una cadena, o nada. En este caso, estoy comprobando para ver si la propiedad FreePct del objeto en la canalización es menor que 80 o no. Si es así, la salida será la cadena "red". Esta cadena se agregará como una clase CSS de la celda en la tabla. Recuerde que en mi CSS definí la clase ".red" y aquí es donde adjunto esa clase a las celdas de la tabla.
  
  - Como una nota aparte, me doy cuenta de que es tonto establecer un color rojo cuando el porcentaje libre de disco es inferior al 80%. Se trata solo de un ejemplo para jugar. Podría fácilmente tener una fórmula más compleja, como _if ($\_.FreePct -lt 20) { 'red' } elseif ($\_.FreePct -lt 40) { 'yellow' } else { 'green' }_ y entonces habría definido las clases ".red", ".yellow" y ".green" en el CSS.

```
$params = @{'As'='Table';
                          'PreContent'='<h2>&diams; Processes</h2>';
                          'MakeTableDynamic'=$true;
                          'TableCssClass'='grid'}
$html_pr = Get-InfoProc -ComputerName $computer |
                              ConvertTo-EnhancedHTMLFragment @params 

$params = @{'As'='Table';
                          'PreContent'='<h2>&diams; Services to Check</h2>';
                          'EvenRowCssClass'='even';
                          'OddRowCssClass'='odd';
                          'MakeHiddenSection'=$true;
                          'TableCssClass'='grid'}
       
 $html_sv = Get-InfoBadService -ComputerName $computer |
                               ConvertTo-EnhancedHTMLFragment @params 
```

More of the same in the above two examples, with just one new parameter: -MakeHiddenSection. This will cause that section of the report to be collapsed by default, displaying only the -PreContent string. Clicking on the string will expand and collapse the report section.

Notice way back in my CSS that, for the class .sectionHeader, I set the cursor to a pointer icon, and made the section text color red when the mouse hovers over it. That helps cue the user that the section header can be clicked. The EnhancedHTML2 module always adds the CSS class "sectionheader" to the -PreContent, so by defining ".sectionheader" in your CSS, you can further style the section headers.

```
        $params = @{'As'='Table';
                    'PreContent'='<h2>&diams; NICs</h2>';
                    'EvenRowCssClass'='even';
                    'OddRowCssClass'='odd';
                    'MakeHiddenSection'=$true;
                    'TableCssClass'='grid'}
        $html_na = Get-InfoNIC -ComputerName $Computer |
                   ConvertTo-EnhancedHTMLFragment @params
```

Nothing new in the above snippet, but now we're ready to assemble the final HTML:

```
        $params = @{'CssStyleSheet'=$style;
                    'Title'="System Report for $computer";
                    'PreContent'="<h1>System Report for $computer</h1>";
            'HTMLFragments'=@($html_os,$html_cs,$html_dr,$html_pr,$html_sv,$html_na);
                    'jQueryDataTableUri'='C:\html\jquerydatatable.js';
                    'jQueryUri'='C:\html\jquery.js'}
        ConvertTo-EnhancedHTML @params |
        Out-File -FilePath $filepath

        <#
        $params = @{'CssStyleSheet'=$style;
                    'Title'="System Report for $computer";
                    'PreContent'="<h1>System Report for $computer</h1>";
            'HTMLFragments'=@($html_os,$html_cs,$html_dr,$html_pr,$html_sv,$html_na)}
        ConvertTo-EnhancedHTML @params |
        Out-File -FilePath $filepath
        #>
    }
}

}
```

The uncommented code and commented code both do the same thing. The first one, uncommented, sets a local file path for the two required JavaScript files. The commented one doesn't specify those parameters, so the final HTML defaults to pulling the JavaScript from Microsoft's Web-based Content Delivery Network (CDN). In both cases:

- -CssStyleSheet specifies my CSS - I'm feeding it my predefined $style variable. You could also link to an external style sheet (there's a different parameter, -CssUri, for that), but having the style embedded in the HTML makes it more self-contained.

- -Title specifies what will be displayed in the browser title bar or tab.

- -PreContent, which I'm defining using the HTML `<H1>` tags, will appear at the tippy-top of the report. There's also a -PostContent if you want to add a footer.

- -HTMLFragments wants an array (hence my use of @() to create an array) of HTML fragments produced by ConvertTo-EnhancedHTMLFragment. I'm feeding it the 6 HTML report sections I created earlier. 

The final result is piped out to the file path I created earlier. The result:

![image004.png](images/image004.png)

I have my two collapsed sections last. Notice that the process list is paginated, with Previous/Next buttons, and notice that my 80%-free disk is highlighted in red. The tables show 10 entries by default, but can be made larger, and they offer a built-in search box. Column headers are clickable for sorting purposes.

Frankly, I think it's pretty terrific!
