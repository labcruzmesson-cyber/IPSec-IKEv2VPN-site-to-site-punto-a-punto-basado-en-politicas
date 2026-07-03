# Reporte Técnico de Migración: VPN Site-to-Site IKEv2 Basada en Políticas

**Asignatura:** Seguridad de Redes
**Estudiante:** Manuel Cruz
**Docente:** Jonathan Rondón
**Fecha:** 29 de junio de 2026

> **Nota:** el direccionamiento no se realizó en base a la matrícula del estudiante; este requerimiento se recordó demasiado tarde y no fue posible rehacer las topologías y videos.

---

## Tabla de Contenidos

- [1. Resumen y Objetivos](#1-resumen-y-objetivos)
- [2. Topología de Red y Direccionamiento](#2-topología-de-red-y-direccionamiento)
- [3. Especificación de Políticas y Parámetros de Seguridad (IKEv2)](#3-especificación-de-políticas-y-parámetros-de-seguridad-ikev2)
- [5. Puntos Críticos Analizados](#5-puntos-críticos-analizados)
- [6. Protocolo de Verificación y Diagnóstico Técnico](#6-protocolo-de-verificación-y-diagnóstico-técnico)

---

## 1. Resumen y Objetivos

Este documento describe la evolución tecnológica y la reestructuración del laboratorio original. Se migra la arquitectura de una VPN de seguridad perimetral tradicional basada en el protocolo heredado **IKEv1** hacia el estándar contemporáneo **IKEv2**. La solución se despliega utilizando una arquitectura basada en políticas (Crypto Maps), manteniendo intacto el direccionamiento IP de la infraestructura física del proyecto original.

### Objetivos del Proyecto

- **Modernización del Canal de Control:** reemplazar el intercambio de fases rígido de IKEv1 por la eficiencia y la seguridad nativa del protocolo IKEv2.
- **Confidencialidad e Integridad Avanzada:** garantizar la protección estricta del flujo de datos entre la LAN de PEER A (`172.16.1.0/24`) y la LAN de PEER B (`172.16.2.0/24`) mediante el uso de algoritmos de cifrado simétrico robustos de extremo a extremo.
- **Coexistencia de Servicios:** asegurar el correcto funcionamiento del enrutamiento estático y el aislamiento criptográfico frente a la traducción de direcciones de red (NAT/PAT).

---

## 2. Topología de Red y Direccionamiento

La topología lógica mantiene el diseño de dos sedes principales interconectadas a través de un gateway WAN que emula un proveedor de servicios de Internet (R-ISP).

| Dispositivo / Sede | Interfaz | Dirección IP | Máscara de Subred | Propósito / Rol |
|---|---|---|---|---|
| PEER A | Gi0/0 | 10.0.0.60 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER A | Gi0/1 | 172.16.1.1 | 255.255.255.0 | Gateway LAN A / NAT Inside |
| PEER B | Gi0/0 | 10.0.0.70 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER B | Gi0/1 | 172.16.2.1 | 255.255.255.0 | Gateway LAN B / NAT Inside |
| R-ISP (Gateway) | N/A | 10.0.0.1 | 255.255.255.0 | Puerta de enlace predeterminada WAN |

---

## 3. Especificación de Políticas y Parámetros de Seguridad (IKEv2)

A diferencia de la suite anterior basada en ISAKMP, la nueva implementación agrupa las directivas criptográficas de forma modular.

### Fase 1: IKEv2 Proposal & Policy

| Parámetro | Valor |
|---|---|
| Algoritmo de Cifrado | AES-CBC-256 (Advanced Encryption Standard con encadenamiento de bloques de cifrado de 256 bits) |
| Integridad y Hash | SHA-256 (Secure Hash Algorithm de 256 bits para procesos de autenticación e intercambio seguro) |
| Grupo Diffie-Hellman | Grupo 14 (Exponenciación modular de 2048 bits para la generación segura de claves efímeras) |
| Autenticación | Llaves precompartidas asimétricas/simétricas mediante contenedores locales (keyring) |
| Pre-Shared Key Unificada | `ClaveSeguraVpn2026` |

### Fase 2: IPsec Transform Set

| Parámetro | Valor |
|---|---|
| Nombre del Set | `TS_VPN` |
| Encapsulación Criptográfica | ESP con AES de 256 bits (esp-aes 256) |
| Autenticación/Hashing de Datos | ESP bajo código de autenticación de mensajes cifrados (esp-sha256-hmac) |
| Modo de Operación | Modo Túnel nativo (mode tunnel), encapsulando la cabecera IP original completa |

---

## 5. Puntos Críticos Analizados

### A. Mecanismo de Exclusión de NAT (NAT Bypass)

Al denegar este tráfico del proceso de traducción de direcciones, los paquetes de datos inter-LAN retienen sus cabeceras e IPs de origen originales intactas. Esto permite que el motor criptográfico intercepte y asocie de manera efectiva el tráfico dentro del Crypto Map.

### B. Inyección por Crypto Map en VPN Basada en Políticas

Al implementar una VPN estrictamente basada en políticas (Crypto Maps), el enrutamiento no se realiza hacia una interfaz Tunnel lógica virtualizada. El router evalúa el tráfico saliente a través de su interfaz física WAN (Gi0/0). Si las direcciones IP del paquete coinciden con los criterios declarados de tráfico interesante en la lista `ACL_VPN_TRAFFIC`, el dispositivo suspende el reenvío regular e inicia la negociación dinámica del túnel cifrado IPsec de forma transparente.

---

## 6. Protocolo de Verificación y Diagnóstico Técnico

### Paso 1: Generación de Tráfico Interesante para Levantamiento de Túnel

Por definición, los mecanismos IPsec levantan los túneles de comunicación bajo demanda en presencia del primer paquete de datos legítimo. Para activar la infraestructura, se emite una solicitud de eco ICMP desde un host terminal interno, forzando el direccionamiento de la LAN local.

### Paso 2: Validación del Canal de Control en Fase 1 (IKEv2 SA)

Para examinar el estado del intercambio de llaves y la correcta concordancia criptográfica de los peers remotos, se emplea el comando de diagnóstico avanzado para IKEv2:

```
show crypto ikev2 sa
```

**Criterio de Aceptación:** el resultado en consola debe declarar de forma explícita el parámetro **READY** en la columna de estado. Esto certifica que la autenticación mutua mediante las claves precompartidas y los perfiles asignados ha culminado con éxito.

### Paso 3: Validación del Canal de Datos en Fase 2 (IPsec SA)

Una vez establecido el enlace de control, es indispensable auditar que los flujos de datos de usuario sean procesados por los algoritmos de encriptación simétrica ESP:

```
show crypto ipsec sa
```

Los registros y contadores del sistema para `#pkts encaps` (paquetes cifrados salientes) y `#pkts decaps` (paquetes descifrados entrantes) deben mostrar valores enteros mayores a cero e incrementar en tiempo real conforme persista el envío de ráfagas de datos.
