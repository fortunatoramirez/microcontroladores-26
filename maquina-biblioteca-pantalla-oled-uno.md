# Práctica: Visualización de la máquina de estados en una pantalla OLED

## Pasos

1. Conectar la pantalla OLED al ESP32:

   | Pantalla OLED | ESP32   |
   | ------------- | ------- |
   | GND           | GND     |
   | VCC           | 3.3 V   |
   | SCL           | GPIO 22 |
   | SDA           | GPIO 21 |

2. Abrir el **Administrador de bibliotecas** de Arduino IDE.

3. Instalar las siguientes bibliotecas:

   * `Adafruit SSD1306`
   * `Adafruit GFX Library`

4. Crear un nuevo programa y agregar el siguiente código.

5. Compilar y cargar el programa al ESP32.

6. Realizar pulsaciones cortas para cambiar el estado de la máquina.

7. Mantener presionado el botón durante dos segundos para regresar al estado inicial.

8. Observar en la pantalla OLED el número del estado y el nombre de la función ejecutada.

## Código

```cpp
#include <MaquinaEstados.h>

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>


const uint8_t ANCHO_PANTALLA = 128;
const uint8_t ALTO_PANTALLA = 64;

Adafruit_SSD1306 pantalla(
  ANCHO_PANTALLA,
  ALTO_PANTALLA,
  &Wire,
  -1
);


uint8_t leds[] = {32, 33, 25, 26};


void mostrarEstado(uint8_t estado)
{
  pantalla.clearDisplay();

  pantalla.setTextColor(SSD1306_WHITE);

  pantalla.setTextSize(1);
  pantalla.setCursor(0, 0);
  pantalla.println("Estado maquina:");

  pantalla.setTextSize(2);
  pantalla.setCursor(0, 12);
  pantalla.println(estado);

  pantalla.setTextSize(1);
  pantalla.setCursor(0, 38);
  pantalla.println("Funcion:");

  pantalla.setCursor(0, 50);

  switch(estado)
  {
    case 0:
      pantalla.println("Caminata de uno");
      break;

    case 1:
      pantalla.println("Prender y apagar");
      break;

    case 2:
      pantalla.println("Caminata de cero");
      break;

    case 3:
      pantalla.println("Mitad alternada");
      break;

    case 4:
      pantalla.println("Aleatorio");
      break;

    case 5:
      pantalla.println("Aleatorio inverso");
      break;
  }

  pantalla.display();
}


void setup()
{
  iniciarMaquina(15, leds, 4);

  Wire.begin(21, 22);

  pantalla.begin(
    SSD1306_SWITCHCAPVCC,
    0x3C
  );

  pantalla.clearDisplay();
  pantalla.display();
}


void loop()
{
  actualizarBoton();
  actualizarMaquina();

  uint8_t estadoMaquina = optenerEstadoMaquina();

  static uint8_t estadoAnterior = 255;

  if(estadoMaquina != estadoAnterior)
  {
    mostrarEstado(estadoMaquina);
    estadoAnterior = estadoMaquina;
  }

  switch(estadoMaquina)
  {
    case 0:
      caminataDeUno();
      break;

    case 1:
      prenderApagarTodos();
      break;

    case 2:
      caminataDeCero();
      break;

    case 3:
      prenderApagarMitadAlternada();
      break;

    case 4:
      prenderAleatorios();
      break;

    case 5:
      prenderAleatoriosInversa();
      break;
  }
}
```

## Investigar

1. Bibliotecas utilizadas en el programa y la función de cada una:

   * `MaquinaEstados.h`
   * `Wire.h`
   * `Adafruit_GFX.h`
   * `Adafruit_SSD1306.h`

2. Función de la biblioteca `Wire`, incluida por defecto en Arduino IDE.

3. Protocolo de comunicación utilizado por la pantalla OLED.

4. Funcionamiento del protocolo I²C.

5. Función de las líneas:

   * SDA.
   * SCL.

6. Significado de la dirección I²C `0x3C`.

7. Motivo por el cual varios dispositivos I²C pueden compartir los mismos cables SDA y SCL.

8. Diferencia entre las direcciones I²C `0x3C` y `0x3D`.

9. Función de la instrucción:

```cpp
Wire.begin(21, 22);
```

10. Función de la instrucción:

```cpp
pantalla.begin(SSD1306_SWITCHCAPVCC, 0x3C);
```

11. Función de los siguientes métodos:

```cpp
pantalla.clearDisplay();
pantalla.setTextColor();
pantalla.setTextSize();
pantalla.setCursor();
pantalla.println();
pantalla.display();
```

12. Razón por la que es necesario ejecutar:

```cpp
pantalla.display();
```

después de escribir el contenido.

13. Resolución de la pantalla OLED utilizada: `128 × 64 píxeles`.

14. Función del controlador SSD1306.

15. Diferencia entre modificar el contenido almacenado en memoria y mostrarlo físicamente en la pantalla.

16. Razón por la que la pantalla se actualiza únicamente cuando cambia el estado de la máquina.

17. Funcionamiento de la variable:

```cpp
static uint8_t estadoAnterior = 255;
```

18. Investigar qué es un escáner I²C y cómo puede utilizarse para conocer la dirección de una pantalla OLED.
