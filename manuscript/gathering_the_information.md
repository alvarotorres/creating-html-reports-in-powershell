# Recopilación de la información

Soy un gran fan de la programación modular. Gran, gran fan. Con eso en mente, tiendo a escribir funciones que recopilan la información que quiero incluir en mi informe, y normalmente haré una función por sección principal de mi informe. Verá dentro de poco cómo es eso de beneficioso. Escribiendo cada función individualmente, hago más fácil de usar esa misma información en otras tareas, y hago más fácil depurar cada una. El truco consiste en que cada salida de función sea un solo tipo de objeto que combine toda la información de esa sección para el informe. He creado cinco funciones, que he pegado en un solo archivo de script. Le mostraré cada una de esas funciones una a la vez, con un breve comentario. Aquí va la primera:

```
function Get-InfoOS {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$True)][string]$ComputerName
    )
    $os = Get-WmiObject -class Win32_OperatingSystem -ComputerName $ComputerName
    $props = @{'OSVersion'=$os.version;
               'SPVersion'=$os.servicepackmajorversion;
               'OSBuild'=$os.buildnumber}
    New-Object -TypeName PSObject -Property $props
}
```

This is a straightforward function, and the main reason I bothered to even make it a function - as opposed to just using Get-WmiObject directly - is that I want different property names, like "OSVersion" instead of just "Version." That said, I tend to follow this exact same programming pattern for all info-retrieval functions, just to keep them consistent. 

```
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
```

Very similar to the last one. You'll notice here that I'm using the -f formatting operator with the RAM property, so that I get a value in gigabytes with 2 decimal places. The native value is in bytes, which isn't useful for me.

```
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
```

Here, I've had to recognize that I'll be getting back more than one object from WMI, so I have to enumerate through them using a ForEach construct. Again, I'm primarily just renaming properties. I absolutely could have done that with a Select-Object command, but I like to keep the overall function structure similar to my other functions. Just a personal preference that helps me include fewer bugs, since I'm used to doing things this way.

```
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
```

Very similar to the function for services. You can probably start to see how using this same structure makes a certain amount of copy-and-paste pretty effective when I create a new function.

```
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
```

The main thing of note here is how I've converted the speed property, which is natively in bytes, to megabytes. Because I don't care about decimal places here (I want a whole number), casting the value as an integer, by using the -as operator, is easier for me than the -f formatting operator. Also, it gives me a chance to show you this technique!

Note that, for the purposes of this book, I'm going to be putting these functions into the same script file as the rest of my code, which actually generates the HTML. I don't normally do that. Normally, info-retrieval functions go into a script module, and I then write my HTML-generation script to load that module. Having the functions in a module makes them easier to use elsewhere, if I want to. I'm skipping the module this time just to keep things simpler for this demonstration. If you want to learn more about script modules, pick up Learn PowerShell Toolmaking in a Month of Lunches or PowerShell in Depth, both of which are available from Manning.com.
