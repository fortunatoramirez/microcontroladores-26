# Práctica: Escritura de texto con joystick y pantalla OLED

La palanca del joystick permitirá seleccionar grupos de caracteres.

* Mover a la izquierda o derecha: cambiar de grupo.
* Mover hacia arriba o abajo: seleccionar un carácter del grupo.
* Presionar el joystick: escribir el carácter seleccionado.
* Borrar: para borrar el último caracter.
* Mantener presionado el joystick: entrar o salir del modo de desplazamiento (scroll).
* En modo de desplazamiento, mover arriba o abajo para recorrer el texto.

## Conexión de la pantalla OLED

| Pantalla OLED | ESP32   |
| ------------- | ------- |
| VCC           | 3.3 V   |
| GND           | GND     |
| SDA           | GPIO 21 |
| SCL           | GPIO 22 |

La dirección I2C utilizada en el programa será `0x3C`.

---

## Paso 2. Conexión del Joystick

| Joystick | ESP32   |
| -------- | ------- |
| VCC      | 3.3 V   |
| GND      | GND     |
| VRx      | GPIO 34 |
| VRy      | GPIO 35 |
| SW       | GPIO 32 |

El joystick debe alimentarse con **3.3 V**, ya que sus salidas analógicas estarán conectadas directamente al ESP32.

---

## Bibliotecas

Verificar que tengan instalads las bibliotecas de las prácticas pasadas para la pantalla desde Administrador de bibliotecas de Arduino IDE:

```text
Adafruit GFX Library
Adafruit SSD1306
```

---

## Probar

1. Mover el joystick a la izquierda y a la derecha; arriba y abajo; y presionar (botón del joystick) para escribir el caracter. Probar espacios y borrar.
2. Escribir suficiente texrto para verificar el scroll. Mantener el Joystick presionado para poner `MODO: SCROLL` y mover arriba y abajo. Presionar corto para regresar.

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

/* Pines del joystick */

const int PIN_X = 34;
const int PIN_Y = 35;
const int PIN_BOTON = 32;

/* Grupos de caracteres */

const char* grupos[] = {
  "ABCabc",
  "DEFdef",
  "GHIghi",
  "JKLjkl",
  "MNOmno",
  "PQRSpqrs",
  "TUVtuv",
  "WXYZwxyz",
  "0123456789",
  " .,!?-_@#:/",
  "<"
};

const int TOTAL_GRUPOS =
  sizeof(grupos) / sizeof(grupos[0]);

int grupoActual = 0;
int caracterActual = 0;

/* Texto */

String texto = "";

const int MAXIMO_CARACTERES = 200;
const int CARACTERES_POR_LINEA = 20;
const int LINEAS_VISIBLES = 5;

int primeraLinea = 0;

bool modoScroll = false;


unsigned long ultimoMovimiento = 0;
const unsigned long TIEMPO_MOVIMIENTO = 180;

// Límites aproximados del joystick
const int LIMITE_BAJO = 1000;
const int LIMITE_ALTO = 3000;

/* Botón */
int lecturaAnteriorBoton = HIGH;
int estadoBoton = HIGH;

unsigned long tiempoCambioBoton = 0;
unsigned long tiempoInicioBoton = 0;

const unsigned long ANTIRREBOTE = 30;
const unsigned long PRESION_LARGA = 800;


void setup()
{
  pinMode(PIN_BOTON, INPUT_PULLUP);

  Wire.begin(21, 22);

  if (!pantalla.begin(
        SSD1306_SWITCHCAPVCC,
        DIRECCION_OLED))
  {
    while (true)
    {
      // La pantalla no fue encontrada
    }
  }

  pantalla.clearDisplay();
  pantalla.setTextColor(SSD1306_WHITE);
  pantalla.setTextSize(1);

  actualizarPantalla();
}


void loop()
{
  leerBoton();
  leerJoystick();
}


void leerJoystick()
{
  if (millis() - ultimoMovimiento <
      TIEMPO_MOVIMIENTO)
  {
    return;
  }

  int valorX = analogRead(PIN_X);
  int valorY = analogRead(PIN_Y);

  if (modoScroll)
  {
    if (valorY < LIMITE_BAJO)
    {
      moverScroll(-1);
      ultimoMovimiento = millis();
    }
    else if (valorY > LIMITE_ALTO)
    {
      moverScroll(1);
      ultimoMovimiento = millis();
    }

    return;
  }

  // Cambiar de grupo
  if (valorX < LIMITE_BAJO)
  {
    cambiarGrupo(-1);
    ultimoMovimiento = millis();
  }
  else if (valorX > LIMITE_ALTO)
  {
    cambiarGrupo(1);
    ultimoMovimiento = millis();
  }

  // Cambiar de carácter
  else if (valorY < LIMITE_BAJO)
  {
    cambiarCaracter(-1);
    ultimoMovimiento = millis();
  }
  else if (valorY > LIMITE_ALTO)
  {
    cambiarCaracter(1);
    ultimoMovimiento = millis();
  }
}


