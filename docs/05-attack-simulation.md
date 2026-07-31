\# Fase 5: Simulación de ataques y verificación de detección



\## Objetivo



Simular técnicas de ataque reales desde la VM de Kali contra el endpoint 

Windows, y verificar la capacidad de detección del SIEM (Wazuh), 

documentando tanto los éxitos como las limitaciones observadas en el 

stack de telemetría actual.



\## Preparación previa



Antes del primer ataque se revisó el `ossec.conf` del agente Windows, 

confirmando que el canal `Security` ya estaba configurado, pero \*\*Sysmon 

no estaba siendo recolectado\*\* — se añadió el bloque `<localfile>` 

correspondiente y se reinició el servicio del agente.



​```xml

<localfile>

&#x20; <location>Microsoft-Windows-Sysmon/Operational</location>

&#x20; <log\_format>eventchannel</log\_format>

</localfile>

​```



Tras el cambio, se confirmó en Wazuh (Threat Hunting) la llegada de 

eventos de Sysmon.



\## Ataque 1: Reconocimiento de red (Nmap)



\*\*MITRE ATT\&CK:\*\* T1046 - Network Service Scanning



​```bash

nmap -sV -sC 192.168.56.20

​```



!\[Nmap scan filtrado](../images/nmap-scan-filtered.png)



\*\*Resultado:\*\* Sin detección en Wazuh.



\*\*Análisis:\*\* El Firewall de Windows filtra todo el tráfico entrante 

(`1000 filtered tcp ports`), por lo que Nmap no logra completar ningún 

handshake TCP. Sysmon (Event ID 3, Network Connection) solo audita 

conexiones \*\*salientes\*\* iniciadas por procesos del propio sistema, no 

intentos de conexión entrantes descartados por el firewall antes de 

llegar a ninguna aplicación — de ahí que no se genere ningún evento.



Esto constituye una \*\*brecha de detección\*\* identificada: para cubrir 

este vector sería necesario habilitar el logging del propio Firewall de 

Windows (Windows Filtering Platform, Event ID 5157) o desplegar un IDS 

de red (ej. Suricata) monitorizando el segmento.



\## Ataque 2: Fuerza bruta contra SMB



\*\*MITRE ATT\&CK:\*\* T1110 - Brute Force



\### Herramienta



Se intentó primero con Hydra, que falló por incompatibilidad con SMB2/3 

(protocolo usado por defecto en Windows 10):



​```bash

hydra -l javier -P wordlist.txt smb://192.168.56.20

\# \[ERROR] invalid reply from target smb://192.168.56.20:445/

​```



!\[Error de Hydra contra SMB2/3](../images/hydra-smb-error.png)



Se sustituyó por \*\*NetExec (nxc)\*\*, herramienta estándar actual para 

pentesting de SMB:



​```bash

nxc smb 192.168.56.20 -u javier -p wordlist.txt

​```



!\[Ataque de fuerza bruta exitoso con nxc](../images/nxc-smb-bruteforce-success.png)



\### Detección en Wazuh



| Rule ID | Descripción | Nivel |

|---|---|---|

| 60122 | Logon Failure - Unknown user or bad password | 5 |

| 60204 | Multiple Windows Logon Failures (correlación) | 10 |

| 92652 | Successful Remote Logon | 6 |



!\[Correlación de alertas de fuerza bruta en Wazuh](../images/wazuh-bruteforce-correlation.png)



El detalle del evento de logon exitoso confirma el origen del ataque 

(`ipAddress: 192.168.56.30`, la IP de la VM de Kali) y el paquete de 

autenticación usado (`NTLM V2`):



!\[Detalle del evento de logon exitoso](../images/wazuh-logon-success-detail.png)



\### Efecto colateral observado



Tras superar el umbral de intentos fallidos, Windows bloqueó 

automáticamente la cuenta (`STATUS\_ACCOUNT\_LOCKED\_OUT`) — una 

contramedida nativa del sistema operativo, registrable vía Event ID 4740.



\### Análisis



La detección funcionó en tres niveles distintos: el evento individual de 

fallo, la correlación automática de patrón de ataque (nivel 10, mucho 

más visible que un evento aislado de nivel 5), y el logon exitoso final. 

La secuencia completa — fallos repetidos → alerta de correlación → 

bloqueo de cuenta → éxito tras desbloqueo — reproduce la firma típica de 

un ataque de fuerza bruta exitoso tal como lo vería un analista SOC en 

un caso real.



\## Próximos pasos



\- Ataque 3: acceso a credenciales vía LSASS (T1003)

\- Ataque 4: movimiento lateral / ejecución remota (T1105 / T1021)

