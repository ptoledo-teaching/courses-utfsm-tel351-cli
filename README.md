# CLI - AWS Command Line Interface

Este laboratorio utiliza AWS Command Line Interface (AWS CLI) para descubrir, crear y consultar recursos de AWS desde un intérprete de comandos, o shell. El trabajo comienza en AWS CloudShell, continúa con el aprovisionamiento completo de una instancia EC2 y termina en Ubuntu, donde se comparan credenciales temporales obtenidas mediante un inicio de sesión interactivo con una Access Key de permisos limitados. AWS Management Console se utilizará con la interfaz en inglés.

## Resultados esperados

Al completar el laboratorio, será posible:

1. Reconocer la estructura `aws <service> <operation>` y obtener ayuda sobre comandos y parámetros
2. Utilizar `--query` y los formatos de salida `json`, `text` y `table` para consultar recursos
3. Conservar resultados en variables de shell y reutilizarlos en operaciones posteriores
4. Descubrir una AMI, un tipo de instancia, una VPC y una subnet sin depender de identificadores proporcionados previamente
5. Aprovisionar una instancia EC2 mediante `--generate-cli-skeleton` y `--cli-input-json`
6. Distinguir la instalación de AWS CLI, la autenticación de una identidad y la autorización de sus operaciones
7. Diferenciar una EC2 key pair utilizada por SSH de una Access Key utilizada para llamar a las APIs de AWS
8. Comparar credenciales temporales obtenidas mediante `aws login` con credenciales permanentes almacenadas mediante `aws configure`
9. Relacionar una identidad IAM y sus policies con operaciones permitidas y denegadas

## Preparación previa

Antes del laboratorio, se deben cumplir las condiciones siguientes:

- La cuenta AWS personal debe encontrarse operativa, y la identidad administrativa de uso regular debe estar protegida con MFA y tener asociada la managed policy `AdministratorAccess`
- La cuenta debe mantener su default VPC en São Paulo (`sa-east-1`). Si fue eliminada o modificada, se debe informar antes del laboratorio; su reconstrucción no forma parte de esta actividad
- El computador a utilizar debe tener disponible el cliente OpenSSH

## Actividad

### 1. Iniciar CloudShell y reconocer AWS CLI

#### 1.1 Establecer la identidad y la Región

1. Ingresar a AWS Management Console con la identidad administrativa de uso regular
2. Seleccionar **South America (São Paulo)** y comprobar que la Región corresponda a `sa-east-1`
3. Buscar `CloudShell` en el buscador de AWS Management Console, abrir el servicio y esperar hasta que aparezca la línea de comandos

> Mantener la misma sesión de CloudShell durante la actividad, ya que las variables obtenidas en los pasos siguientes se reutilizarán posteriormente

#### 1.2 Reconocer el comando y su contexto

Ejecutar los comandos siguientes:

```bash
aws --version
aws sts get-caller-identity
aws configure list
```

`aws --version` comprueba el programa instalado. `sts get-caller-identity` muestra la cuenta y la identidad cuyas credenciales se están utilizando. `aws configure list` muestra los valores efectivos y la fuente desde la cual AWS CLI los obtuvo.

CloudShell permite utilizar AWS CLI sin ejecutar previamente `aws login` porque recibe credenciales temporales asociadas con la sesión activa de AWS Management Console. Tampoco es necesario ingresar una Access Key mediante `aws configure`. En esta etapa, `aws configure list` se utiliza solamente para inspeccionar la configuración efectiva y el origen de sus valores; no inicia una sesión ni modifica credenciales. CloudShell también establece como región predeterminada de AWS CLI la Región correspondiente a su pestaña.

La forma general de un comando es:

```text
aws <service> <operation> [parameters] [global options]
```

Revisar los tres niveles de ayuda siguientes. Cada comando abre un visor; presionar `q` para regresar a la línea de comandos.

