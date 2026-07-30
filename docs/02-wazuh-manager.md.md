\## Instalación



Wazuh se instaló usando el script de instalación "todo en uno" oficial, que despliega

el manager, el indexer y el dashboard en un único nodo:



\\`\\`\\`bash

curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh

sudo bash ./wazuh-install.sh -a

\\`\\`\\`



\## Resultado



Instalación completada sin errores. Dashboard accesible en `https://192.168.56.10`.



En las primeras 24h el propio sistema generó alertas de severidad media y baja,

correspondientes a la actividad normal del manager recién desplegado.
