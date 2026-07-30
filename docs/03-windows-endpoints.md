# Configuración del endpoint Windows

## Sistema base

- **SO:** Windows 10 Pro
- **RAM asignada:** 2,5 GB
- **IP:** 192.168.56.20 (red host-only, IP estática)

## Sysmon

Se instaló Sysmon con la configuración de detección de **SwiftOnSecurity**
(https://github.com/SwiftOnSecurity/sysmon-config), un conjunto de reglas de referencia
en la comunidad de blue team que prioriza el registro de eventos de alto valor
(creación de procesos, conexiones de red, cambios en el registro, carga de drivers)
frente al ruido de una configuración por defecto.

\`\`\`
Sysmon64.exe -accepteula -i sysmonconfig-export.xml
\`\`\`

## Agente Wazuh

Se instaló el agente Wazuh 4.9.0 y se configuró para reportar al manager en
`192.168.56.10`, editando manualmente la dirección del servidor en `ossec.conf`
tras un fallo inicial en la configuración vía instalador silencioso.

## Verificación

El agente aparece como **Active** en el dashboard de Wazuh, correctamente
identificado con su sistema operativo e IP:

## Problemas encontrados y solución

- El comando `net start WazuhSvc` fallaba silenciosamente. El log
  (`C:\Program Files (x86)\ossec-agent\ossec.log`) reveló la causa real:
  `Invalid server address found: '0.0.0.0'` — el parámetro `WAZUH_MANAGER` del
  instalador MSI no se aplicó correctamente. Se corrigió editando directamente
  la sección `<server><address>` en `ossec.conf`.