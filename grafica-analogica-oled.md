# Práctica: Gráfica de una señal analógica en pantalla OLED

En esta práctica se leerá una señal analógica con el ESP32 y se mostrará su gráfica en la pantalla OLED.

Para reducir el ruido, cada punto de la gráfica se obtiene calculando el promedio de varias muestras.

## Conexión de la pantalla OLED

| Pantalla OLED | ESP32   |
| ------------- | ------- |
| VCC           | 3.3 V   |
| GND           | GND     |
| SDA           | GPIO 21 |
| SCL           | GPIO 22 |

La dirección I2C utilizada será `0x3C`.

---

## Conexión del potenciómetro

Para probar el programa se puede utilizar un potenciómetro de `10 kΩ`.

| Potenciómetro    | ESP32   |
| ---------------- | ------- |
| Extremo 1        | 3.3 V   |
| Terminal central | GPIO 34 |
| Extremo 2        | GND     |

El voltaje conectado a una entrada analógica del ESP32 no debe ser mayor de **3.3 V**.

---

## Bibliotecas

Verificar que se encuentren instaladas las bibliotecas utilizadas en las prácticas anteriores:

```text
Adafruit GFX Library
Adafruit SSD1306
```

---

## Procedimiento

1. Realizar las conexiones.
2. Cargar el programa en el ESP32.
3. Girar lentamente el potenciómetro.
4. Observar el valor analógico y la gráfica en la pantalla.
5. Girar el potenciómetro rápidamente para observar los cambios de la señal.
6. Tocar el terminal central del potenciómetro para observar el ruido eléctrico.

---

## Código

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

/* Configuración de la pantalla */

#define ANCHO_PANTALLA 128
#define ALTO_PANTALLA 64
#define DIRECCION_OLED 0x3C

Adafruit_SSD1306 pantalla(
  ANCHO_PANTALLA,
  ALTO_PANTALLA,
  &Wire,
  -1
);

/* Entrada analógica */

const int PIN_SENAL = 34;

/* Configuración de la gráfica */

const int INICIO_GRAFICA = 15;
const int FINAL_GRAFICA = 63;

int puntos[ANCHO_PANTALLA];

/* Promedio de muestras */

const int MUESTRAS_PROMEDIO = 8;

long sumaMuestras = 0;
int cantidadMuestras = 0;

/* Tiempo entre muestras */

unsigned long tiempoAnterior = 0;
const unsigned long TIEMPO_MUESTRA = 2;


void setup()
{
  Serial.begin(115200);

  Wire.begin(21, 22);

  analogReadResolution(12);

  if (!pantalla.begin(
        SSD1306_SWITCHCAPVCC,
        DIRECCION_OLED))
  {
    while (true)
    {
      /* La pantalla no fue encontrada */
    }
  }

  pantalla.clearDisplay();
  pantalla.setTextColor(SSD1306_WHITE);
  pantalla.setTextSize(1);

  /* Iniciar la gráfica en la parte inferior */

  for (int i = 0; i < ANCHO_PANTALLA; i++)
  {
    puntos[i] = FINAL_GRAFICA;
  }

  pantalla.display();
}


void loop()
{
  tomarMuestra();
}


/* Tomar muestras de la entrada analógica */

void tomarMuestra()
{
  if (millis() - tiempoAnterior < TIEMPO_MUESTRA)
  {
    return;
  }

  tiempoAnterior = millis();

  int lectura = analogRead(PIN_SENAL);

  sumaMuestras += lectura;
  cantidadMuestras++;

  if (cantidadMuestras >= MUESTRAS_PROMEDIO)
  {
    int promedio =
      sumaMuestras / cantidadMuestras;

    sumaMuestras = 0;
    cantidadMuestras = 0;

    agregarPunto(promedio);
    dibujarGrafica(promedio);
  }
}


/* Recorrer la gráfica y agregar un punto */

void agregarPunto(int valor)
{
  for (int i = 0; i < ANCHO_PANTALLA - 1; i++)
  {
    puntos[i] = puntos[i + 1];
  }

  puntos[ANCHO_PANTALLA - 1] =
    map(
      valor,
      0,
      4095,
      FINAL_GRAFICA,
      INICIO_GRAFICA
    );
}


/* Dibujar la gráfica */

void dibujarGrafica(int valor)
{
  pantalla.clearDisplay();

  /* Mostrar el valor analógico */

  pantalla.setCursor(0, 0);
  pantalla.print("ADC: ");
  pantalla.print(valor);

  /* Mostrar el voltaje aproximado */

  float voltaje =
    valor * 3.3 / 4095.0;

  pantalla.setCursor(72, 0);
  pantalla.print(voltaje, 2);
  pantalla.print(" V");

  /* Línea que separa el texto de la gráfica */

  pantalla.drawLine(
    0,
    11,
    ANCHO_PANTALLA - 1,
    11,
    SSD1306_WHITE
  );

  /* Línea de referencia central */

  for (int x = 0; x < ANCHO_PANTALLA; x += 4)
  {
    pantalla.drawPixel(
      x,
      39,
      SSD1306_WHITE
    );
  }

  /* Dibujar los puntos como una línea continua */

  for (int x = 1; x < ANCHO_PANTALLA; x++)
  {
    pantalla.drawLine(
      x - 1,
      puntos[x - 1],
      x,
      puntos[x],
      SSD1306_WHITE
    );
  }

  pantalla.display();
}
```

## Funcionamiento del promedio

El programa toma ocho muestras de la entrada analógica:

```cpp
const int MUESTRAS_PROMEDIO = 8;
```

Después suma las muestras y calcula su promedio:

```cpp
int promedio =
  sumaMuestras / cantidadMuestras;
```

El promedio se utiliza como un nuevo punto de la gráfica. Esto ayuda a disminuir pequeñas variaciones y ruido en la señal.

## Modificaciones

Probar el programa utilizando los siguientes valores:

```cpp
const int MUESTRAS_PROMEDIO = 1;
```

```cpp
const int MUESTRAS_PROMEDIO = 8;
```

```cpp
const int MUESTRAS_PROMEDIO = 32;
```

Comparar la rapidez y estabilidad de la gráfica en cada caso.

## Actividades

* Investigar el funcionamiento de `analogRead()`.
* Investigar para qué se utiliza `analogReadResolution()`.
* Explicar cómo se calcula el promedio de las muestras.
* Investigar el funcionamiento de `map()`.
* Cambiar el potenciómetro por otro sensor con salida analógica.
* Mostrar en la pantalla el valor mínimo y máximo registrado.