```bash
aws help
aws ec2 help
aws ec2 describe-vpcs help
```

#### 1.3 Comparar consultas y formatos de salida

Consultar el catálogo completo de tipos de instancia de EC2:

```bash
aws ec2 describe-instance-types \
    --output json
```

El arreglo `InstanceTypes` contiene cientos de entradas y cada una posee numerosos campos. La respuesta completa permite conocer el catálogo, pero resulta poco práctica cuando solamente interesa una familia determinada. Presionar `q` después de constatar la extensión de la respuesta.

La opción `--filters` limita los recursos que el servicio devuelve. En las operaciones de EC2, cada filtro utiliza la forma `Name=<atributo>,Values=<valores aceptados>`. Repetir la consulta solicitando solamente los tipos pertenecientes a la familia T3:

```bash
aws ec2 describe-instance-types \
    --filters "Name=instance-type,Values=t3.small,t3.large*" \
    --output json
```

Comparar la cantidad de elementos recibidos antes y después de incorporar el filtro. Cuando se especifican varios filtros, un recurso debe cumplirlos todos; si un filtro contiene varios valores, puede coincidir con cualquiera de ellos.

Aplicar ahora el mismo mecanismo a la red que se utilizará en el laboratorio. Consultar las VPC disponibles en la región:

```bash
aws ec2 describe-vpcs \
    --output json
```

La operación devuelve todas las VPC visibles en la región. En una cuenta nueva, es posible que la respuesta contenga solamente una. Observar el campo `IsDefault` de cada elemento del arreglo `Vpcs`; este campo permite distinguir la default VPC de las restantes.

Repetir la consulta restringiendo la respuesta a la default VPC:

```bash
aws ec2 describe-vpcs \
    --filters Name=is-default,Values=true \
    --output json
```

La opción `--query` aplica una expresión JMESPath sobre la respuesta estructurada antes de generar la salida. En la expresión siguiente, `Vpcs[]` recorre los elementos del arreglo `Vpcs` y `{VpcId:VpcId,...}` construye para cada uno un objeto que contiene solamente los campos indicados. La opción `--output table` presenta ese resultado como una tabla:

```bash
aws ec2 describe-vpcs \
    --filters Name=is-default,Values=true \
    --query 'Vpcs[].{VpcId:VpcId,Cidr:CidrBlock,Default:IsDefault,State:State}' \
    --output table
```

Las expresiones pueden seleccionar un elemento de un arreglo mediante su posición y acceder a uno de sus campos mediante un punto. Por ejemplo, `[0]` selecciona el primer elemento y `.VpcId` accede a su identificador. La opción `--output text` entrega el valor sin la estructura JSON, lo que facilita su reutilización desde el shell. Para conservar la salida de un comando se utiliza la forma `VARIABLE=$(comando)`.

`--filters` limita los recursos solicitados al servicio, mientras que `--query` selecciona y reorganiza la respuesta recibida. Como procedimiento general, antes de construir una expresión `--query` se debe observar la respuesta JSON, identificar el arreglo que contiene los recursos y seguir los nombres de los campos hasta el valor requerido. Las consultas restantes reutilizan solamente las construcciones presentadas en esta sección: nombres de campos, `.`, `[]`, `[0]` y proyecciones entre llaves.

Construir una nueva consulta que obtenga solamente el `VpcId` de la default VPC. Para ello, reutilizar la operación `aws ec2 describe-vpcs` y el filtro de la default VPC, modificar `--query` para seleccionar su primer elemento y acceder al campo `VpcId`, y utilizar `--output text`. Encerrar el comando completo en `$()` y asignar su resultado a la variable `VPC_ID`.

Una vez ejecutada la asignación, comprobar el valor almacenado:

```bash
echo "$VPC_ID"
```

### 2. Descubrir los parámetros de la instancia

#### 2.1 Localizar y verificar Ubuntu Server 26.04

