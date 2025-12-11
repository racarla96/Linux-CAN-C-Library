# Linux CAN C++ Library

**Linux CAN C++ Library** es una librería ligera y moderna en C++ para comunicarse con buses CAN en Linux usando **SocketCAN**.  
Proporciona una API sencilla para enviar y recibir tramas CAN de forma eficiente, usando sockets RAW y `epoll` para E/S no bloqueante.

Autor: **Rafael Carbonell Lázaro (racarla96)**

---

## Características

- Basada en la interfaz nativa **SocketCAN** del kernel Linux.
- Clase C++ simple (`SocketCANInterface`) para:
  - Inicializar y cerrar una interfaz CAN (`can0`, `can1`, `can_steer_drv`, etc.).
  - Enviar tramas CAN (`write`).
  - Leer múltiples tramas con tiempo de espera configurable (`read`).
- Uso de **epoll** para lectura eficiente y no bloqueante.
- Manejo básico de errores mediante mensajes claros a `std::cerr`.
- Código fácil de integrar en proyectos de:
  - Robótica
  - Automoción
  - Automatización industrial
  - ROS/ROS2 y otros frameworks

---

## Requisitos

- Sistema operativo **Linux** con soporte **SocketCAN**.
- Kernel Linux con módulos CAN activados (`can`, `can_raw`, y el driver de tu dispositivo CAN).
- Compilador C++ compatible (`g++` >= 7 recomendado).
- **CMake** (si usas el sistema de build sugerido).

Para pruebas sin hardware real, puedes usar una interfaz virtual `vcan`.

---

## Estructura típica del proyecto

Ejemplo de estructura:

```text
.
├── CMakeLists.txt
├── include/
│   └── socket_can_interface.hpp
├── src/
│   └── socket_can_interface.cpp
├── examples/
│   ├── can_read_example.cpp
│   └── can_write_example.cpp
└── LICENSE
```

*(Ajusta los nombres de archivos y directorios según tu repositorio real.)*

---

## Instalación (con CMake)

Desde la raíz del proyecto:

```bash
mkdir build && cd build
cmake ..
make
sudo make install
```

Esto suele instalar:

- La librería (por ejemplo `liblinux_can_cpp.so`) en `/usr/local/lib`.
- Los headers en `/usr/local/include`.

> Asegúrate de que tu `CMakeLists.txt` instale correctamente la librería y los headers con los nombres que prefieras.

---

## Configuración de la interfaz CAN en Linux

La librería utiliza SocketCAN. Necesitas que la interfaz CAN esté configurada y levantada.

### 1. Interfaz CAN real (USB‑CAN, PCI‑CAN, etc.)

Suponiendo que tu adaptador se ve como `can0`:

```bash
# Configurar bitrate a 1 Mbps
sudo ip link set can0 type can bitrate 1000000

# Levantar la interfaz
sudo ip link set up can0

# Comprobar el estado
ip link show can0
```

### 2. Interfaz virtual para pruebas (`vcan`)

Si no tienes hardware, puedes crear una interfaz virtual:

```bash
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0

ip link show vcan0
```

Después podrás usar `vcan0` con la librería como si fuera una interfaz CAN real.

---

## Nombre fijo de interfaz y configuración automática con udev (USB‑CAN)

Si usas un adaptador USB‑CAN, a veces el nombre de la interfaz (`can0`, `can1`, etc.) puede cambiar dependiendo del orden de conexión.  
Para tener siempre el mismo nombre (por ejemplo `can_steer_drv`) y configurar automáticamente bitrate y estado, puedes usar una regla **udev**.

### Script de ejemplo para crear la regla udev

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

# Solicitar confirmación
read -p "¿Deseas continuar? (s/n): " CONFIRM
if [[ "$CONFIRM" != "s" && "$CONFIRM" != "S" ]]; then
    echo "Cancelado."
    exit 1
fi

# Escribir la regla
echo "$RULE1" | sudo tee /etc/udev/rules.d/81-can-usb.rules

# Recargar las reglas de udev
sudo udevadm control --reload-rules
echo -e "\nRegla añadida y udev recargado."
echo "Para aplicar los cambios:"
echo "1. Desconecta el adaptador USB-CAN"
echo "2. Vuelve a conectarlo"
echo "3. Verifica el nombre con: ip link show $INTERFACE_NAME"
```

Una vez creada la regla:

- Cada vez que conectes el adaptador USB‑CAN:
  - Se creará la interfaz con el nombre fijo (por ejemplo `can_steer_drv`).
  - Se configurará automáticamente el bitrate.
  - La interfaz se levantará sola.
- Podrás usar ese nombre fijo en la librería:

```cpp
SocketCANInterface can("can_steer_drv");
can.init();
```

---

## Uso de la librería

### API principal

La clase central es `SocketCANInterface`:

```cpp
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
```

### Ejemplo: leer tramas CAN

```cpp
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
```

### Ejemplo: enviar una trama CAN

```cpp
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
```

---

## Compilar un programa que use la librería

### Con `g++` directamente

Ejemplo (ajusta el nombre de la librería según tu configuración real):

```bash
g++ -std=c++14 -o can_reader \
    examples/can_read_example.cpp \
    -I/usr/local/include \
    -L/usr/local/lib -llinux_can_cpp
```

- `-I/usr/local/include` debe apuntar al directorio donde se instaló `socket_can_interface.hpp`.
- `-llinux_can_cpp` debe ser el nombre real del `.so` (por ejemplo `-lcanxx`, `-llinux_can_cpp`, etc.).

### Con CMake (recomendado)

Si exportas un paquete CMake, tu `CMakeLists.txt` para una aplicación podría ser algo así:

```cmake
cmake_minimum_required(VERSION 3.10)
project(can_app LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(LinuxCANCpp REQUIRED) # Nombre del paquete que definas

add_executable(can_reader main.cpp)
target_link_libraries(can_reader PRIVATE LinuxCANCpp::LinuxCANCpp)
```

> Cambia `LinuxCANCpp` por el nombre del target/paquete que definas en tu propia configuración CMake.

---

## Licencia

**Linux CAN C++ Library**  
Copyright (c) 2025,  
**Rafael Carbonell Lázaro (racarla96)**

Este proyecto se distribuye bajo la licencia **Creative Commons Attribution 4.0 International (CC BY 4.0)**.

En resumen:

- Puedes usar, modificar y redistribuir la librería, incluso en proyectos comerciales o privados.
- Es obligatorio mantener **atribución** al autor/proyecto  
  (por ejemplo, en la documentación, créditos, “About” de la aplicación, etc.).
- No puedes imponer restricciones adicionales que impidan a otros ejercer los permisos que otorga la licencia.

Texto legal completo:  
[https://creativecommons.org/licenses/by/4.0/legalcode](https://creativecommons.org/licenses/by/4.0/legalcode)

---
