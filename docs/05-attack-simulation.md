\# Fase 5: Simulación de ataques y verificación de detección



\## Objetivo



Simular técnicas de ataque reales desde la máquina atacante (Kali) contra 

el endpoint Windows, y verificar la capacidad de detección del SIEM 

(Wazuh), documentando tanto los aciertos como las limitaciones 

observadas en el stack de telemetría actual.



\## Preparación previa: recolección de Sysmon



Antes de lanzar cualquier ataque, se revisó el `ossec.conf` del agente 

Windows. El canal `Security` ya estaba configurado, pero \*\*Sysmon no 

estaba siendo recolectado por el agente\*\* — se añadió el bloque 

correspondiente:



```xml

<localfile>

&#x20; <location>Microsoft-Windows-Sysmon/Operational</location>

&#x20; <log\_format>eventchannel</log\_format>

</localfile>

```



Tras reiniciar el servicio del agente, se confirmó en Wazuh (Threat 

Hunting) la llegada de eventos de Sysmon.



\### Justificación



Sin este paso, gran parte de la detección buscada en esta fase (creación 

de procesos, acceso a LSASS, conexiones salientes) habría sido invisible 

para el SIEM, ya que dependía por completo de Sysmon y no del log de 

Seguridad estándar de Windows.



\---



\## Ataque 1: Reconocimiento de red (Nmap)



\*\*Técnica MITRE ATT\&CK:\*\* T1046 - Network Service Scanning



\*\*Comando ejecutado:\*\*

```bash

nmap -sV -sC 192.168.56.20

```



!\[Nmap scan filtrado](../images/nmap-scan-filtered.png)



\*\*Resultado:\*\* sin detección en Wazuh. Los 1000 puertos escaneados 

aparecen como `filtered` — el Firewall de Windows descarta el tráfico 

entrante antes de que llegue a ninguna aplicación.



\### Análisis



Sysmon (Event ID 3, Network Connection) audita conexiones \*\*salientes\*\* 

iniciadas por procesos del propio sistema, no intentos de conexión 

entrantes bloqueados por el firewall. Al no completarse ningún handshake 

TCP, no hay ningún proceso que registre la conexión, y por tanto no se 

genera ningún evento.



Esto se documenta como una \*\*brecha de detección\*\* real del stack 

actual: para cubrir este vector haría falta habilitar el logging propio 

del Firewall de Windows (Windows Filtering Platform, Event ID 5157) o 

desplegar un IDS de red (ej. Suricata) monitorizando el segmento.



\---



\## Ataque 2: Fuerza bruta contra SMB



\*\*Técnica MITRE ATT\&CK:\*\* T1110 - Brute Force



\### Elección de herramienta



Se intentó primero con Hydra, que falló por incompatibilidad con SMB2/3 

(protocolo usado por defecto en Windows 10):



```bash

hydra -l javier -P wordlist.txt smb://192.168.56.20

```



!\[Error de Hydra contra SMB2/3](../images/hydra-smb-error.png)



Se sustituyó por \*\*NetExec (nxc)\*\*, herramienta actual de referencia 

para pentesting de SMB, que sí soporta SMB2/3 correctamente:



```bash

nxc smb 192.168.56.20 -u javier -p wordlist.txt

```



!\[Ataque de fuerza bruta exitoso con nxc](../images/nxc-smb-bruteforce-success.png)



\### Justificación



Este cambio de herramienta se documenta explícitamente porque refleja un 

problema real de compatibilidad entre herramientas ofensivas "clásicas" 

y versiones modernas de Windows — un matiz que demuestra comprensión del 

protocolo, no solo ejecución de comandos.



\### Detección en Wazuh



La detección funcionó en tres niveles distintos:



\- \*\*`Logon Failure - Unknown user or bad password`\*\* (rule.id `60122`, 

&#x20; nivel 5) — un evento por cada intento fallido.

\- \*\*`Multiple Windows Logon Failures`\*\* (rule.id `60204`, nivel 10) — 

&#x20; correlación automática del patrón de ataque, mucho más visible que 

&#x20; los eventos individuales.

\- \*\*`Successful Remote Logon`\*\* (rule.id `92652`, nivel 6) — el logon 

&#x20; exitoso final, tras encontrar la contraseña correcta.



!\[Correlación de alertas de fuerza bruta en Wazuh](../images/wazuh-bruteforce-correlation.png)



El detalle del evento de logon exitoso confirma el origen del ataque 

(`ipAddress: 192.168.56.30`, la IP de la máquina Kali) y el paquete de 

autenticación usado (`NTLM V2`):



!\[Detalle del evento de logon exitoso](../images/wazuh-logon-success-detail.png)



\### Efecto colateral observado



Tras superar el umbral de intentos fallidos, Windows bloqueó 

automáticamente la cuenta (`STATUS\_ACCOUNT\_LOCKED\_OUT`) — una 

contramedida nativa del sistema operativo, registrable vía Event ID 

4740\.



\### Análisis



La secuencia completa observada — fallos repetidos → alerta de 

correlación → bloqueo de cuenta → login exitoso tras desbloqueo — 

reproduce la firma típica de un ataque de fuerza bruta exitoso tal como 

la vería un analista SOC en un caso real, y demuestra que la 

correlación de eventos (nivel 10) aporta muchísima más señal que 

cualquier evento individual (nivel 5) por sí solo.



\---



\## Próximos ataques



\- Acceso a credenciales vía LSASS (T1003)

\- Movimiento lateral / ejecución remota (T1105 / T1021)