Las AMI oficiales de Ubuntu son publicadas por Canonical, cuyo AWS account ID es `099720109477`. Una AMI es un recurso regional: una misma publicación de Ubuntu recibe un AMI ID diferente en cada región. Por ello, un identificador obtenido en otra región no puede reutilizarse en `sa-east-1` y la búsqueda debe realizarse en la región donde se creará la instancia. Canonical [documenta los nombres de estas imágenes](https://documentation.ubuntu.com/aws/aws-how-to/instances/find-ubuntu-images/). Para este laboratorio se requiere Ubuntu Server 26.04 para arquitectura x86 de 64 bits y la imagen estable más reciente. Canonical identifica esta arquitectura como `amd64`, mientras que AWS utiliza el nombre `x86_64`; ambos términos se refieren a la misma arquitectura y no restringen la instancia a procesadores fabricados por AMD. El patrón correspondiente es:

```bash
AMI_NAME_PATTERN='ubuntu/images/hvm-ssd-gp3/ubuntu-resolute-26.04-amd64-server-*'
```

En este nombre, `resolute-26.04` identifica la versión de Ubuntu y `amd64-server` indica la arquitectura y el producto. El asterisco representa el serial de publicación, que cambia cada vez que Canonical publica una imagen actualizada. Consultar directamente el catálogo de EC2, restringir el propietario y el nombre, y mostrar los datos necesarios para escoger una publicación:

```bash
aws ec2 describe-images \
    --owners 099720109477 \
    --filters \
        "Name=name,Values=$AMI_NAME_PATTERN" \
        "Name=state,Values=available" \
    --query 'Images[].{ImageId:ImageId,Name:Name,CreationDate:CreationDate}' \
    --output table
```

Comparar `CreationDate`, identificar la publicación más reciente y copiar su `ImageId` en la asignación siguiente:

```bash
AMI_ID=ami-REEMPLAZAR

echo "$AMI_ID"
```

Comprobar la procedencia y las características de la imagen:

```bash
aws ec2 describe-images \
    --image-ids "$AMI_ID" \
    --output json
```

Verificar que el resultado indique:

- `OwnerId` igual a `099720109477`, correspondiente a Canonical
- `State` igual a `available`
- `Architecture` igual a `x86_64`
- `Virtualization` igual a `hvm`
- un nombre que comience con `ubuntu/images/hvm-ssd-gp3/ubuntu-resolute-26.04-amd64-server-`

#### 2.2 Examinar y seleccionar un tipo de instancia

Retomar el catálogo de tipos de instancia, restringirlo a la familia T3 y mostrar sus características principales:

```bash
aws ec2 describe-instance-types \
    --filters "Name=instance-type,Values=t3.*" \
    --query 'InstanceTypes[].{Type:InstanceType,vCPU:VCpuInfo.DefaultVCpus,MemoryMiB:MemoryInfo.SizeInMiB}' \
    --output table
```

La expresión recorre `InstanceTypes[]` y conserva solamente el nombre, la cantidad de vCPU y la memoria. Este catálogo describe las características de los tipos, pero no garantiza que todos puedan utilizarse en la región o en la cuenta.

Seleccionar `t3.small` para disponer de 2 GiB de memoria durante la instalación de AWS CLI:

```bash
INSTANCE_TYPE=t3.small
```

Comprobar que el tipo se ofrece en la Región:

```bash
aws ec2 describe-instance-type-offerings \
    --location-type region \
    --filters "Name=instance-type,Values=$INSTANCE_TYPE" \
    --output table
```

`describe-instance-types` indica que un tipo existe y muestra sus capacidades. `describe-instance-type-offerings` comprueba que se ofrece en una ubicación. Esto todavía no garantiza un lanzamiento: también pueden intervenir permisos, cuotas, direcciones disponibles en la subnet y capacidad transitoria de EC2.

#### 2.3 Seleccionar una subnet de la default VPC

Listar las subnets de la VPC descubierta:

```bash
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,AutoPublicIPv4:MapPublicIpOnLaunch}' \
    --output table
```

Identificar en la tabla una subnet cuyo campo `AutoPublicIPv4` sea `true`, copiar su identificador y asignarlo a la variable:

```bash
SUBNET_ID=subnet-REEMPLAZAR
```

Consultar la subnet seleccionada y conservar su AZ:

```bash
SUBNET_AZ=$(aws ec2 describe-subnets \
    --subnet-ids "$SUBNET_ID" \
    --query 'Subnets[0].AvailabilityZone' \
    --output text)

echo "$SUBNET_ID $SUBNET_AZ"
```

Si no se obtiene una subnet, se debe detener el procedimiento: la configuración prevista no podría asignar automáticamente la dirección IPv4 pública necesaria para SSH. `MapPublicIpOnLaunch` controla esa asignación, pero no determina por sí solo que una subnet sea pública. Las default subnets también poseen una ruta hacia el Internet Gateway de la default VPC.

Confirmar que el tipo seleccionado también se ofrece en la AZ de la subnet:

```bash
aws ec2 describe-instance-type-offerings \
    --location-type availability-zone \
    --filters \
        "Name=location,Values=$SUBNET_AZ" \
        "Name=instance-type,Values=$INSTANCE_TYPE" \
    --output table
```

#### 2.4 Crear el Security Group

Definir el identificador que se utilizará como nombre del Security Group, nombre de la EC2 key pair y valor de la etiqueta `Name` de la instancia. Reemplazar el RUT de ejemplo por el RUT normalizado correspondiente:

```bash
LAB_NAME=tel351-cli-12345678k
echo "$LAB_NAME"
```

Crear un Security Group específico dentro de la VPC y almacenar su ID:

```bash
SG_ID=$(aws ec2 create-security-group \
    --group-name "$LAB_NAME" \
    --description "TEL351 AWS CLI laboratory" \
    --vpc-id "$VPC_ID" \
    --query 'GroupId' \
    --output text)

echo "$SG_ID"
```

Autorizar SSH desde cualquier dirección y comprobar la regla:

```bash
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

aws ec2 describe-security-groups \
    --group-ids "$SG_ID" \
    --output json
```

> El origen `0.0.0.0/0` permite intentar una conexión SSH desde cualquier dirección IPv4. Se utiliza para evitar que un cambio de red interrumpa este laboratorio breve; no representa una configuración recomendada para producción. La autenticación todavía requiere la llave privada, y tanto la instancia como el Security Group deben eliminarse al terminar.

El Security Group nuevo conserva la regla predeterminada que permite todo el tráfico de salida, necesaria para que Ubuntu descargue AWS CLI y alcance los endpoints de AWS.

#### 2.5 Crear la EC2 key pair

Crear una key pair y guardar en un archivo el material de la llave privada entregado por EC2:

```bash
KEY_NAME="$LAB_NAME"
KEY_FILE="$KEY_NAME.pem"

aws ec2 create-key-pair \
    --key-name "$KEY_NAME" \
    --key-type rsa \
    --key-format pem \
    --query 'KeyMaterial' \
    --output text > "$KEY_FILE"

chmod 400 "$KEY_FILE"
ls -l "$KEY_FILE"
```

AWS conserva la llave pública e instala su contenido en la instancia. El archivo `.pem` contiene la llave privada y solo puede obtenerse durante la creación de la key pair; debe mantenerse protegido.

```text
EC2 key pair y archivo .pem → autenticación SSH en Ubuntu
Access Key de IAM            → autenticación de solicitudes a las APIs de AWS
```

### 3. Crear la instancia mediante una entrada JSON

#### 3.1 Generar y simplificar la estructura base

Generar la estructura base, o skeleton, de la entrada completa de `run-instances` y revisarla:

```bash
aws ec2 run-instances --generate-cli-skeleton input > run-instances.json
less run-instances.json
```

Presionar `q` para cerrar el visor. El skeleton permite descubrir la forma de una operación con muchos parámetros sin construir una línea extensa ni memorizar toda su estructura. Contiene numerosas opciones que no corresponden a este caso y debe simplificarse antes de utilizarlo.

Conservar el skeleton completo como referencia y abrir un nuevo `run-instances.json` para construir una versión simplificada:

```bash
mv run-instances.json run-instances-full.json
nano run-instances.json
```

Ingresar la estructura siguiente y guardar los cambios con `Ctrl+O`, `Enter` y `Ctrl+X`:

```json
{
  "ImageId": "REEMPLAZAR_AMI_ID",
  "InstanceType": "REEMPLAZAR_INSTANCE_TYPE",
  "KeyName": "REEMPLAZAR_KEY_NAME",
  "MinCount": 1,
  "MaxCount": 1,
  "SubnetId": "REEMPLAZAR_SUBNET_ID",
  "SecurityGroupIds": [
    "REEMPLAZAR_SECURITY_GROUP_ID"
  ],
  "TagSpecifications": [
    {
      "ResourceType": "instance",
      "Tags": [
        {
          "Key": "Name",
          "Value": "REEMPLAZAR_LAB_NAME"
        }
      ]
    }
  ]
}
```

Reemplazar cada marcador por el valor mostrado por la variable correspondiente:

```bash
echo "$AMI_ID"
echo "$INSTANCE_TYPE"
echo "$KEY_NAME"
echo "$SUBNET_ID"
echo "$SG_ID"
echo "$LAB_NAME"
```

El archivo JSON recibe valores literales: referencias como `$AMI_ID` no se expanden dentro de `--cli-input-json`. La configuración no incluye `IamInstanceProfile`, por lo que la instancia comenzará sin credenciales AWS asociadas a un role.

Antes de continuar, comprobar que el archivo contenga JSON válido:

```bash
python3 -m json.tool run-instances.json > /dev/null
```

El comando no muestra una respuesta cuando el documento es válido. Si informa una línea y una columna, se debe corregir la sintaxis antes de invocar EC2.

#### 3.2 Ejecutar la operación y esperar el estado disponible

Crear la instancia y conservar el ID entregado por la respuesta:

```bash
INSTANCE_ID=$(aws ec2 run-instances \
    --cli-input-json file://run-instances.json \
    --query 'Instances[0].InstanceId' \
    --output text)

echo "$INSTANCE_ID"
```

La respuesta de `run-instances` confirma que EC2 aceptó la solicitud, pero la instancia cambia de estado de manera asincrónica. Se utilizan comandos de espera, o waiters, para detener el shell hasta que alcance los estados requeridos:

```bash
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"
```

#### 3.3 Consultar la instancia

Consultar la instancia creada y reconocer la estructura de la respuesta:

```bash
aws ec2 describe-instances \
    --instance-ids "$INSTANCE_ID" \
    --output json
```

En la respuesta anterior, identificar la ruta que comienza en el arreglo `Reservations`, continúa por el primer elemento de `Instances` y termina en `PublicIpAddress`. Construir una consulta que entregue ese campo como texto y lo almacene en la variable `PUBLIC_IP`. Comprobar el resultado mediante:

```bash
echo "$PUBLIC_IP"
```

La variable debe contener una dirección utilizable. Si `PUBLIC_IP` muestra `None`, no se debe continuar con SSH; se deben revisar la subnet seleccionada y el estado de la instancia.

### 4. Conectarse desde Windows Terminal

#### 4.1 Descargar y proteger la llave privada

Obtener en CloudShell la ruta absoluta del archivo:

```bash
realpath "$KEY_FILE"
```

En el menú **Actions** de CloudShell, seleccionar **Download file**, ingresar la ruta absoluta mostrada y descargar el archivo en Windows. Mantener abierta la sesión de CloudShell para la limpieza posterior.

Abrir una pestaña de PowerShell en Windows Terminal, acceder al directorio de descargas y reemplazar los dos valores de ejemplo:

```powershell
cd "$HOME\Downloads"

$KeyFile = ".\tel351-cli-12345678k.pem"
$PublicIp = "REEMPLAZAR_IP_PUBLICA"

icacls $KeyFile /reset
icacls $KeyFile /grant:r "$($env:USERNAME):(R)"
icacls $KeyFile /inheritance:r
```

Los permisos aplicados mediante `chmod` pertenecían al sistema de archivos de CloudShell y no se transfieren como permisos de Windows. `icacls` restringe la copia local para que OpenSSH no rechace una llave accesible por otras identidades del equipo.

#### 4.2 Ingresar a Ubuntu mediante SSH

Establecer la conexión con la instancia:

```powershell
ssh -i $KeyFile "ubuntu@$PublicIp"
```

Confirmar la huella del host cuando OpenSSH la solicite. El usuario `ubuntu` y la llave `.pem` autentican el acceso al sistema operativo; todavía no entregan una identidad para las APIs de AWS.

### 5. Instalar AWS CLI dentro de Ubuntu

Comprobar primero que el programa no se encuentra instalado:

```bash
aws --version
```

La AMI seleccionada no incluye AWS CLI v2 y el resultado esperado es `aws: command not found`. Instalar la versión vigente mediante el [instalador oficial para Linux](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html):

```bash
curl -fsSL https://awscli.amazonaws.com/v2/install.sh | sudo bash -s -- --system

hash -r
command -v aws
aws --version
```

Comprobar que la versión indicada comience con `aws-cli/2.` y sea `2.32.0` o posterior, requisito de `aws login`.

Aunque el programa ya está instalado, la instancia todavía no posee credenciales. Comprobarlo mediante los comandos siguientes:

```bash
aws configure list
aws sts get-caller-identity
```

`sts get-caller-identity` debe fallar indicando que no puede localizar credenciales. La progresión observada hasta este momento es:

```text
Programa no instalado
        ↓
AWS CLI instalada
        ↓
AWS CLI sin credenciales
```

### 6. Utilizar credenciales temporales mediante `aws login`

#### 6.1 Iniciar una sesión desde la instancia remota

Iniciar una sesión en modo remoto, disponible desde AWS CLI v2.32.0:

```bash
aws login --remote --region sa-east-1
```

`--remote` se utiliza porque AWS CLI se ejecuta en una instancia sin navegador. El [flujo remoto de AWS Sign-In](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sign-in.html) no abre un nuevo puerto: muestra una URL que debe abrirse en el navegador de Windows y mantiene el terminal a la espera de un código.

1. Copiar la URL mostrada y abrirla en el navegador donde permanece abierta AWS Management Console
2. Seleccionar la sesión correspondiente a la identidad administrativa de la cuenta utilizada en el laboratorio
3. Copiar el código de autorización mostrado por el navegador
4. Regresar a Windows Terminal, pegar el código y esperar la confirmación

AWS proporciona la managed policy `SignInLocalDevelopmentAccess` como permiso mínimo para este flujo. La identidad administrativa utilizada en el laboratorio posee `AdministratorAccess`, una policy más amplia que ya incluye esas acciones. `aws login` obtiene credenciales temporales para la identidad seleccionada; no concede permisos adicionales sobre los servicios.

#### 6.2 Comprobar la identidad y sus permisos

Comprobar la fuente de las credenciales y la identidad autenticada:

```bash
aws configure list
aws sts get-caller-identity --output table
```

En `aws configure list`, la fuente de las credenciales debe aparecer como `login`. Comparar también `Account` y `Arn` con la cuenta y la identidad administrativa esperadas. Luego, realizar consultas sobre dos servicios:

```bash
aws ec2 describe-instances \
    --max-results 5 \
    --output json

aws s3api list-buckets \
    --query 'Buckets[].Name' \
    --output table
```

Las operaciones funcionan porque las credenciales representan una identidad cuyas policies las autorizan. La autenticación no modifica los permisos que esa identidad ya posee.

#### 6.3 Cerrar la sesión temporal

Cerrar el login y volver a comprobar el estado:

```bash
aws logout
aws configure list
aws sts get-caller-identity
```

`aws logout` elimina las credenciales de login almacenadas en caché. No cierra la sesión de Management Console en el navegador ni elimina otras clases de credenciales. En esta instancia, `sts get-caller-identity` debe volver a fallar por ausencia de credenciales utilizables.

### 7. Crear una identidad de permisos limitados

#### 7.1 Crear el IAM user y su policy

Regresar a AWS Management Console en el navegador:

1. Abrir **IAM → Users → Create user**
2. Utilizar el nombre `tel351-cli-limited` y no habilitar el acceso a Management Console
3. Crear el IAM user sin agregarlo a un grupo ni asociarle una managed policy
4. Abrir el IAM user y seleccionar **Permissions → Add permissions → Create inline policy → JSON**
5. Ingresar la policy siguiente:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBuckets",
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    }
  ]
}
```

6. Guardar la policy como `tel351-cli-list-buckets`

La policy permite listar los buckets de la cuenta y no autoriza operaciones sobre EC2. Esta diferencia producirá un resultado observable después de configurar las credenciales.

#### 7.2 Crear una Access Key

1. Abrir en el IAM user **Security credentials → Access keys → Create access key**
2. Seleccionar el caso de uso **Command Line Interface (CLI)**, confirmar la recomendación mostrada y continuar
3. Crear la Access Key y mantener disponibles el **Access key ID** y la **Secret access key**

La Secret Access Key se muestra una sola vez y debe mantenerse en secreto. No debe compartirse ni aparecer en capturas. En Ubuntu quedará almacenada mediante `aws configure` y se eliminará durante la limpieza del laboratorio.

```text
Access Key ID + Secret Access Key → credenciales AWS del IAM user
Archivo .pem                      → llave privada SSH de la instancia EC2
```

### 8. Configurar credenciales permanentes y comparar permisos

#### 8.1 Almacenar las credenciales en Ubuntu

Regresar a la conexión SSH en Windows Terminal y ejecutar:

```bash
aws configure
```

Ingresar los valores solicitados:

```text
AWS Access Key ID:      Access key ID de tel351-cli-limited
AWS Secret Access Key: Secret access key de tel351-cli-limited
Default region name:   sa-east-1
Default output format: json
```

Comprobar el nuevo contexto:

```bash
aws sts get-caller-identity
aws configure list
ls -l ~/.aws/credentials
grep -E '^\[|^[a-z_]+[[:space:]]*=' ~/.aws/credentials | cut -d= -f1
```

`aws configure list` debe mostrar `shared-credentials-file` como fuente de las credenciales, en contraste con `login` en la sesión temporal. `sts get-caller-identity` identifica al principal y no demuestra que posea permisos amplios sobre otros servicios. La última consulta muestra los nombres de los campos sin imprimir sus valores. La Secret Access Key permanece almacenada en `~/.aws/credentials` y no incluye una expiración automática, por lo que se trata de una credencial de larga duración conservada en la instancia. Normalmente una instancia EC2 debería obtener credenciales temporales mediante un IAM role, pero la configuración manual permite comparar ambos mecanismos sin extender este laboratorio a instance profiles.

#### 8.2 Ejecutar una operación permitida y otra denegada

Ejecutar la operación S3 autorizada por la inline policy:

```bash
aws s3api list-buckets \
    --query 'Buckets[].Name' \
    --output table
