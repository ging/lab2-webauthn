<img  align="left" width="150" style="float: left;" src="https://www.upm.es/sfs/Rectorado/Gabinete%20del%20Rector/Logos/UPM/CEI/LOGOTIPO%20leyenda%20color%20JPG%20p.png">
<img  align="right" width="60" style="float: right;" src="https://www.dit.upm.es/images/dit08.gif" >


<br/><br/>


# Laboratorio 2: Autenticación sin contraseña con WebAuthn y passkeys

*Curso de Identidad Digital, Autenticación y Autorización — Módulo 2: Autenticación robusta y
criptografía web.*

## Objetivos

Durante décadas, la autenticación ha descansado sobre un secreto compartido: una contraseña que el
usuario conoce y que entrega al servidor para demostrar su identidad. El problema de fondo de ese
modelo es que un secreto que el usuario puede teclear es también un secreto que puede entregar, sin
darse cuenta, a quien no debe. Ni el segundo factor por SMS ni los códigos de un solo uso resuelven
el problema, porque también pueden interceptarse en tiempo real. Esta práctica aborda la alternativa
que la industria ha adoptado para cerrar esa vía: la autenticación criptográfica sin contraseña,
basada en el estándar **WebAuthn**.

Al finalizar la práctica, el estudiante será capaz de:

- Explicar por qué WebAuthn es resistente al *phishing* y qué papel juega el "anclaje al origen".
- Configurar en Keycloak una política de acceso sin contraseña (*passwordless*) basada en passkeys.
- Registrar y utilizar una passkey empleando el autenticador virtual del navegador.
- Razonar sobre qué ocurre cuando una credencial se pierde y sobre la recuperación de cuenta.
- Distinguir entre una passkey vinculada al dispositivo y una passkey sincronizada en un teléfono
  móvil.

## Dependencias

Para realizar la práctica, el alumno deberá tener instalado en su ordenador:

