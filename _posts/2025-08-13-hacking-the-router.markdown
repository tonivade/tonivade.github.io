---
layout: post
title:  "Hackeando el router"
date:   2025-08-13 20:00:00
categories: python programming
---

Este mes de agosto estoy de rodríguez y como consecuencia de ello, me aburro mucho. Así que 
he tenido que buscarme algún entretenimiento.

Desde hace un tiempo tengo una Pi-hole funcionando en mi red local, para usarla de caché de DNS y
ya de paso bloquear anuncios y sitios poco recomendables. El problema es que tenía que configurar
dispositivo a dispositivo el DNS para que usaran Pi-hole como servidor. 

¿Por qué he tenido que hacer esto? Pues muy sencillo: mi proveedor de internet, por alguna razón,
tiene bloqueadas ciertas funcionalidades del router. Puedo cambiar el servidor DNS del servidor 
DHCP del router, pero una vez que se reinicia, se pierde la configuración. Es realmente molesto.

Buscando información sobre mi router (un Sagemcom Fast 5655v2AC) encontré algo realmente sorprendente.
Aunque la interfaz gráfica tiene ciertas partes bloqueadas, si abres la consola de desarrollo del
navegador puedes acceder a todo, por ejemplo:

```javascript
$.xmo.getValuesTree("Device/DHCPv4/Server/Pools/Pool"); 
```

Es absolutamente increíble. Resulta que tiene una API que permite hacer login, consultar y modificar 
configuración. Vamos, todo. Lamentablemente, a pesar de todo, cuando reinicio el router, vuelve a 
poner las DNS por defecto del proveedor.

Y ya que existe una API, ¿no habrá alguien que se haya currado un cliente? Pues sí, efectivamente,
[existe](https://github.com/iMicknl/python-sagemcom-api). Se trata de una librería en Python que permite
loguearte en el router y hacer todo lo que necesites.

Asi que me he hecho un pequeño script en Python para configurar automáticamente mi servidor DNS en el
router, he aquí el código:

```python
# Copyright (c) 2025, Antonio Gabriel Muñoz Conejo <me AT tonivade DOT es>
# Distributed under the terms of the MIT License

import asyncio
import os
from sagemcom_api.client import SagemcomClient
from sagemcom_api.enums import EncryptionMethod
from sagemcom_api.exceptions import NonWritableParameterException

HOST = "192.168.1.1"
USERNAME = os.environ["ROUTER_USERNAME"]
PASSWORD = os.environ["ROUTER_PASSWORD"]
ENCRYPTION_METHOD = EncryptionMethod.MD5
VALIDATE_SSL_CERT = True
DNS_SERVER = "192.168.1.131"

async def main() -> None:
    async with SagemcomClient(HOST, USERNAME, PASSWORD, ENCRYPTION_METHOD, verify_ssl=VALIDATE_SSL_CERT) as client:
        try:
            await client.login()
        except Exception as exception:  # pylint: disable=broad-except
            print(exception)
            return

        # Print device information of Sagemcom F@st router
        device_info = await client.get_device_info()
        print(f"{device_info.software_version} {device_info.model_name}")

        # Retrieve values via XPath notation, output is a dict
        custom_command_output = await client.get_value_by_xpath("Device/DHCPv4/Server/Pools/Pool[@uid='1']/StaticDNSServers")
        print(f'dns server: {custom_command_output}')


        if custom_command_output != DNS_SERVER:
            try:
                custom_command_output = await client.set_value_by_xpath("Device/DHCPv4/Server/Pools/Pool[@uid='1']/StaticDNSServers", DNS_SERVER)
            except NonWritableParameterException as exception:  # pylint: disable=broad-except
                print("Not allowed to set AdvancedMode parameter on your device.")
                return

            print(custom_command_output)
        else:
            print("already using dns server")


        custom_command_output = await client.get_value_by_xpath("Device/DHCPv4/Server/Pools/Pool[@uid='1']/StaticDns")
        print(f'static dns enabled: {custom_command_output}')

        # Set value via XPath notation and catch specific errors
        if not bool(custom_command_output):
            try:
                custom_command_output = await client.set_value_by_xpath("Device/DHCPv4/Server/Pools/Pool[@uid='1']/StaticDns", "true")
            except NonWritableParameterException as exception:  # pylint: disable=broad-except
                print("Not allowed to set AdvancedMode parameter on your device.")
                return

            print(custom_command_output)
        else:
            print("already using static dns")

asyncio.run(main())
```

Y por último, solo me queda configurar un **cron** para lanzarlo cada 10 minutos y listo. No es una solución
óptima, pero es lo mejor que he conseguido por ahora.