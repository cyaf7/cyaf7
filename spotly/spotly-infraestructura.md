---
description: El diagrama muestra la infraestructura física de red de Spotly.
---

# SPOTLY -  Infraestructura

<figure><img src="../.gitbook/assets/Screenshot 2026-05-14 at 4.23.17 pm.png" alt=""><figcaption></figcaption></figure>

El tráfico de internet entra por OPNsense a través de la interfaz `em0`, la NIC integrada del equipo, conectada directamente a la red de la escuela mediante DHCP. OPNsense actúa como router y firewall central — todo el tráfico que necesita cruzar entre redes pasa obligatoriamente por él, donde se aplican las reglas de seguridad antes de permitir o denegar el paso. Desde la interfaz `ue0`, un adaptador USB Ethernet Realtek añadido al equipo para disponer de una segunda interfaz de red, OPNsense se conecta al switch HP 1810-24G transportando todas las VLANs etiquetadas en un único enlace físico. El switch distribuye el tráfico hacia el resto de equipos según la configuración de participación VLAN de cada puerto.

Los tres nodos del cluster (node1, node2, node3) reciben las VLANs 10, 20 y 50 en modo tagged. La VLAN 10 gestiona el tráfico intra-cluster de MicroCloud — descubrimiento mDNS entre nodos, heartbeat de LXD y comunicación interna de MicroCeph. La VLAN 20 es la red de administración, utilizada para acceso SSH, ejecución de playbooks de Ansible y autenticación contra el servidor OpenLDAP y Wazuh instalados en el Admin PC. La VLAN 50 no tiene dirección IP asignada y actúa exclusivamente como uplink físico de MicroOVN — es el canal por el que las redes virtuales internas del cluster (ovn-app, ovn-dmz, ovn-infra, ovn-backup) se conectan con el exterior a través de OPNsense para salir a internet.

El servidor de backup está aislado en la VLAN 40, con acceso al resto de redes únicamente durante la ventana nocturna controlada mediante Firewall Schedules en OPNsense. Fuera de esa ventana, el firewall bloquea completamente cualquier tráfico originado en esa red. El PC de gestión pertenece a la VLAN 20 y es el punto de entrada para la administración del sistema, desde donde se ejecutan los playbooks de Ansible contra los nodos físicos y las VMs del cluster, y donde corre el agente Wazuh para la monitorización de seguridad del entorno.
