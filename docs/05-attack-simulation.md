# Fase 5: Simulación de ataques y verificación de detección

## Objetivo

Simular técnicas de ataque reales desde la máquina atacante (Kali Linux) contra el endpoint Windows y verificar la capacidad de detección del SIEM (Wazuh), documentando tanto las detecciones obtenidas como las limitaciones observadas en la telemetría disponible.

---

## Preparación previa: recolección de eventos de Sysmon

Antes de iniciar las pruebas de ataque, se revisó la configuración del agente de Wazuh instalado en Windows.

El canal **Security** ya estaba siendo monitorizado, pero el agente **no estaba recopilando los eventos de Sysmon**, imprescindibles para detectar actividad relacionada con creación de procesos, conexiones de red y otras acciones avanzadas.

Se añadió el siguiente bloque al archivo `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Tras reiniciar el servicio del agente, se verificó en **Threat Hunting** la correcta recepción de eventos procedentes de Sysmon.

### Justificación

Este paso era imprescindible para que Wazuh pudiera detectar parte de la actividad generada durante las pruebas.

Sin la integración de Sysmon, gran parte de la telemetría utilizada en esta fase habría permanecido invisible para el SIEM, ya que el registro de Seguridad estándar de Windows no proporciona el mismo nivel de detalle.

---

# Ataque 1: Reconocimiento de red (Nmap)

**Técnica MITRE ATT&CK:** T1046 – Network Service Scanning

### Comando ejecutado

```bash
nmap -sV -sC 192.168.56.20
```

![Nmap scan filtrado](../images/nmap-scan-filtered.png)

### Resultado

No se generó ninguna alerta en Wazuh.

El escaneo mostró los **1000 puertos en estado `filtered`**, indicando que el Firewall de Windows descartó todas las conexiones entrantes antes de que alcanzaran cualquier servicio del sistema.

### Análisis

Sysmon únicamente registra conexiones **salientes** iniciadas por procesos locales (Event ID 3).

En este caso, las conexiones procedentes de Kali fueron bloqueadas directamente por el Firewall de Windows, por lo que:

- No se completó ningún handshake TCP.
- Ningún proceso de Windows llegó a gestionar la conexión.
- Sysmon no generó ningún evento.

Esta situación representa una **limitación real de la arquitectura de monitorización implementada**.

Para detectar este tipo de actividad sería necesario complementar Wazuh con alguna de las siguientes opciones:

- Habilitar el registro del **Windows Filtering Platform** (Event ID 5157).
- Incorporar un IDS de red como **Suricata** monitorizando el segmento de laboratorio.

---

# Ataque 2: Fuerza bruta contra SMB

**Técnica MITRE ATT&CK:** T1110 – Brute Force

## Elección de la herramienta

Inicialmente se intentó realizar el ataque utilizando **Hydra**:

```bash
hydra -l javier -P wordlist.txt smb://192.168.56.20
```

![Error de Hydra contra SMB2/3](../images/hydra-smb-error.png)

La herramienta presentó problemas de compatibilidad con **SMB2/SMB3**, protocolo utilizado por defecto en Windows 10.

Por este motivo se sustituyó por **NetExec (nxc)**, herramienta actualmente más utilizada para auditorías sobre SMB moderno.

```bash
nxc smb 192.168.56.20 -u javier -p wordlist.txt
```

![Ataque de fuerza bruta exitoso con nxc](../images/nxc-smb-bruteforce-success.png)

### Justificación

El cambio de herramienta se documenta porque refleja una situación habitual durante un pentest real.

Aunque Hydra continúa siendo una herramienta muy conocida, presenta limitaciones frente a implementaciones modernas de SMB, mientras que NetExec mantiene compatibilidad completa con SMB2 y SMB3.

Esta decisión demuestra una adaptación de la metodología al entorno objetivo en lugar de limitarse a ejecutar herramientas de forma automática.

---

## Detección en Wazuh

La actividad fue detectada correctamente por Wazuh mediante varios niveles de reglas:

- **Logon Failure - Unknown user or bad password**
  - **Rule ID:** `60122`
  - **Nivel:** 5
  - Se registró un evento por cada intento fallido de autenticación.

- **Multiple Windows Logon Failures**
  - **Rule ID:** `60204`
  - **Nivel:** 10
  - Wazuh correlacionó automáticamente los múltiples intentos fallidos, generando una única alerta de mayor criticidad.

- **Successful Remote Logon**
  - **Rule ID:** `92652`
  - **Nivel:** 6
  - Detectó el inicio de sesión remoto exitoso una vez encontrada la contraseña válida.

![Correlación de alertas de fuerza bruta en Wazuh](../images/wazuh-bruteforce-correlation.png)

El detalle del evento confirmó:

- Dirección IP de origen: **192.168.56.30** (Kali Linux).
- Protocolo de autenticación: **NTLM V2**.

![Detalle del evento de logon exitoso](../images/wazuh-logon-success-detail.png)

---

## Efecto observado

Después de superar el umbral configurado de intentos fallidos, Windows bloqueó automáticamente la cuenta del usuario.

Este comportamiento quedó registrado mediante el **Event ID 4740**, correspondiente al bloqueo de cuentas por política de seguridad.

---

## Análisis

La secuencia completa registrada durante el ataque fue la siguiente:

1. Múltiples intentos fallidos de autenticación.
2. Generación de eventos individuales de fallo.
3. Correlación automática de los eventos por parte de Wazuh.
4. Bloqueo automático de la cuenta por Windows.
5. Inicio de sesión exitoso tras el desbloqueo de la cuenta.

Esta cadena de eventos reproduce el comportamiento esperado de un ataque de fuerza bruta observado desde la perspectiva de un analista SOC.

Además, pone de manifiesto el valor de las reglas de correlación, ya que una única alerta de nivel alto proporciona mucha más información operativa que decenas de eventos individuales de baja criticidad.