void cambiarGrupo(int cambio)
{
  grupoActual += cambio;

  if (grupoActual < 0)
  {
    grupoActual = TOTAL_GRUPOS - 1;
  }

  if (grupoActual >= TOTAL_GRUPOS)
  {
    grupoActual = 0;
  }

  caracterActual = 0;

  actualizarPantalla();
}

void cambiarCaracter(int cambio)
{
  int cantidadCaracteres =
    strlen(grupos[grupoActual]);

  caracterActual += cambio;

  if (caracterActual < 0)
  {
    caracterActual = cantidadCaracteres - 1;
  }

  if (caracterActual >= cantidadCaracteres)
  {
    caracterActual = 0;
  }

  actualizarPantalla();
}


void leerBoton()
{
  int lecturaActual = digitalRead(PIN_BOTON);

  if (lecturaActual != lecturaAnteriorBoton)
  {
    tiempoCambioBoton = millis();
    lecturaAnteriorBoton = lecturaActual;
  }

  if (millis() - tiempoCambioBoton >
      ANTIRREBOTE)
  {
    if (lecturaActual != estadoBoton)
    {
      estadoBoton = lecturaActual;

      if (estadoBoton == LOW)
      {
        tiempoInicioBoton = millis();
      }
      else
      {
        unsigned long duracion =
          millis() - tiempoInicioBoton;

        if (duracion >= PRESION_LARGA)
        {
          cambiarModo();
        }
        else
        {
          presionCorta();
        }
      }
    }
  }
}


void presionCorta()
{
  if (modoScroll)
  {
    modoScroll = false;
    actualizarPantalla();
    return;
  }

  escribirCaracter();
}


void cambiarModo()
{
  modoScroll = !modoScroll;
  actualizarPantalla();
}


void escribirCaracter()
{
  char caracter =
    grupos[grupoActual][caracterActual];

  // El símbolo < funciona como borrar

  if (caracter == '<')
  {
    if (texto.length() > 0)
    {
      texto.remove(texto.length() - 1);
    }
  }
  else
  {
    if (texto.length() < MAXIMO_CARACTERES)
    {
      texto += caracter;
    }
  }

  colocarUltimasLineas();
  actualizarPantalla();
}


int obtenerTotalLineas()
{
  if (texto.length() == 0)
  {
    return 1;
  }

  return (
    texto.length() +
    CARACTERES_POR_LINEA - 1
  ) / CARACTERES_POR_LINEA;
}


void colocarUltimasLineas()
{
  int totalLineas = obtenerTotalLineas();

  primeraLinea =
    totalLineas - LINEAS_VISIBLES;

  if (primeraLinea < 0)
  {
    primeraLinea = 0;
  }
}


void moverScroll(int movimiento)
{
  int totalLineas = obtenerTotalLineas();

  int ultimaPosicion =
    totalLineas - LINEAS_VISIBLES;

  if (ultimaPosicion < 0)
  {
    ultimaPosicion = 0;
  }

  primeraLinea += movimiento;

  if (primeraLinea < 0)
  {
    primeraLinea = 0;
  }

  if (primeraLinea > ultimaPosicion)
  {
    primeraLinea = ultimaPosicion;
  }

  actualizarPantalla();
}

void actualizarPantalla()
{
  pantalla.clearDisplay();

  pantalla.setCursor(0, 0);

  if (modoScroll)
  {
    pantalla.print("MODO: SCROLL");
  }
  else
  {
    pantalla.print("MODO: ESCRIBIR");
  }

  pantalla.setCursor(0, 8);

  if (modoScroll)
  {
    pantalla.print("Linea: ");
    pantalla.print(primeraLinea + 1);
    pantalla.print("/");
    pantalla.print(obtenerTotalLineas());
  }
  else
  {
    mostrarSeleccion();
  }

  pantalla.drawLine(
    0,
    17,
    ANCHO_PANTALLA - 1,
    17,
    SSD1306_WHITE
  );

  mostrarTexto();

  pantalla.display();
}


void mostrarSeleccion()
{
  char caracter =
    grupos[grupoActual][caracterActual];

  pantalla.print("G");
  pantalla.print(grupoActual + 1);
  pantalla.print(": ");

  if (caracter == ' ')
  {
    pantalla.print("ESPACIO");
  }
  else if (caracter == '<')
  {
    pantalla.print("BORRAR");
  }
  else
  {
    pantalla.print(caracter);
  }
}


void mostrarTexto()
{
  if (texto.length() == 0)
  {
    pantalla.setCursor(0, 24);
    pantalla.print("(sin texto)");
    return;
  }

  for (int linea = 0;
       linea < LINEAS_VISIBLES;
       linea++)
  {
    int lineaTexto = primeraLinea + linea;

    int inicio =
      lineaTexto * CARACTERES_POR_LINEA;

    if (inicio >= texto.length())
    {
      break;
    }

    String parte =
      texto.substring(
        inicio,
        inicio + CARACTERES_POR_LINEA
      );

    pantalla.setCursor(0, 24 + linea * 8);
    pantalla.print(parte);
  }
}
```

## Actividades 

* Investigar el uso de size().
* Realizar un modo de EDITAR que permita navegar en el texto como un cursor y editar en diferentes partes del texto.
* Realizar una Biblioteca para escritura a apartir de este programa.

