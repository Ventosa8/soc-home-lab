\# Máquina atacante: Kali Linux



\## Sistema base



\- \*\*Imagen:\*\* Kali Linux 2026.2, imagen preconstruida oficial de VirtualBox

\- \*\*RAM asignada:\*\* 2 GB

\- \*\*IP:\*\* 192.168.56.30 (red host-only, IP estática)

\- \*\*Credenciales por defecto:\*\* kali/kali (imagen oficial)



\## Justificación de la elección



Se optó por partir de la imagen oficial preconstruida en lugar de reutilizar un

disco de un proyecto anterior, para mantener el entorno y la documentación

del lab limpios y trazables desde cero.



\## Configuración de red



Igual que el resto de máquinas del lab, Kali cuenta con dos adaptadores:

\- \*\*Adaptador 1 (NAT):\*\* acceso a internet para actualizar herramientas.

\- \*\*Adaptador 2 (Host-Only):\*\* comunicación con el resto del lab (`192.168.56.0/24`).



La IP estática se configuró vía `nmcli`:



\\`\\`\\`bash

sudo nmcli connection modify "Wired connection 2" ipv4.addresses 192.168.56.30/24

sudo nmcli connection modify "Wired connection 2" ipv4.method manual

sudo nmcli connection up "Wired connection 2"

\\`\\`\\`



\## Verificación de conectividad



Se confirmó la comunicación con el manager Wazuh mediante ping, con 0% de

pérdida de paquetes:



\## Nota sobre falsos positivos de antivirus



Durante la descarga, Windows Defender marcó el archivo `.7z` de Kali como

`Trojan:Script/Ulthar.A!ml`. Es una detección heurística basada en machine

learning (`!ml`), un patrón conocido de falso positivo asociado a

distribuciones de pentesting como Kali, cuyas herramientas incluidas

(frameworks de explotación, generadores de payloads) se asemejan a

comportamiento malicioso desde el punto de vista de un antivirus genérico.

Se verificó que la fuente de descarga era un mirror oficial de kali.org antes

de continuar.
