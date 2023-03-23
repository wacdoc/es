Ingrese al almacén de configuración ops.soft, ejecute `./ssl.sh` y se creará una carpeta `conf` en **el directorio superior** .

## preámbulo

SMTP puede comprar directamente servicios de proveedores de la nube, como:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Envío de correo electrónico en la nube de Ali](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

También puede construir su propio servidor de correo: envío ilimitado, bajo costo general.

A continuación, demostramos paso a paso cómo construir nuestro propio servidor de correo.

## Selección de servidor

El servidor SMTP autohospedado requiere una IP pública con los puertos 25, 456 y 587 abiertos.

Las nubes públicas de uso común han bloqueado estos puertos de forma predeterminada y es posible abrirlos emitiendo una orden de trabajo, pero después de todo es muy problemático.

Recomiendo comprar desde un host que tenga estos puertos abiertos y admita la configuración de nombres de dominio inversos.

Aquí, recomiendo [Contabo](https://contabo.com) .

Contabo es un proveedor de alojamiento con sede en Munich, Alemania, fundado en 2003 con precios muy competitivos.

Si elige el euro como moneda de compra, el precio será más económico (un servidor con 8 GB de memoria y 4 CPU cuesta alrededor de 529 yuanes por año, y la tarifa de instalación inicial es gratuita durante un año).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Al realizar un pedido, indique que `prefer AMD` , y el servidor con CPU AMD tendrá un mejor rendimiento.

A continuación, tomaré el VPS de Contabo como ejemplo para demostrar cómo construir su propio servidor de correo.

## Configuración del sistema Ubuntu

El sistema operativo aquí es Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Si el servidor en ssh muestra `Welcome to TinyCore 13!` (como se muestra en la figura a continuación), significa que el sistema aún no se ha instalado. Desconecte ssh y espere unos minutos para volver a iniciar sesión.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Cuando aparece `Welcome to Ubuntu 22.04.1 LTS` , la inicialización está completa y puede continuar con los siguientes pasos.

### [Opcional] Inicializar el entorno de desarrollo

Este paso es opcional.

Para mayor comodidad, puse la instalación y configuración del sistema del software ubuntu en [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Ejecute el siguiente comando para instalar con un solo clic.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Usuarios chinos, utilicen el siguiente comando en su lugar, y el idioma, la zona horaria, etc. se configurarán automáticamente.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo habilita IPV6

Habilite IPV6 para que SMTP también pueda enviar correos electrónicos con direcciones IPV6.

editar `/etc/sysctl.conf`

Modifique o agregue las siguientes líneas

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Continúe con [el tutorial de contabo: Agregar conectividad IPv6 a su servidor](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Edite `/etc/netplan/01-netcfg.yaml` , agregue algunas líneas como se muestra en la figura a continuación (el archivo de configuración predeterminado de Contabo VPS ya tiene estas líneas, simplemente elimínelas).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Luego `netplan apply` para que la configuración modificada surta efecto.

Después de que la configuración sea exitosa, puede usar `curl 6.ipw.cn` para ver la dirección ipv6 de su red externa.

## Clonar las operaciones del repositorio de configuración

```
git clone https://github.com/wactax/ops.soft.git
```

## Genere un certificado SSL gratuito para su nombre de dominio

El envío de correo requiere un certificado SSL para el cifrado y la firma.

Usamos [acme.sh](https://github.com/acmesh-official/acme.sh) para generar certificados.

acme.sh es una herramienta de firma de certificados automatizada de código abierto,

Ingrese al almacén de configuración ops.soft, ejecute `./ssl.sh` y se creará una carpeta `conf` en **el directorio superior** .

Encuentre su proveedor de DNS desde [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , edite `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Luego ejecute `./ssl.sh 123.com` para generar certificados `123.com` y `*.123.com` para su nombre de dominio.

La primera ejecución instalará automáticamente [acme.sh](https://github.com/acmesh-official/acme.sh) y agregará una tarea programada para la renovación automática. Puede ver `crontab -l` , hay una línea como la siguiente.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

La ruta para el certificado generado es algo así como `/mnt/www/.acme.sh/123.com_ecc。`

La renovación del certificado llamará al script `conf/reload/123.com.sh` , edite este script, puede agregar comandos como `nginx -s reload` para actualizar la memoria caché del certificado de las aplicaciones relacionadas.

## Construir servidor SMTP con chasquid

[chasquid](https://github.com/albertito/chasquid) es un servidor SMTP de código abierto escrito en lenguaje Go.

Como sustituto de los antiguos programas de servidor de correo como Postfix y Sendmail, chasquid es más simple y fácil de usar, y también es más fácil para el desarrollo secundario.

Ejecute `./chasquid/init.sh 123.com` se instalará automáticamente con un clic (reemplace 123.com con su nombre de dominio de envío).

## Configurar firma de correo electrónico DKIM

DKIM se utiliza para enviar firmas de correo electrónico para evitar que las cartas se traten como spam.

Después de que el comando se ejecute correctamente, se le pedirá que configure el registro DKIM (como se muestra a continuación).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Simplemente agregue un registro TXT a su DNS (como se muestra a continuación).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Ver el estado del servicio y los registros

 `systemctl status chasquid` Ver el estado del servicio.

El estado de funcionamiento normal es como se muestra en la siguiente figura

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` o `journalctl -xeu chasquid` pueden ver el registro de errores.

## Configuración inversa del nombre de dominio

El nombre de dominio inverso es para permitir que la dirección IP se resuelva en el nombre de dominio correspondiente.

Establecer un nombre de dominio inverso puede evitar que los correos electrónicos se identifiquen como spam.

Cuando se recibe el correo, el servidor receptor realizará un análisis de nombre de dominio inverso en la dirección IP del servidor de envío para confirmar si el servidor de envío tiene un nombre de dominio inverso válido.

Si el servidor de envío no tiene un nombre de dominio inverso o si el nombre de dominio inverso no coincide con la dirección IP del servidor de envío, el servidor de recepción puede reconocer el correo electrónico como spam o rechazarlo.

Visite [https://my.contabo.com/rdns](https://my.contabo.com/rdns) y configure como se muestra a continuación

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Después de configurar el nombre de dominio inverso, recuerde configurar la resolución directa del nombre de dominio ipv4 e ipv6 al servidor.

## Edite el nombre de host de chasquid.conf

Modifique `conf/chasquid/chasquid.conf` al valor del nombre de dominio inverso.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Luego ejecute `systemctl restart chasquid` para reiniciar el servicio.

## Copia de seguridad de conf en el repositorio de git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Por ejemplo, hago una copia de seguridad de la carpeta conf en mi propio proceso de github de la siguiente manera

Crear un almacén privado primero

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Ingrese al directorio conf y envíelo al almacén.

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Agregar remitente

correr

```
chasquid-util user-add i@wac.tax
```

Puede agregar remitente

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Verifique que la contraseña esté configurada correctamente

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Después de agregar el usuario, `chasquid/domains/wac.tax/users` se actualizará, recuerde enviarlo al almacén.

## DNS agregar registro SPF

SPF (Sender Policy Framework) es una tecnología de verificación de correo electrónico utilizada para prevenir el fraude por correo electrónico.

Verifica la identidad del remitente de un correo verificando que la dirección IP del remitente coincida con los registros DNS del nombre de dominio que dice ser, evitando que los estafadores envíen correos electrónicos falsos.

Agregar registros SPF puede evitar que los correos electrónicos se identifiquen como spam tanto como sea posible.

Si su servidor de nombres de dominio no admite el tipo SPF, simplemente agregue el registro de tipo TXT.

Por ejemplo, el SPF de `wac.tax` es el siguiente

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF para `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Tenga en cuenta que he `include:_spf.google.com` aquí, esto se debe a que configuraré `i@wac.tax` como la dirección de envío en el buzón de correo de Google más adelante.

## Configuración DNS DMARC

DMARC es la abreviatura de (Domain-based Message Authentication, Reporting & Conformance).

Se usa para capturar rebotes SPF (tal vez causados ​​por errores de configuración, o alguien más se hace pasar por usted para enviar spam).

Agregar registro TXT `_dmarc` ,

El contenido es el siguiente

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

El significado de cada parámetro es el siguiente

### p (Política)

Indica cómo manejar los correos electrónicos que no superan la verificación SPF (Sender Policy Framework) o DKIM (DomainKeys Identified Mail). El parámetro p se puede establecer en uno de tres valores:

* ninguno: no se realiza ninguna acción, solo se envía el resultado de la verificación al remitente a través del mecanismo de notificación por correo electrónico.
* Cuarentena: coloque el correo que no haya pasado la verificación en la carpeta de correo no deseado, pero no rechazará el correo directamente.
* rechazar: Rechazar directamente los correos electrónicos que fallan en la verificación.

### fo (Opciones de falla)

Especifica la cantidad de información devuelta por el mecanismo de informes. Se puede establecer en uno de los siguientes valores:

* 0: Informe de resultados de validación para todos los mensajes
* 1: Informar solo los mensajes que fallan en la verificación
* d: Informar solo errores de verificación de nombres de dominio
* s: solo informar fallas de verificación de SPF
* l: Informar solo fallas de verificación de DKIM

### rua y ruf

* rua (URI de informes para informes agregados): dirección de correo electrónico para recibir informes agregados
* ruf (URI de informes para informes forenses): dirección de correo electrónico para recibir informes detallados

## Agregue registros MX para reenviar correos electrónicos a Google Mail

Debido a que no pude encontrar un buzón corporativo gratuito que admita direcciones universales (Catch-All, puede recibir cualquier correo electrónico enviado a este nombre de dominio, sin restricciones en los prefijos), usé chasquid para reenviar todos los correos electrónicos a mi buzón de Gmail.

**Si tiene su propio buzón comercial pagado, no modifique el MX y omita este paso.**

Edite `conf/chasquid/domains/wac.tax/aliases` , configure el buzón de reenvío

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indica todos los correos electrónicos, `i` es el prefijo de la dirección de correo electrónico del usuario remitente creado anteriormente. Para reenviar correo, cada usuario debe agregar una línea.

Luego agregue el registro MX (señalo directamente la dirección del nombre de dominio inverso aquí, como se muestra en la primera línea de la figura a continuación).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Una vez completada la configuración, puede usar otras direcciones de correo electrónico para enviar correos electrónicos a `i@wac.tax` y `any123@wac.tax` para ver si puede recibir correos electrónicos en Gmail.

De lo contrario, verifique el registro de chasquid ( `grep chasquid /var/log/syslog` ).

## Envíe un correo electrónico a i@wac.tax con Google Mail

Después de que Google Mail recibió el correo, naturalmente esperaba responder con `i@wac.tax` en lugar de i.wac.tax@gmail.com.

Visite [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) y haga clic en "Agregar otra dirección de correo electrónico".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Luego, ingrese el código de verificación recibido por el correo electrónico al que fue reenviado.

Finalmente, se puede configurar como la dirección del remitente predeterminada (junto con la opción de responder con la misma dirección).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

De esta forma, hemos completado el establecimiento del servidor de correo SMTP y al mismo tiempo usamos Google Mail para enviar y recibir correos electrónicos.

## Envíe un correo electrónico de prueba para verificar si la configuración es exitosa

Introduzca `ops/chasquid`

Ejecute `direnv allow` to install dependencies (direnv se instaló en el proceso anterior de inicialización de una tecla y se agregó un enlace al shell)

entonces corre

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

El significado de los parámetros es el siguiente

* usuario: nombre de usuario SMTP
* contraseña: contraseña SMTP
* a: destinatario

Puede enviar un correo electrónico de prueba.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Se recomienda usar Gmail para recibir correos electrónicos de prueba para verificar si las configuraciones son exitosas.

### Cifrado estándar TLS

Como se muestra en la figura a continuación, existe este pequeño candado, lo que significa que el certificado SSL se ha habilitado correctamente.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Luego haga clic en "Mostrar correo electrónico original"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Como se muestra en la figura a continuación, la página de correo original de Gmail muestra DKIM, lo que significa que la configuración de DKIM se realizó correctamente.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Verifique Recibido en el encabezado del correo electrónico original y podrá ver que la dirección del remitente es IPV6, lo que significa que IPV6 también se configuró correctamente.