```

La tabla puede estar vacía si la cuenta no contiene buckets, pero el comando debe finalizar sin un error de autorización.

A continuación, intentar consultar EC2:

```bash
aws ec2 describe-instances \
    --max-results 5 \
    --output table
```

La solicitud está correctamente firmada y AWS reconoce la identidad IAM, pero la operación debe finalizar con `AccessDenied` o un error equivalente porque sus policies no permiten `ec2:DescribeInstances`.

El recorrido completo separa los componentes involucrados:

```text
AWS CLI
   ↓
Credenciales disponibles
   ↓
Identidad autenticada
   ↓
Policies IAM aplicables
   ↓
Operación permitida o denegada
```

Estar autenticado significa que AWS puede determinar qué identidad realizó la solicitud. No implica que esa identidad esté autorizada para ejecutar cualquier operación.

## Limpieza posterior al laboratorio

Esta limpieza obligatoria se realiza después del bloque de 70 minutos. Se utiliza CloudShell con la identidad administrativa para eliminar los recursos de la cuenta y Windows Terminal para eliminar la copia local de la llave.

### 1. Terminar la instancia

Regresar a la sesión de CloudShell correspondiente a `sa-east-1`. Si la variable ya no existe, volver a definir el nombre común de los recursos y descubrir la instancia mediante el tag utilizado en el laboratorio:

```bash
LAB_NAME=tel351-cli-12345678k

