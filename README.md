# Linux CAN C++ Library

Librería C++ para comunicación CAN en Linux basada en **SocketCAN**.  
Proporciona una API sencilla para enviar y recibir tramas CAN de forma eficiente, usando sockets RAW y `epoll` para I/O no bloqueante.

## Características

- Basada en la interfaz nativa **SocketCAN** del kernel Linux.
- Clase C++ simple (`SocketCANInterface`) para:
  - Inicializar y cerrar una interfaz CAN (`can0`, `can1`, …).
  - Enviar tramas CAN (`write`).
  - Leer múltiples tramas con tiempo de espera configurable (`read`).
- Uso de **epoll** para una lectura eficiente y no bloqueante.
- Manejo básico de errores con mensajes claros por `std::cerr`.
- Código pensado para integrarse fácilmente en proyectos existentes (ROS2, robótica, automatización, etc.).

## Requisitos

- Linux con soporte **SocketCAN**.
- Kernel con módulos CAN activados (`can`, `can_raw`, drivers específicos).
- Compilador C++ compatible (por ejemplo, `g++` >= 7).
- CMake (opcional, si usas el sistema de construcción propuesto).

Para probar en local puedes usar, por ejemplo:

```bash
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
```

Y usar vcan0 como interfaz CAN virtual.

Instalación (con CMake)

Suponiendo que el proyecto tiene esta estructura:

text

Copy
.
├── CMakeLists.txt
├── include/
│   └── socket_can_interface.hpp
└── src/
    └── socket_can_interface.cpp

Construcción e instalación

bash

Copy
mkdir build && cd build
cmake ..
make
sudo make install

Esto instalará:

    La librería (por ejemplo, liblinux_can_cpp.so) en /usr/local/lib (o equivalente según tu sistema).
    Los headers en /usr/local/include.

    Ajusta nombres y CMakeLists.txt según cómo finalmente llames al target de la librería.

Uso básico
API principal

La clase central es SocketCANInterface:

cpp

Copy
class SocketCANInterface {
public:
  SocketCANInterface(const std::string& interface_name);
  ~SocketCANInterface();

  bool init();   // Inicializa la interfaz (abre socket, configura epoll)
  void close();  // Cierra socket y epoll

  // Lee mensajes CAN con timeout en ms.
  // Devuelve número de frames leídos, 0 en timeout, -1 en error.
  int read(std::vector<struct can_frame>& frames, int timeout_ms = 100);

  // Envía un frame CAN. Devuelve true en éxito, false en error.
  bool write(const struct can_frame& frame);

  bool isInitialized() const;
};

Ejemplo: recibir mensajes

cpp

Copy
#include <iostream>
#include <iomanip>
#include <vector>
#include <linux/can.h>
#include "socket_can_interface.hpp"

int main(int argc, char** argv) {
    std::string iface = "can0";
    if (argc > 1) {
        iface = argv[1];
    }

    SocketCANInterface can(iface);

    if (!can.init()) {
        std::cerr << "No se pudo inicializar la interfaz CAN: " << iface << std::endl;
        return 1;
    }

    std::cout << "Leyendo desde interfaz: " << iface << std::endl;

    std::vector<struct can_frame> frames;

    while (true) {
        int n = can.read(frames, 1000); // timeout 1000 ms

        if (n < 0) {
            std::cerr << "Error al leer tramas CAN" << std::endl;
            break;
        } else if (n == 0) {
            // Timeout, no hay datos
            continue;
        }

        for (const auto& frame : frames) {
            std::cout << "ID: 0x"
                      << std::hex << std::setw(3) << std::setfill('0')
                      << frame.can_id << "  Data: ";

            for (int i = 0; i < frame.can_dlc; ++i) {
                std::cout << std::hex << std::setw(2) << std::setfill('0')
                          << static_cast<int>(frame.data[i]) << ' ';
            }

            std::cout << std::dec << '\n';
        }
    }

    can.close();
    return 0;
}

Ejemplo: enviar un mensaje

cpp

Copy
#include <iostream>
#include <cstring>
#include <linux/can.h>
#include "socket_can_interface.hpp"

int main(int argc, char** argv) {
    std::string iface = "can0";
    if (argc > 1) {
        iface = argv[1];
    }

    SocketCANInterface can(iface);

    if (!can.init()) {
        std::cerr << "No se pudo inicializar la interfaz CAN: " << iface << std::endl;
        return 1;
    }

    struct can_frame frame;
    std::memset(&frame, 0, sizeof(frame));

    frame.can_id  = 0x123;  // ID estándar
    frame.can_dlc = 8;      // Longitud de datos
    frame.data[0] = 0xDE;
    frame.data[1] = 0xAD;
    frame.data[2] = 0xBE;
    frame.data[3] = 0xEF;

    if (!can.write(frame)) {
        std::cerr << "Error al enviar trama CAN" << std::endl;
        return 1;
    }

    std::cout << "Trama CAN enviada correctamente en " << iface << std::endl;

    can.close();
    return 0;
}

