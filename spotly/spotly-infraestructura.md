---
description: El diagrama muestra la infraestructura física de red de Spotly.
---

# SPOTLY -  Infraestructura

<figure><img src="../.gitbook/assets/Screenshot 2026-05-14 at 2.48.21 pm.png" alt=""><figcaption></figcaption></figure>

El tráfico de internet entra por OPNsense a través de la interfaz `em0`, conectada directamente a la red de la escuela. OPNsense actúa como router y firewall central — todo el tráfico que necesita cruzar entre redes pasa obligatoriamente por él. Desde la interfaz `ue0`, OPNsense se conecta al switch HP 1810-24G, que distribuye el tráfico hacia el resto de equipos.

Los tres nodos del cluster (node1, node2, node3) reciben las VLANs 10 y 20 en modo tagged. La VLAN 10 gestiona el tráfico intra-cluster de MicroCloud y la VLAN 20 se utiliza para administración y acceso SSH. El servidor de backup está aislado en la VLAN 40, con acceso al resto de redes únicamente durante la ventana nocturna controlada mediante Firewall Schedules en OPNsense. El PC de gestión pertenece a la VLAN 20 y es el punto de entrada para la administración del sistema, donde se ejecutan Ansible y el agente Wazuh.