INSTANCE_ID=$(aws ec2 describe-instances \
    --filters \
        "Name=tag:Name,Values=$LAB_NAME" \
        "Name=instance-state-name,Values=pending,running,stopping,stopped" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)
```

Terminar la instancia y esperar hasta que alcance el estado `terminated`:

```bash
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"
```

### 2. Eliminar los recursos de acceso de EC2

Descubrir y eliminar el Security Group después de que haya desaparecido la interfaz de red de la instancia:

```bash
VPC_ID=$(aws ec2 describe-vpcs \
    --filters Name=is-default,Values=true \
    --query 'Vpcs[0].VpcId' \
    --output text)

SG_ID=$(aws ec2 describe-security-groups \
    --filters \
        "Name=group-name,Values=$LAB_NAME" \
        "Name=vpc-id,Values=$VPC_ID" \
    --query 'SecurityGroups[0].GroupId' \
    --output text)

aws ec2 delete-security-group --group-id "$SG_ID"
aws ec2 delete-key-pair --key-name "$LAB_NAME"
```

Si EC2 informa `DependencyViolation` al eliminar el Security Group, se debe esperar hasta que termine la eliminación de la interfaz de red de la instancia y repetir `delete-security-group`.

### 3. Revocar la Access Key y eliminar el IAM user

Eliminar primero la credencial y la inline policy; IAM no permite eliminar un IAM user que todavía las conserva:

```bash
LIMITED_USER=tel351-cli-limited

ACCESS_KEY_ID=$(aws iam list-access-keys \
    --user-name "$LIMITED_USER" \
    --query 'AccessKeyMetadata[0].AccessKeyId' \
    --output text)

aws iam delete-access-key \
    --user-name "$LIMITED_USER" \
    --access-key-id "$ACCESS_KEY_ID"

aws iam delete-user-policy \
    --user-name "$LIMITED_USER" \
    --policy-name tel351-cli-list-buckets

aws iam delete-user --user-name "$LIMITED_USER"
```

### 4. Eliminar los archivos con material de acceso

Eliminar en CloudShell la llave privada y los archivos de entrada:

```bash
rm -f "$LAB_NAME.pem" run-instances.json run-instances-full.json
```

Eliminar en PowerShell la copia descargada y ajustar el nombre al RUT utilizado:

```powershell
Remove-Item "$HOME\Downloads\tel351-cli-12345678k.pem"
```

Finalmente, comprobar que no permanezcan la instancia, el Security Group, la EC2 key pair, la Access Key, el IAM user ni copias locales de la llave privada. La default VPC, sus subnets y la identidad administrativa existían antes del laboratorio y no deben eliminarse.