Compilación de un programa de ejemplo
Con g++ directamente

bash

Copy
g++ -std=c++14 -o can_reader \
    examples/can_reader.cpp \
    -I/usr/local/include \
    -L/usr/local/lib -llinux_can_cpp

Asegúrate de usar el nombre real de la librería (por ejemplo -lcanxx, -llinux_can_cpp, etc., según cómo la hayas configurado en tu CMakeLists.txt).
Con CMake (recomendado)

Si has instalado la librería con un paquete CMake (find_package), tu CMakeLists.txt para una app podría ser:

cmake

Copy
cmake_minimum_required(VERSION 3.10)
project(can_app LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(LinuxCanCpp REQUIRED) # o el nombre que definas

add_executable(can_reader main.cpp)
target_link_libraries(can_reader PRIVATE LinuxCanCpp::LinuxCanCpp)

    Ajusta LinuxCanCpp al nombre real de tu paquete CMake (por ejemplo LinuxCanCpp, LinuxCan, canxx, etc.).

Buenas prácticas y mejoras posibles

    Añadir filtros CAN (struct can_filter) configurables vía métodos públicos.
    Exponer estadísticas (nº de frames enviados/recibidos, nº de errores).
    Sustituir std::cerr por un sistema de logging configurable.
    Añadir soporte opcional para CAN FD.
    Añadir namespaces (por ejemplo, namespace lincan { ... }) para evitar colisiones de nombres.

Licencia

Indica aquí la licencia de tu elección, por ejemplo:

    MIT
    BSD-3-Clause
    Apache-2.0

text

Copy
Copyright (c) 2025 <Tu Nombre>

Permission is hereby granted, free of charge, to any person obtaining a copy
...

Si me dices cómo vas a llamar exactamente al target de la librería en CMake (por ejemplo, linux_can_cpp, canxx, etc.), puedo ajustarte el README y los comandos de compilación para que encajen 100% con tu proyecto real.

Te propongo tres cosas:

1. Texto para añadir al README explicando cómo activar/configurar una interfaz CAN en Linux (con ejemplo de script).
2. Una pequeña mejora a tu script/explicación actual.
3. Sugerencia de licencia con atribución obligatoria incluso en uso privado: **Creative Commons Attribution 4.0 (CC BY 4.0)** o, si quieres una licencia más de “software”: **Elastic License 2.0 (ELv2)**. Te doy ejemplo con CC BY 4.0 que es más simple de entender.

---

### 1. Sección sugerida para el README: cómo activar una interfaz CAN en Linux

Podrías añadir algo así al README de *Linux CAN C++ Library*:

```markdown
## Configuración de una interfaz CAN en Linux

La librería utiliza SocketCAN, por lo que necesita que la interfaz CAN (por ejemplo `can0`) esté correctamente configurada en el sistema operativo.

### Ejemplo simple con interfaz real (USB-CAN, PCI-CAN, etc.)

Suponiendo que tu adaptador se detecta como `can0`:

```bash
# Ajusta el bitrate a 1 Mbps y levanta la interfaz
sudo ip link set can0 type can bitrate 1000000
sudo ip link set up can0

# Comprobar el estado
ip link show can0
```

### Interfaz virtual para pruebas (vcan)

Si no tienes hardware, puedes usar una interfaz virtual:

```bash
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0

ip link show vcan0
```

### Asignar nombre fijo y configuración automática con udev (ejemplo USB‑CAN)

Para evitar que el nombre de la interfaz cambie (por ejemplo `can0`, `can1` según el orden de conexión), puedes crear una regla `udev` que:

- Asigne un nombre fijo (por ejemplo `can_steer_drv`).
- Configure automáticamente bitrate y levante la interfaz al conectar el USB-CAN.

Ejemplo de script para crear la regla:

```bash
#!/bin/bash

# Script para crear una regla de udev para un adaptador USB-CAN
echo "Este script creará una regla udev para asignar un nombre personalizado a tu adaptador USB-CAN."
echo "Primero, veamos los mensajes del sistema para identificar tu adaptador:"
echo "-------------------------------------------------------------------"
dmesg | grep -i "usb\\|can"
echo "-------------------------------------------------------------------"
echo
echo "↑ Busca en la salida anterior una línea que contenga 'SerialNumber:' o similar"
echo
echo "Introduce el número de serie (por ejemplo, HW033197):"
read HWSERIAL

INTERFACE_NAME="can_steer_drv"
BITRATE="1000000"

# Crear la regla udev
RULE1="SUBSYSTEM==\"net\", ACTION==\"add\", ATTRS{serial}==\"$HWSERIAL\", NAME=\"$INTERFACE_NAME\", RUN+=\"/bin/sh -c 'sleep 1 && /sbin/ip link set $INTERFACE_NAME type can bitrate $BITRATE && /sbin/ip link set up $INTERFACE_NAME'\""

echo -e "\nLa siguiente regla será añadida a /etc/udev/rules.d/81-can-usb.rules:"
echo "$RULE1"

read -p "¿Deseas continuar? (s/n): " CONFIRM
if [[ \"$CONFIRM\" != \"s\" && \"$CONFIRM\" != \"S\" ]]; then
    echo "Cancelado."
    exit 1
fi

# Escribe la regla
echo "$RULE1" | sudo tee /etc/udev/rules.d/81-can-usb.rules

# Recarga las reglas de udev
sudo udevadm control --reload-rules

echo -e "\nRegla añadida y udev recargado."
echo "Para aplicar los cambios:"
echo "1. Desconecta el adaptador USB-CAN"
echo "2. Vuelve a conectarlo"
echo "3. Verifica el nombre con: ip link show $INTERFACE_NAME"
```

> Ajusta `INTERFACE_NAME` y `BITRATE` según tu hardware y necesidades.

Una vez creada la regla, cada vez que conectes el adaptador USB-CAN:
- Se creará la interfaz con el nombre fijo (por ejemplo `can_steer_drv`).
- Se configurará automáticamente el bitrate y se levantará la interfaz.
- Podrás usar la librería con ese nombre fijo:
  ```cpp
  LinuxCANInterface can("can_steer_drv");
  can.init();
  ```
```

Puedes cambiar `LinuxCANInterface` por el nombre real de tu clase si la renombras.

---

### 2. Pequeñas mejoras al script que ya tienes

Tu script está bien; sugerencias mínimas:

- Permitir `S` además de `s` para confirmar.
- Parametrizar también el bitrate.
- Escapar bien los `\` en la documentación (en README) para que no se traguen por el Markdown.

Versión ligeramente mejorada (ya incluida en el bloque anterior):

```bash
BITRATE="1000000"
...
RULE1="SUBSYSTEM==\"net\", ACTION==\"add\", ATTRS{serial}==\"$HWSERIAL\", NAME=\"$INTERFACE_NAME\", RUN+=\"/bin/sh -c 'sleep 1 && /sbin/ip link set $INTERFACE_NAME type can bitrate $BITRATE && /sbin/ip link set up $INTERFACE_NAME'\""

...

if [[ "$CONFIRM" != "s" && "$CONFIRM" != "S" ]]; then
    echo "Cancelado."
    exit 1
fi
```

---

### 3. Licencia con uso open‑source y atribución obligatoria también en privado

Si quieres que:

- Se pueda usar en proyectos open‑source y cerrados (privados).
- Pero **SIEMPRE** con obligación de atribución (crédito a tu proyecto), incluso en uso privado.

Una opción sencilla y conocida es:

#### Opción recomendada: **Creative Commons Attribution 4.0 (CC BY 4.0)**

- Permite uso comercial y no comercial, redistribución, modificación, etc.
- Obliga SIEMPRE a dar **crédito al autor/origen**.
- Es muy clara y estándar respecto a la parte de atribución.
- Aunque CC se usa más para documentación, también se puede aplicar a código (más puristas suelen preferir licencias tipo MIT/BSD, pero no obligan a atribución en uso privado, solo en redistribución).

Un texto simple de licencia en tu `LICENSE` podría ser:

```text
Linux CAN C++ Library
Copyright (c) 2025, Tu Nombre o Tu Organización

This work is licensed under the Creative Commons Attribution 4.0 International License.
To view a copy of this license, visit

    https://creativecommons.org/licenses/by/4.0/

You are free to:
- Share: copy and redistribute the material in any medium or format.
- Adapt: remix, transform, and build upon the material for any purpose, even commercially.

Under the following terms:
- Attribution: You must give appropriate credit, provide a link to the license,
  and indicate if changes were made. You may do so in any reasonable manner,
  but not in any way that suggests the licensor endorses you or your use.

No additional restrictions:
- You may not apply legal terms or technological measures that legally restrict others
  from doing anything the license permits.
```

Y en el README, una sección de licencia como:

```markdown
## Licencia

Este proyecto se distribuye bajo la licencia **Creative Commons Attribution 4.0 International (CC BY 4.0)**.

En resumen:

- Puedes usar, modificar y redistribuir la librería, incluso en proyectos comerciales o privados.
- Es obligatorio mantener **atribución** al autor/proyecto (por ejemplo, en la documentación, about box, créditos o similar).
- No puedes imponer restricciones adicionales más allá de las de la licencia.

Texto legal completo: [https://creativecommons.org/licenses/by/4.0/](https://creativecommons.org/licenses/by/4.0/).
```

Si prefieres algo más “estilo software” pero con cláusula de atribución más fuerte, se puede redactar una licencia personalizada inspirada en MIT o BSD donde se exija atribución visible también en uso interno/privado. Si quieres eso, dime si prefieres:

- Español o inglés para el texto legal.
- “Atribución mínima” (por ejemplo, solo en documentación y about) o algo más estricto.
