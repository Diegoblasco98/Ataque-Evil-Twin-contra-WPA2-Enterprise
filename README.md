# Auditoría de Seguridad: Ataque Evil Twin contra WPA2 Enterprise

## 📝 1. Explicación del Ataque
Un ataque de **Evil Twin (Gemelo Malvado)** consiste en desplegar un punto de acceso inalámbrico falso que suplanta la identidad de una red legítima (utilizando el mismo SSID). En entornos empresariales (WPA2/WPA3 Enterprise), este ataque se utiliza para interceptar los intentos de inicio de sesión de los usuarios. 

Cuando una víctima intenta conectarse al punto de acceso falso, el servidor de autenticación modificado (en este caso `hostapd-wpe`) captura el intercambio de desafío/respuesta (**MS-CHAPv2**). Aunque la contraseña no se transmite en texto plano, el hash capturado puede ser extraído y descifrado posteriormente de forma local mediante ataques de fuerza bruta o diccionario, demostrando la debilidad del protocolo frente a contraseñas predecibles.

---

## 🛠️ 2. Guía Paso a Paso del Laboratorio

### Paso 1: Conexión y Configuración del Hardware
En primer lugar, se conecta la antena Wi-Fi **Alfa Network** al sistema mediante el puerto USB, asegurando que el chipset sea reconocido correctamente por la máquina de auditoría (Kali Linux).

```bash
iwconfig
```

<img width="917" height="312" alt="image" src="https://github.com/user-attachments/assets/b9b187f2-3bdb-45fa-9893-9776d8bf0849" />

### Paso 2: Preparación de la Interfaz (Modo Monitor)

Para poder interceptar el tráfico inalámbrico y actuar posteriormente como un punto de acceso, es necesario cambiar el modo de operación de la tarjeta de red de `Managed` a `Monitor`. Antes de realizar este cambio, se deben detener los procesos internos del sistema operativo (como NetworkManager o wpa_supplicant) que puedan generar conflictos o intentar controlar la interfaz de forma simultánea.

```bash
# Eliminar procesos conflictivos que puedan bloquear la interfaz
sudo airmon-ng check kill

# Activar el modo monitor en la interfaz inalámbrica
sudo airmon-ng start wlan0
```
<img width="987" height="482" alt="image" src="https://github.com/user-attachments/assets/cbd9c17c-f469-42f7-ac05-128da47f43fb" />

### Paso 3: Configuración y Despliegue del Punto de Acceso (Evil Twin)

Con la interfaz inalámbrica en modo monitor, se procede a preparar el entorno para levantar el punto de acceso falso. Para este laboratorio se utiliza **`hostapd-wpe`** (Wireless Pwnage Edition), una herramienta modificada que integra un servidor de autenticación interno optimizado para interceptar y extraer de forma automatizada los hashes de las credenciales durante el proceso de saludo.

En primer lugar, es necesario editar el archivo de configuración de la herramienta para asociarlo con nuestra tarjeta de red y definir la identidad de la red simulada. Para ello, se accede al fichero mediante el editor de texto `nano`:

```bash
sudo nano /etc/hostapd-wpe/hostapd-wpe.conf
```
<img width="1067" height="292" alt="image" src="https://github.com/user-attachments/assets/a94dd858-8fd3-4dea-9705-a0f676999589" />
<img width="1132" height="582" alt="image" src="https://github.com/user-attachments/assets/dbc99492-e81a-46b6-b217-4fd6a7dc34a3" />

Una vez guardados los cambios en el archivo de configuración, se procede a inicializar el punto de acceso simulado cargando el fichero modificado. Al ejecutar el servicio, la antena comenzará a emitir la señal con el SSID configurado y el entorno quedará a la escucha de conexiones entrantes.

```bash
# Lanzar el punto de acceso con el motor WPE
sudo hostapd-wpe /etc/hostapd-wpe/hostapd-wpe.conf
```
<img width="687" height="202" alt="image" src="https://github.com/user-attachments/assets/42e97483-5e7a-4dc8-9861-2b5c8a4b2ccb" />

### Paso 4. Captura de Credenciales y Handshake MS-CHAPv2

Una vez configurado y desplegado el punto de acceso falso empleando la herramienta `hostapd-wpe`, se procedió a la monitorización de la interfaz en modo monitor (`wlan0mon`) a la espera de conexiones por parte de los clientes objetivo.

Al intentar asociarse el dispositivo de la víctima a la red clonada, `hostapd-wpe` interceptó con éxito la fase de negociación de identidad y el posterior intercambio de desafío/respuesta correspondiente al protocolo **MS-CHAPv2**.

### 4.1. Datos de la Captura Obtenida

A partir del log generado en tiempo real por el servicio, se han extraído los siguientes identificadores y vectores de autenticación:

* **Fecha y Hora de la Intercepción:** Domingo, 17 de Mayo de 2026, 15:10:07
* **Identidad / Usuario (Identity):** `DiegoBlasco`
* **Dirección MAC del Cliente (STA):** `14:b5:cd:28:58:db`
* **Desafío (Challenge):** `10:a3:4f:f9:3f:18:49:72`
* **Respuesta (Response):** `c8:21:c8:1d:0a:f1:42:6f:8d:f3:77:ae:32:80:b5:ec:a7:d0:e5:45:a0:fb:3a:02`
<img width="1372" height="760" alt="image" src="https://github.com/user-attachments/assets/17cef24a-8624-49d4-85d2-baa57dec7525" />

