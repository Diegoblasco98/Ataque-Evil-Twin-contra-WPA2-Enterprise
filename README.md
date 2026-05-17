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
