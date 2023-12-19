# 2-IBM-PowerVS
Aquí voy a crear la guía para el powervs con la VPN

# IBM Power Virtual Servers ☁️ - Conexión VPN On-Premise 🗄️

## 📃 Introducción
Power Systems Virtual Server ***PowerVS*** tiene servicio VPNaaS, pero existen algunas limitaciones, como se describe en la documentación ["Limitaciones VPNaaS de PowerVS"](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-VPN-connections). Es por lo que, en este artículo, vas a aprender a cómo acceder a PowerVS usando una VPN site-to-site que se puede usar en VPC, en lugar de la VPNaaS de PowerVS.

### Los puntos clave a tener en cuenta antes de empezar con la guía son:
- PowerVS no conoce la información de enrutamiento al rango de direcciones IP de la infraestructura local, por lo que no es posible enviar paquetes desde PowerVS a la red local.
- La clave de esta configuración es definir el rango de direcciones IP de la red local como el prefijo de la VPC. Esta definición permite que la información de enrutamiento para el rango de IP local se anuncie a PowerVS a través de conexiones en la nube (Direct Link 2.0) entre la VPC y PowerVS, lo que permite que PowerVS envíe paquetes para el rango de IP local a la VPC.
- Defina una tabla de enrutamiento de entrada en la VPC para que los paquetes de PowerVS a la VPC (destinados a la IP local) se enruten a la puerta de enlace VPN.
- La puerta de enlace VPN pasa los paquetes a las instalaciones a través del túnel VPN, lo que permite la comunicación de un extremo a otro.

A conitnuación se muestra la arquitectura de esta conexión, en esta también se muestra las distintas subnets involucradas tanto del lado de IBM como on-premise:
<p align="center"><img width="800" src="https://github.com/samirsoft-ux/Playbook_Power/blob/main/Imagenes/IS-arqui-power.png"></p>

<br />

## 📑 Índice  
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
* Tener una VPC ya creada la cual no tenga un prefijo de dirección predeterminado ya que en el transcurso de esta guía se le va agregar de forma manual.
* Tener un Workspace dentro del servicio de PowerVS con una instancia que solo tenga una subred privada.
<br />

## 1° Configuración de la VPN site-to-site
```Esta configuración permite la conexión entre la red local(on-premise) con la VPC.```

1. Ingresar al ***Navigation Menu*** dentro dirigirse a la sección ***VPC Infraestructure*** y seleccionar el apartado ***VPNs***.
   
2. Dar click en el botón "Create +".

   **Parámetros de creación**
   * El tipo de VPN debe ser ***Site-to-site gateways***.
   * La locación debe ser en ***Dallas*** ya que es donde menos latencia existe si se encuentra en Perú.
   * Escribir un nombre para el gateway que haga referencia al servicio y donde se encuentra.
   * Elegir el grupo de recursos de su preferencia.
   * Si desea ingrese una etiqueta esta te ayuda a identificar detalles del recurso.
   * Si desea ingrese una etiquetaa de administración de acceso esta te ayuda a aplicar políticas de acceso flexibles en recursos específicos.
   * Elige la VPC que ya está previamente creada.
   * La sección ***Subnet*** debe de estar vacía.
   * El modo debe ser ***Policy-based***.
   * Escribir un nombre para la conexión que haga referencia de donde a donde se está realizando la conexión.
   * Ingrese la ***Peer gateway address***, esta es la dirección IP pública del gateway de la red local.
   * Ingrese un ***Preshared key***, este es la clave configurada en la Peer gateway.
   * Los parámetros de la sección ***Dead peer detection*** dejarlos por defecto.
   * Crear un IKE policy.
   * Crear un IPsec policy.

3. Finalmente, luego de haber creado la conexión asegurarse que el estado de la VPN sea ***Activa***.

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