### Paso 5. Preparación del Hash y Creación del Diccionario de Pruebas

Antes de iniciar el proceso de cracking, es necesario extraer la cadena formateada obtenida en la fase de captura y estructurar un archivo de palabras (*wordlist*) que contenga los candidatos a contraseña.

### 5.1. Almacenamiento del Hash Objetivo

Se extrae la línea correspondiente al formato de la herramienta seleccionada (en este caso, optimizado para Hashcat) y se almacena en un archivo de texto plano denominado `hash.txt`. 

Para generar el archivo desde la terminal se puede emplear el siguiente comando:

```bash
echo "DiegoBlasco::::008bedbe4a528be86c5abcc6d50337df6c0d22fe40377584:f5f9c878a8abdb56" > hash.txt
```
<img width="1318" height="156" alt="image" src="https://github.com/user-attachments/assets/5bd609de-6a87-497d-8112-e6ff92c8e6b4" />

### 5.2. Creación del Diccionario de Contraseñas (`diccionario_test.txt`)

Para llevar a cabo la fase de ruptura de la clave de forma dirigida en este entorno de laboratorio, se procede a la generación de un archivo de texto plano denominado `diccionario_test.txt`. 

Este archivo actúa como nuestra *wordlist* base, conteniendo una selección estructurada de palabras y candidatos a contraseña por línea. El propósito de este diccionario es abastecer al motor de cracking en la siguiente fase, permitiendo verificar si la credencial del usuario `DiegoBlasco` asociada al handshake MS-CHAPv2 interceptado forma parte de los patrones de texto predefinidos.

La correcta disposición de este archivo es un requisito técnico indispensable para optimizar los tiempos de computación durante el ataque por diccionario que se documentará a continuación.

---

*Con los archivos `hash.txt` y `diccionario_test.txt` preparados y ubicados en el directorio de trabajo, el entorno queda listo para ejecutar el proceso de cracking en el punto 6 utilizando Hashcat.*
<img width="1115" height="638" alt="image" src="https://github.com/user-attachments/assets/964937c6-c32c-4f9f-a723-e43a08456c79" />

### Paso 6. Ruptura de Contraseña con Hashcat (Cracking)

En este apartado se detalla el proceso de descifrado del protocolo MS-CHAPv2 mediante el uso de la herramienta **Hashcat**, aprovechando la potencia de computación para contrastar los elementos interceptados con el archivo de contraseñas previamente estructurado.

### 6.1. Comando de Ejecución de Hashcat

Para procesar el hash capturado se utiliza el modo específico de Hashcat diseñado para interceptaciones de red NetNTLMv1 / MS-CHAPv2 (identificado técnicamente como el **modo 5500**). 

El comando empleado en la consola de Kali Linux para iniciar el ataque por diccionario es el siguiente:

```bash
hashcat -m 5500 hash.txt diccionario_test.txt
```
<img width="1770" height="530" alt="image" src="https://github.com/user-attachments/assets/470381df-c63a-4261-8c78-0d0ba9a3e1b2" />
<img width="1193" height="636" alt="image" src="https://github.com/user-attachments/assets/be72a360-15f5-4466-a6e1-d963d7c59c95" />

## 🛡️ 3. Conclusiones del Proyecto

A partir de las auditorías de seguridad realizadas sobre la infraestructura de red inalámbrica corporativa y tras el posterior análisis de las capturas de tráfico obtenidas durante la fase de evaluación, se establecen las siguientes conclusiones técnicas fundamentales:

* **⚙️ Robustez y Arquitectura del Mecanismo de Autenticación:** Se ha validado de manera concluyente la implementación de un entorno de red basado en el estándar **IEEE 802.1X (WPA-Enterprise)**, utilizando el protocolo de desafío y respuesta **MS-CHAPv2**. Esta infraestructura proporciona una capa de seguridad superior en comparación con las redes tradicionales de clave compartida (PSK), ya que individualiza de forma estricta los vectores y parámetros de autenticación para cada usuario dentro de la organización.
* **🔒 Naturaleza y Complejidad Criptográfica de los Vectores NetNTLMv2:** Los intercambios de credenciales capturados generan hashes de tipo **NetNTLMv2**. A diferencia de los hashes NTLM locales y estáticos (frecuentemente extraídos de bases de datos SAM), estos hashes dinámicos dependen de un valor aleatorio o desafío (*challenge*) enviado en tiempo real por el servidor. Esta propiedad matemática neutraliza por completo la efectividad de los ataques de diccionario optimizados mediante el uso de tablas precalculadas (*Rainbow Tables*).
* **💻 Especificidad y Rigor Técnico en la Fase de Cracking:** La fase de validación y estructuración de los datos determinó que el formato de la captura es exclusiva y estrictamente compatible con el **modo 5500 de Hashcat**. 
* **👥 Impacto Crítico del Factor Humano en la Seguridad Global:** Dado que el algoritmo NetNTLMv2 exige una alta carga computacional para cada intento de descifrado, la resolución final de la credencial queda supeditada por completo a la predictibilidad y longitud de la clave elegida por el usuario (en este caso particular, la identidad correspondiente al usuario `Diegoblasco16`). Por consiguiente, se concluye que las políticas de seguridad de la organización deben centrarse de forma prioritaria en el cumplimiento riguroso de contraseñas de alta entropía, puesto que un ataque por fuerza bruta pura resulta inviable bajo arquitecturas de cómputo estándar.
