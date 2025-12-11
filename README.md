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