- **Docker** y **Docker Compose** ([instalación](https://docs.docker.com/get-docker/)).
- Los puertos **8080** y **9000** libres. Si el Keycloak del Laboratorio 1 sigue en marcha, deténgalo
  antes, ejecutando `docker compose down` en la carpeta de aquel laboratorio.
- Un navegador basado en **Chromium** (Chrome o Edge). El autenticador virtual que se usará es una
  herramienta de las DevTools de Chromium y no está disponible en Firefox ni en Safari.
- Para el apartado 6, que es opcional: un teléfono móvil con bloqueo de pantalla y gestor de
  passkeys (iPhone con el Llavero de iCloud activado, o Android con una cuenta de Google), Bluetooth
  activado en el ordenador y en el teléfono, y conexión a internet en ambos.

La práctica es independiente del Laboratorio 1: el entorno arranca con todo lo necesario ya creado.

## Descripción de la práctica

Para comprender qué se construye conviene entender primero qué se sustituye. Cuando una persona se
autentica con contraseña, un atacante que logre interponerse —por ejemplo, mediante una página de
*login* falsa que reenvía todo en tiempo real al servidor legítimo— puede capturar tanto la
contraseña como el código de un solo uso y reutilizarlos. El factor adicional no ayuda, porque sigue
siendo un dato que el usuario transmite.

**WebAuthn** elimina esa posibilidad cambiando la naturaleza de la prueba. En el momento del
registro, el dispositivo del usuario (el *autenticador*) genera un par de claves criptográficas: la
clave **privada** se queda dentro del autenticador y no sale jamás; la **pública** se entrega al
servidor. Para iniciar sesión, el servidor envía un reto aleatorio que el autenticador **firma** con
la clave privada, y el servidor verifica esa firma con la pública. Nunca viaja un secreto
reutilizable. Y, de forma crucial, la firma incorpora el **origen** (el dominio) del sitio que la
solicita. Conviene subrayarlo, porque suele malinterpretarse: la resistencia al *phishing* no
proviene de la biometría —la huella o el rostro solo desbloquean el autenticador localmente—, sino
de ese anclaje criptográfico al origen.

El conjunto de la tecnología, estandarizado por la FIDO Alliance y el W3C, se conoce como **FIDO2** y
se compone de dos piezas: **WebAuthn**, la interfaz del navegador, y **CTAP**, el protocolo entre el
navegador y el autenticador físico. Las credenciales así creadas se denominan coloquialmente
**passkeys**. Los dos flujos que se practicarán —registro e inicio de sesión— son los siguientes:

```mermaid
sequenceDiagram
  participant N as "Navegador + autenticador virtual"
  participant K as "Keycloak"
  Note over N,K: Registro
  N->>K: Solicitud de registro
  K-->>N: Reto aleatorio (challenge)
  N->>N: Genera par de claves (la privada no sale)
  N->>K: Clave publica + firma del reto
  Note over N,K: Inicio de sesion
  K-->>N: Reto nuevo
  N->>N: Firma con la clave privada (incluye el origen)
  N->>K: Firma
  K->>K: Verifica con la clave publica
```

Queda una dificultad práctica: probar passkeys requiere un autenticador (un sensor de huella, un
chip del dispositivo o una llave física), y no todos los equipos disponen de uno. Para que la
práctica sea reproducible en cualquier ordenador, se utilizará el **autenticador virtual** que
incorporan las DevTools de Chromium, que emula por software un dispositivo FIDO2 real.

---

## 1. Puesta en marcha y autenticador virtual

El fichero `docker-compose.yml` que acompaña a la práctica arranca Keycloak en modo de desarrollo
(`start-dev`) e **importa al arrancar** el fichero `realm/realm-curso.json`. Ese fichero crea un
realm `curso` con un único usuario, `diego` (contraseña `diego`), con permiso para gestionar su
propia cuenta. Para iniciarlo, copie el fichero de configuración y levante el contenedor:

```bash
cp .env.example .env
docker compose up -d
```

Espere a que la columna de estado indique `healthy` antes de continuar:

```bash
docker compose ps
```

El orden de los pasos que siguen es importante, porque el autenticador virtual se activa por ventana
del navegador y no de forma global. Realícelos tal cual:

1. Abra **primero** una ventana de incógnito (Ctrl+Shift+N) y confirme que Keycloak responde en
   `http://localhost:8080/realms/curso/account/`.
2. **Dentro de esa misma ventana**, abra las DevTools (F12) → menú **⋮** → **More tools** →
   **WebAuthn**.
3. Marque **Enable virtual authenticator environment**.
4. Cree un autenticador virtual con Protocol `ctap2`, Transport `usb`, marque **Supports resident
   keys** y **Supports user verification**, y pulse **Add**.

> **Advertencia.** El autenticador virtual es obligatorio: sin él, un equipo sin sensor biométrico ni
> llave física no podría completar la práctica. Y, puesto que se activa por ventana, si las DevTools
> se abren en una ventana distinta de aquella en la que se inicia sesión, esta última no dispondrá del
> entorno y el navegador intentará recurrir a un teléfono móvil mediante código QR. Por eso se abre
> primero la ventana de incógnito y, dentro de ella, las DevTools.

---

## 2. Activación de las passkeys en el realm

Keycloak ofrece dos políticas distintas basadas en WebAuthn: **WebAuthn Policy** y **WebAuthn
Passwordless Policy**. En esta práctica se utiliza la segunda.

Acceda a la consola de administración (`http://localhost:8080/`, credenciales `admin` / `admin`) →
realm `curso` → **Authentication** → pestaña **Policies** → **WebAuthn Passwordless Policy**:

1. Active **Enable Passkeys**.
2. Compruebe que **Discoverable credential** tiene el valor **Required**, que es el valor por
   defecto. Esta opción exige que la credencial sea *descubrible*: que el propio autenticador sepa a
   qué usuario pertenece, de modo que no sea necesario teclear el nombre de usuario para iniciar
   sesión.
3. Pulse **Save**.

Al activar las passkeys aparece además la opción **Passkey Mediation**, que puede dejarse con su
valor por defecto.

El cambio tiene efectos visibles inmediatos. Si se abre ahora la pantalla de inicio de sesión del
realm, bajo el botón habitual figura una opción nueva, **Sign in with Passkey**, y en la consola de
cuenta de cada usuario aparece la posibilidad de registrar una passkey. Ambas cosas se usarán en los
apartados siguientes.

> **Por debajo.** Cada campo de esta pantalla es un parámetro que Keycloak entregará al navegador
> cuando pida crear una passkey, mediante la función `navigator.credentials.create` de la API
> WebAuthn:
>
> - **Discoverable credential** → `residentKey`: si la credencial debe guardar a qué usuario
>   pertenece.
> - **User verification requirement** → `userVerification`: si el autenticador debe verificar a la
>   persona (con un PIN o con biometría) o basta con que alguien lo toque.
> - **Relying party ID** → `rp.id`: el dominio al que quedará ligada la passkey. Vacío significa "el
>   dominio desde el que se accede", aquí `localhost`.
> - **Signature algorithms** → los algoritmos de firma que el servidor acepta (`ES256`, `RS256`).
> - **Authenticator Attachment** → `authenticatorAttachment`: si se admite solo un autenticador
>   integrado en el dispositivo (`platform`), solo uno externo, como una llave USB
>   (`cross-platform`), o cualquiera.
>
> Es decir, la política no es una casilla de "activar passkeys" sin más: es la lista de exigencias
> que el servidor impone a cualquier autenticador que quiera registrarse.

> **Pregunta.** Antes de seguir, una cuestión de intuición: ¿qué considera más seguro, una passkey
> sola o una contraseña acompañada de un código por SMS (es decir, dos factores)? Razone su
> respuesta; se volverá sobre ella al final del apartado 4.

---

## 3. Registro de la passkey

Se registrará una passkey para el usuario `diego`, actuando como lo haría él mismo desde su portal de
cuenta.

1. En la ventana de incógnito, acceda a `http://localhost:8080/realms/curso/account/` e inicie sesión
   con el usuario `diego` y la contraseña `diego` (todavía con el método tradicional).
2. Vaya a **Account security** → **Signing in**. En la sección **Passwordless**, junto a **Passkey**,
   pulse **Set up Passkey**.
3. Keycloak solicita confirmar el registro. Si pide un nombre para identificar la passkey, indique
   uno cualquiera (por ejemplo, `portatil`). El navegador lanzará entonces el diálogo de WebAuthn,
   que el autenticador virtual resolverá automáticamente.
4. Vuelva a la pestaña **WebAuthn** de las DevTools: figurará ahora una credencial, con su
   **Credential ID** y un contador de firmas (**Sign Count**).

Esa credencial que aparece en el panel es la evidencia visual del concepto central de la práctica.
La clave **privada reside dentro del autenticador** —no en Keycloak, que solo guarda la pública— y
está ligada a este dominio concreto: no serviría en ningún otro sitio.

> **Por debajo.** Al pulsar **Set up Passkey** ocurre lo siguiente:
>
> 1. Keycloak genera un **reto** (*challenge*) aleatorio y llama a `navigator.credentials.create`
>    con ese reto, el dominio (`rp.id`), un identificador de Diego y las exigencias de la política.
> 2. El autenticador comprueba la presencia y la identidad de la persona, genera un **par de
>    claves** nuevo para `localhost` y guarda la privada junto con el identificador de Diego.
> 3. El navegador devuelve a Keycloak dos estructuras. El **`clientDataJSON`** contiene el reto y el
>    **origen** de la página (`http://localhost:8080`), añadido por el propio navegador. El
>    **`authenticatorData`** contiene un resumen (*hash*) del dominio, unos indicadores
>    (*flags*: **UP**, la persona estaba presente; **UV**, fue verificada; **BE** y **BS**, la
>    credencial puede sincronizarse y está sincronizada), el **contador** de firmas, el identificador
>    de la credencial y la **clave pública**.
> 4. Keycloak comprueba que el reto es el suyo, que el origen y el dominio son los esperados y que los
>    indicadores cumplen la política. Solo entonces **guarda el identificador de la credencial, la
>    clave pública y el contador**. Nada más.

```mermaid
sequenceDiagram
  participant K as Keycloak
  participant N as Navegador
  participant A as Autenticador
  K->>N: Reto + dominio + usuario + exigencias de la política
  N->>N: Construye el clientDataJSON (reto + origen)
  N->>A: create (dominio, usuario, resumen del clientDataJSON)
  A->>A: Verifica a la persona (UV)
  A->>A: Genera el par de claves y guarda la privada
  A-->>N: authenticatorData (flags, contador, clave pública)
  N-->>K: clientDataJSON + authenticatorData
  K->>K: Comprueba reto, origen, dominio y flags
  K->>K: Guarda el ID, la clave pública y el contador
```

> **Pregunta.** Supóngase que un atacante obtiene una copia completa de la base de datos de
> Keycloak. ¿Qué encontraría en ella sobre la passkey de Diego? ¿Le serviría para suplantarle? Compare
> la situación con la de una contraseña.

> **Pregunta.** El panel muestra un contador de firmas. ¿Para qué puede servirle al servidor
> conocer ese contador?

---

## 4. Inicio de sesión sin contraseña

Comprobado el registro, se verifica que el acceso funciona ya sin contraseña:

1. Cierre la sesión de Diego (menú de usuario, arriba a la derecha → **Sign out**).
2. Acceda de nuevo a `http://localhost:8080/realms/curso/account/`.
3. Pulse **Sign in with Passkey**, sin escribir el nombre de usuario. El autenticador virtual
   resolverá el reto y el acceso se completará sin introducir ninguna credencial.
4. Vuelva a la pestaña **WebAuthn** de las DevTools y observe el contador de firmas de la
   credencial.

Keycloak ha sabido quién era el usuario sin que nadie lo tecleara: se lo ha indicado el propio
autenticador, gracias a que la credencial es descubrible.

> **Por debajo.** Al pulsar **Sign in with Passkey**, Keycloak genera un reto nuevo y llama a
> `navigator.credentials.get` **sin indicar ningún usuario**. El autenticador busca una credencial
> para `localhost`, verifica a la persona y **firma** el reto junto con los datos de la operación.
> Con la firma devuelve el identificador de usuario que guardó en el registro: así sabe Keycloak que
> se trata de Diego. Antes de aceptar, Keycloak hace cuatro comprobaciones:
>
> 1. **Reto:** es el mismo que acaba de generar, de modo que la respuesta no es una repetición de
>    otra anterior.
> 2. **Origen y dominio:** el `clientDataJSON` dice `http://localhost:8080` y el resumen del dominio
>    corresponde a `localhost`.
> 3. **Verificación de usuario:** el indicador **UV** está activo, como exige la política.
> 4. **Firma:** es válida con la clave pública guardada en el registro.
>
> Además, comprueba que el **contador** ha avanzado respecto al último valor conocido.

```mermaid
sequenceDiagram
  participant K as Keycloak
  participant N as Navegador
  participant A as Autenticador
  K->>N: Reto nuevo, sin indicar usuario
  N->>N: Construye el clientDataJSON (reto + origen)
  N->>A: get (dominio, resumen del clientDataJSON)
  A->>A: Busca una credencial de localhost y verifica a la persona
  A->>A: Firma el authenticatorData y el resumen del clientDataJSON
  A-->>N: Firma + authenticatorData + identificador de usuario
  N-->>K: Firma + clientDataJSON + authenticatorData + usuario
  K->>K: 1 reto, 2 origen y dominio, 3 UV, 4 firma con la clave pública
  K->>K: Contador mayor que el anterior, acceso concedido
```

> **Pregunta.** Acaba de iniciar sesión sin escribir ni el usuario ni la contraseña. ¿En qué se
> diferencia esto de un doble factor de autenticación (2FA), en el que primero se teclea la
> contraseña y después se introduce un código? Relaciónelo con las dos políticas que ofrecía Keycloak
> en el apartado 2: **WebAuthn Policy** y **WebAuthn Passwordless Policy**.

> **Pregunta.** Supóngase que Diego recibe un correo con un enlace a una copia exacta de esta
> pantalla de inicio de sesión, alojada en un dominio controlado por un atacante. ¿Qué ocurriría si
> Diego iniciara sesión allí con su contraseña? ¿Y si intentara hacerlo con su passkey? Razone la
> respuesta a partir de lo que contiene la firma. ¿Mantiene la respuesta que dio en el apartado 2?

---

## 5. Verificación: eliminar la credencial

Toda credencial puede perderse o comprometerse, y conviene observar qué ocurre entonces. Elimine la
passkey del autenticador y compruebe el comportamiento:

1. En las DevTools → WebAuthn → elimine la credencial del panel.
2. Cierre la sesión e intente acceder de nuevo con **Sign in with Passkey**: la operación no puede
   completarse.
3. Acceda ahora como Diego con usuario y contraseña tradicionales: funciona sin problema.
4. En la consola de cuenta de Diego → **Signing in**, observe que la passkey **sigue figurando** como
   registrada.

Lo que se ha inutilizado es *un medio* de demostrar la identidad de Diego, no su cuenta ni sus
permisos. El modelo configurado es híbrido: retirar la passkey no deja a nadie fuera mientras la
contraseña siga vigente. Y Keycloak no tiene forma de saber que la clave privada ha desaparecido:
para el servidor, la credencial sigue existiendo. Esto conecta con el problema más delicado de la
autenticación sin contraseña, que es la recuperación de cuenta.

> **Por debajo.** En WebAuthn la comunicación siempre la inicia el servidor: el autenticador solo
> habla cuando se le pide crear una credencial o firmar un reto. No existe ningún mensaje con el que
> el autenticador avise de que ha perdido o borrado una clave. Por eso, para Keycloak, la credencial
> de Diego sigue existiendo, y seguirá así hasta que alguien la elimine de la cuenta.

> **Pregunta.** ¿Qué ocurre si el usuario pierde el dispositivo que contiene la passkey? ¿Qué
> alternativas existen para que no pierda el acceso a su cuenta?

---

## 6. Una passkey en el teléfono móvil (opcional)

Hasta ahora, la passkey ha vivido en un autenticador simulado y **vinculado** a un único
dispositivo: al borrarlo, la credencial desapareció para siempre. En la práctica, la mayoría de las
personas guardan sus passkeys en el teléfono, dentro del gestor del fabricante (el Llavero de
iCloud en un iPhone, el Gestor de contraseñas de Google en Android), que las **sincroniza** entre
todos sus dispositivos. En este apartado se registra una segunda passkey para Diego, esta vez en un
teléfono real, mediante el transporte que WebAuthn denomina *hybrid*: el ordenador muestra un código
QR, el teléfono lo escanea y ambos se comunican para completar la operación.

1. **Desactive el autenticador virtual.** En la pestaña **WebAuthn** de las DevTools, desmarque
   **Enable virtual authenticator environment**. Este paso es imprescindible: mientras el entorno
   virtual esté activo, el navegador entrega todas las operaciones de WebAuthn al autenticador
   simulado y nunca ofrece usar el teléfono.
2. Inicie sesión como Diego con usuario y contraseña, vaya a **Account security** → **Signing in** y
   pulse de nuevo **Set up Passkey**.
3. Esta vez, el diálogo del navegador (o del sistema operativo) ofrecerá guardar la passkey en otro
   dispositivo, mediante una opción del tipo "usar un teléfono o tableta". Elíjala: aparecerá un
   código QR.
4. Escanee el código con la cámara del teléfono y acepte. El teléfono le pedirá desbloquearlo (con la
   cara, la huella o el código) y guardará la passkey.
5. En **Signing in**, compruebe qué passkeys tiene ahora registradas Diego.
6. Cierre la sesión, pulse **Sign in with Passkey** y complete el acceso con el teléfono.

> **Por debajo.** El código QR contiene los datos necesarios para que el teléfono y el ordenador
> establezcan un canal cifrado entre ellos. El teléfono anuncia su presencia por **Bluetooth**, y el
> ordenador solo continúa si lo detecta cerca. Los datos de la operación viajan después, cifrados,
> por un servicio de enlace en internet del fabricante. Para Keycloak, el resultado es una passkey
> como cualquier otra, con una diferencia visible en el `authenticatorData`: los indicadores **BE**
> y **BS** aparecen activos (la credencial es sincronizable y está sincronizada), y el contador
> suele valer siempre 0.

```mermaid
sequenceDiagram
  participant O as Ordenador
  participant T as Teléfono
  participant E as Servicio de enlace
  O->>O: Muestra un QR con las claves del canal
  Note over T: Escanea el QR con la cámara
  T-->>O: Anuncio por Bluetooth (prueba de proximidad)
  O->>E: Se conecta al enlace
  T->>E: Se conecta al enlace
  Note over O,E: Canal cifrado de extremo a extremo
  O->>T: Petición WebAuthn (a través del enlace)
  T->>T: Desbloqueo (cara, huella o código) y firma
  T-->>O: Respuesta firmada (a través del enlace)
```

> **Nota.** Al terminar la práctica, elimine la passkey de prueba del gestor de contraseñas del
> teléfono (en iPhone, **Ajustes** → **Contraseñas**; en Android, el **Gestor de contraseñas de
> Google**). Figurará asociada al dominio `localhost`.

> **Pregunta.** Durante el registro, el teléfono y el ordenador han tenido que estar cerca, con el
> Bluetooth activado, aunque la información viaja por internet. ¿Qué tipo de ataque impide esa
> comprobación de proximidad?

> **Pregunta.** Compare la passkey del teléfono con la del autenticador virtual. ¿Dónde reside la
> clave privada en cada caso? ¿Qué ventaja y qué riesgo tiene que una passkey se sincronice a través
> de la nube del fabricante? Relaciónelo con la pregunta del apartado 5.

---

## Resolución de problemas

- El navegador ofrece usar el teléfono móvil mediante un código QR → el entorno virtual no está
  activo en esa ventana. Repita el apartado 1 en la misma ventana en la que están abiertas las
  DevTools.
- **Sign in with Passkey** no encuentra ninguna credencial → es probable que no se marcara
  **Supports resident keys** al crear el autenticador virtual. Revíselo, elimine el autenticador,
  cree uno nuevo y repita el registro del apartado 3.
- No aparece **Sign in with Passkey** ni la opción **Set up Passkey** → la política del apartado 2 no
  se ha guardado. Compruebe que **Enable Passkeys** está activado en el realm `curso`, no en `master`.
- En el apartado 6, el navegador no ofrece usar el teléfono → el entorno virtual sigue activo
  (desmárquelo en la pestaña **WebAuthn**) o el ordenador no tiene el Bluetooth activado.
- `docker compose up` falla porque el puerto 8080 está ocupado → el Keycloak del Laboratorio 1 sigue
  en marcha. Deténgalo con `docker compose down` en la carpeta de aquel laboratorio.
- Para reiniciar el entorno desde cero (se pierde la configuración y las passkeys registradas, y el
  realm se vuelve a importar): `docker compose down -v && docker compose up -d`.
