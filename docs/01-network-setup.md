# Configuración de red del lab

## Topología

El lab utiliza una red interna aislada (VirtualBox Host-Only Network) para simular
un entorno corporativo separado de la red doméstica real:

- **Red:** 192.168.56.0/24
- **DHCP:** deshabilitado (IPs estáticas asignadas manualmente)

## Máquinas planificadas

| Máquina         | Rol                      | IP             |
|------------------|---------------------------|----------------|
| Wazuh Manager    | SIEM / recolección de logs | 192.168.56.10 |
| Windows 10       | Endpoint monitorizado      | 192.168.56.20 |
| Kali Linux       | Máquina atacante            | 192.168.56.30 |

## Configuración del adaptador host-only

Cada VM tiene dos adaptadores de red:
- **Adaptador 1 (NAT):** acceso a internet para instalar paquetes.
- **Adaptador 2 (Host-Only):** comunicación interna aislada entre las VMs del lab.