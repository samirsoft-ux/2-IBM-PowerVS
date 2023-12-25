# IBM Power Virtual Servers Conexión VPN Gateway Appliance On-Premise 🗄️

## 📃 Introducción
Aunque las instancias individuales de Power Virtual Server pueden tener acceso a Internet, actualmente no existe ningún servicio VPN IPsec de sitio a sitio que conecte las subredes de Power Virtual Server con sus redes remotas.Hasta el momento conozco tres formas de conectar el Workspace de Power VS a on-prem:
- VPNaaS de PowerVS
- VPN site-to-site a través de VPC
- VPN Gateway Appliance (Juniper o Virtual Router Appliance)
En esta guía vamos a aprender la tercera forma [(IBM Blog)](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-network-architecture-diagrams#network-reference-architecture-privateipsec).

## Disclaimer
- Los vendors, versiones, availability, entre otras variables que se puedan encontrar durante cualquier proceso de esta guía solo se aplican para los fines prácticos de esta guía. En caso sea necesario modificar estas variables según tus requerimientos.

### Los puntos clave a tener en cuenta antes de empezar con la guía son:
- PowerVS no conoce la información de enrutamiento al rango de direcciones IP de la infraestructura local, por lo que no es posible enviar paquetes desde PowerVS a la red local.
- La puerta de enlace VPN del Gateway Appliance pasa los paquetes a las instalaciones a través del túnel VPN, lo que permite la comunicación de un extremo a otro.
- ((PONER MÁS PUNTOS CLAVE))

A conitnuación se muestra la arquitectura de esta conexión:
<p align="center"><img width="800" src="https://github.com/samirsoft-ux/2-IBM-PowerVS/blob/main/Imagenes/IS-arqui-power.png"></p>
((PONER UNA EXPLICACIÓN DE QUÉ ES LO QUE SE ENTIENDE DE ESTA ARQUITECTURA))
<br />

## 📑 Índice  
((MODIFICAR ESTO))
1. [Pre-Requisitos](#pencil-Pre-Requisitos)
2. [1° Configuración de la VPN site-to-site](#1-Configuración-de-la-VPN-site-to-site)
3. [2° Configuración del Cloud Connection en PowerVS](#2-Configuración-del-Cloud-Connection-en-PowerVS)
4. [3° Configuración del Transit Gateway](#3-Configuración-del-Transit-Gateway)
5. [4° Configuración del prefijo de la VPC](#4-Configuración-del-prefijo-de-la-VPC)
6. [5° Conectar la VPC con el Transit Gateway](#5-conectar-la-vpc-con-el-transit-gateway)
7. [6° Definir la tabla de enrutamiento de entrada en la VPC](#6-Definir-la-tabla-de-enrutamiento-de-entrada-en-la-VPC)
8. [7° Probando la conexión](#7-Probando-la-conexión)
9. [Referencias](#mag-Referencias)
10. [Autores](#black_nib-Autores)
<br />

## :pencil: Pre-Requisitos 
* Contar con una cuenta facturable en <a href="https://cloud.ibm.com/"> ***IBM Cloud®*** </a>.
* Definir la región donde se va a desplegar la infraestructura.
* Tener un Juniper Gateway Appliance ya creado.((El dispositivo de puerta de enlace de IBM Cloud le permite enrutar selectivamente el tráfico de red pública y privada a través de un firewall de nivel empresarial con todas las funciones que funciona con las características de software de VyOS, JunOS o cualquier otro sistema operativo (Bring Your Own Appliance) que usted elegir.))
* ((Al utilizar la interfaz de usuario, la CLI o la API de IBM Cloud, puede seleccionar sus VLAN y, por lo tanto, las subredes asociadas que desea asociar con su dispositivo de puerta de enlace. Asociar una VLAN con un dispositivo de puerta de enlace redirige (o troncaliza) esa VLAN y todas sus subredes a su dispositivo, lo que le brinda control sobre el filtrado, el reenvío y la protección.)(Un dispositivo de puerta de enlace está conectado a dos VLAN de tránsito no extraíbles, una para sus redes públicas y otra privada.))
* Tener un Workspace dentro del servicio de PowerVS con una instancia que solo tenga una subred privada ((SI SE QUISIERA CREAR CON UNA INTERFAZ PÚBLICA HAY UNOS PASOS EXTRAS A REALIZAR.))
<br />

## 1° Creación del Gateway Appliance
```Durante este proceso vas a crear el dispositivo de puerta de enlace que va a permitir enrutar selectivamente el tráfico de red privada a través de un firewall de nivel empresarial JunOS.```

1. Selecciona la ***Search Bar*** dentro escribe la palabra clave ***Gateway*** y seleccionar el servicio ***Gateway Appliance***.
   
2. Una vez cargue la nueva vista da click en el botón "Create" (En caso no aparezca habilitado este botón da click en cualquier de los recuadros de Juniper vSRX o Virtual Router Appliance):

   **Parámetros de creación**
   * El tipo de Vendor debe ser ***Juniper*** junto con la versión por defecto que aparece.
   * La licencia debe ser ***Standard license***
   * Se debe deshabilitar la opción de ***High availability*** ya que para esta guía va a ser suficiente la opción ***Single appliance***.
   * La locación debe ser en ***Dallas*** (cualquiera de las opciones "DALxx") ya que es donde menos latencia existe si se va a utilizar el servicio desde Perú.
   * El pod déjalo por defecto.
   * Las capacidades de computo son las que aparecen por defecto.
   * Agregar la clave SSH de la máquina que se va a conectar por SSH al Gateway Appliance.
   * El perfil de storage son los que están por defecto.
   * La Interfaz de Red debe ser pública y privada.
   * La Redundancia de puertos debe ser ***automática***
   * La Velocidad del Puerto y la Salida Pública deben ser por defecto junto con los distintos Add-ons que aparecen.

3. Finalmente, luego de haber creado la conexión asegurarse que el estado del Gateway sea ***Activa***.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_1.gif>
</p>

   **Notas**
   * La conexión debe ser ***Policy Based***.
   * Esta es la <a href="https://cloud.ibm.com/docs/vpc?topic=vpc-using-vpn"> ***documentación oficial*** </a> en la cual puedes ver un overview de lo que es una Site-to-Site VPN.
   * En el enrutador VPN de la red local, también especifique la subred PowerVS, no la subred de la VPC, para los CIDR del mismo nivel.

## 2° Configuración del Cloud Connection en PowerVS
```Esta configuración es el primer paso para poder establecer la conexión del Power con la VPC ya que se establece que el power tiene que hacer uso de una conexión Direct Link 2.0.```

1. Ingresa a la sección de ***Lista de recursos*** y dentro ubicar el apartado ***Compute***.

2. Seleccionar el ***Workspace*** en donde se va a trabajar y dirigirse a la sección ***Cloud connections***.

3. Dentro darle click al botón "Create connection +".

   **Parámetros de creación**
   * Escribir un nombre para la conexión que haga referencia de donde a donde se está realizando la conexión.
   * Seleccionar una velocidad de 50 Mbps ya que con esta es suficiente para solo probar la conexión una vez terminada toda la guía.
   * Asegurarse que las opciones ***Enable global routing*** y ***Enable IBM Cloud Transit Gateway*** se encuentren habilitadas.
   * Seleccionar el botón "Done editing".
   * Habilitar la opción ***I understand virtual connections must be configured by creating a transit gateway in IBM interconnectivity***.
   * Seleccionar el botón "Continue".
   * En la seccion ***Subnet*** conectar la subnet privada de la instancia creada previamente.

4. Finalmente, luego de haber creado el ***Cloud connection*** asegurarse que el estado sea ***Established***.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_2.gif>
</p>

   **Notas**
   * La conexión debe ser de tipo ***Transit Gateway***.

## 3° Configuración del Transit Gateway
```Esta configuración es el segundo paso para poder establecer la conexión del Power con la VPC ya que se hace uso de la conexión Direct Link 2.0 ya creada para que el Transit Gateway lo reconozca.```

1. Ingresar al ***Navigation Menu*** y dentro dirigirse a la sección ***Interconnectivity***.

2. Dentro de esta sección dirigirse al apartado ***Transit Gateway***.

3. Seleccionar el botón "Create transit gateway".

   **Parámetros de creación**
   * Escribir un nombre para el Transit que haga referencia de donde a donde se está realizando la conexión.
   * Elegir el grupo de recursos de su preferencia.
   * Dentro de la sección ubicación la opción de routung debe de ser ***Local routing*** y la ubicación debe ser en Dallas la misma en donde se encuentra el Workspace de Powervs.
   * Establecer una conexión de tipo ***Direct Link*** y seleccionar la que hemos creado en la configuración anterior.
   * Dejar el nombre por defecto que aparece y seleccionar el botón ***Create***.

4. Finalmente, luego de haber creado el ***Transit Gateway*** asegurarse que el estado de la conexión ***Direct Link*** creada sea ***Attached***.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_3.gif>
</p>

## 4° Configuración del prefijo de la VPC
```Esta configuración permite que la información de enrutamiento al rango de la subnet de la red local se anuncie en el lado del PowerVS a través del Cloud Connection (Direct Link 2.0) que hemos creado entre la VPC y el PowerVS, lo que permite que PowerVS envíe paquetes al rango de la subnet de la red local a la VPC.```

1. Ingresar al ***Navigation Menu*** dentro dirigirse a la sección ***VPC Infraestructure*** y seleccionar el apartado ***VPCs***.

2. Ingresar a la VPC creada anteriormente.

3. Dirigirse al apartado ***Address prefixes***

4. Dentro darle click al botón "Create"

   **Parámetros de creación**
   * En la sección ***IP range*** ingresar la subnet de la red local.
   * En la ubicación seleccionar Dallas 1 que es donde se encuentra la conexión VPN.

5. Finalmente, luego de haber creado el prefijo asegurarse que este se visualice.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_4.gif>
</p>

## 5° Conectar la VPC con el Transit Gateway
```Esta configuración es el último paso para poder establecer la conexión del Power con la VPC ya que se configura la conexión con la VPC desde el Transit Gateway.```

1. Ingresar al ***Navigation Menu*** dentro dirigirse a la sección ***Interconnectivity*** y seleccionar el apartado ***Transit Gateway***.

2. Ingresar al ***Transit Gateway*** creado anteriormente.

3. Dentro darle click al botón "Add connection +"

   **Parámetros de creación**
   * En la sección ***Network connection*** seleccionar VPC.
   * En las opciones ***Connection reach*** dejar la que se habilita por defecto.
   * En la sección ***Region*** elegir Dallas.
   * En la sección ***Available connections*** seleccionar la VPC que creamos anteriormente.
   * Dar click en el botón "Add".

5. Asegurarse que el estado de la conexión sea ***Attached***.

6. Dirigirse a la sección ***Routes***.

7. Seleccionar el botón "Generate Report +"

8. Finalmente, al verificar la información de enrutamiento, podemos ver que la información de la ruta se anuncia desde PowerVS y desde la VPC. Dado que definimos el rango de direcciones IP locales (10.241.0.0/24 en este caso) como un prefijo en la VPC, esa información de ruta también se anuncia desde la VPC. Esto se anuncia a PowerVS, por lo que los paquetes destinados a 10.241.0.0/24 enviados desde PowerVS llegarán a la VPC.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_5.gif>
</p>

## 6° Definir la tabla de enrutamiento de entrada en la VPC
```Por fin el último paso no sé que poner acá pero si llegaste hasta aca ya entendiste el truco campeon ;V.```

1. Ingresar al ***Navigation Menu*** dentro dirigirse a la sección ***VPC Infraestructure*** y seleccionar el apartado ***VPCs***.

2. Ingresar a la ***VPC*** creada anteriormente.

3. Dirigirse a la sección ***Routing tables*** y seleccionar la opción ***Manage routing tables***.

4. Seleccionar el botón "Create +"

   **Parámetros de creación**
   * En la sección ***Ubicación*** dejar por defecto seleccionado la opción de Dallas.
   * Escribir un nombre para la conexión que haga referencia al servicio que se está usando.
   * Asegurarse que esée seleccionada la VPC en la que se está trabajando.
   * Elegir como tipo de tráfico ***Ingress***.
   * Habilitar la opción ***Transit gateway***.
   * Seleccionar el botón "Create routing table".

5. Luego de haber creado la tabla de enrutamiento asegurarse que esta se visualice.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_6.gif>
</p>

6. Ingresar al IBM Cloud Shell

7. Ejecutar el comando 
   ```
   ibmcloud is vpc-routing-table-update <VPC ID> <INGRESS ROUTING_TABLE ID> --accept-routes-from-resource-type-filters vpn_gateway
   ```
   * Con esto la tabla recibe información de enrutamiento de la VPN Gateway y configura la ruta personalizada de ingreso para que el próximo salto de los paquetes, destinados a las instalaciones locales, sea la puerta de enlace de la VPN.
   * Con los comandos anteriores, la tabla de enrutamiento ahora se reescribe automáticamente para seguir el cambio de la dirección IP de VPN Gateway. Una vez que los paquetes llegan a la puerta de enlace de VPN, la puerta de enlace de VPN conoce la ruta a las instalaciones más allá, por lo que puede entregar los paquetes a las instalaciones a través del túnel VPN.

8. Asegurarse que las rutas creadas ejecutando el comando se visualicen en el portal.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_7.gif>
</p>

## 7° Probando la conexión
```Para probar la conexión vamos a ingresar desde una máquina que se encuentre dentro de la red local y a través de un comando TCP/IP vamos a probar si se llega a conectar con una instancia del Workspace de PowerVS(la cual tiene el ip privado 172.16.42.230).```

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_8.gif>
</p>

## :mag: Referencias 
* <a href="https://qiita.com/y_tama/items/50791f4c2256d2801804"> Guía original en Qiita </a>.

## Comentarios del autor original a tener en cuenta
* The following Docs contains a connection configuration using the same concept as this article. https://cloud.ibm.com/docs/vpc topic=vpc-vpn-policy-based-ingress-routing-integration-example
* I used Transit Gateway, but from November 2022, Transit Gateway will charge a metered fee for data transfer even for local type. I think it is also a good idea to connect Cloud Connection (Direct Link 2.0) directly to VPC without using Transit Gateway.
* The route learned from the VPN Gateway cannot be deleted from the GUI, so if you want to delete it, use the following command.
```
ibmcloud is vpc-routing-table-update <VPC ID> <INGRESS ROUTING_TABLE ID> --clean-all-accept-routes-from-filters
```
## :black_nib: Autores 
Italo Silva Public Cloud Perú.
