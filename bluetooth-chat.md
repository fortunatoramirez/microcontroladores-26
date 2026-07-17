# Práctica: Chat Bluetooth entre dos ESP32

En esta práctica se utilizarán dos ESP32 con el mismo programa.

Al encender, cada ESP32 permitirá seleccionar su funcionamiento:

* `MASTER`: inicia la conexión.
* `SLAVE`: espera la conexión.

El ESP32 configurado como `SLAVE` mostrará su dirección MAC en la pantalla. Esta dirección deberá ingresarse en el ESP32 configurado como `MASTER`.

Una vez conectados, los dos ESP32 podrán escribir y enviar mensajes utilizando el joystick y la pantalla OLED.

## Controles para ingresar la dirección MAC

* Izquierda o derecha: cambiar de posición.
* Arriba o abajo: cambiar el número hexadecimal.
* Presión corta: avanzar a la siguiente posición.
* Presión larga: intentar la conexión.

Los caracteres disponibles para la dirección MAC son:

```text
0123456789ABCDEF
```

Los dos puntos de la dirección se colocan automáticamente.

## Controles del chat

* Izquierda o derecha: cambiar de grupo.
* Arriba o abajo: cambiar de carácter.
* Presión corta: seleccionar el carácter.
* Seleccionar `<`: borrar.
* Seleccionar `>`: enviar el mensaje.
* Presión larga: entrar o salir del modo scroll.
* En modo scroll, mover arriba o abajo para recorrer los mensajes.

---

## Conexión de la pantalla OLED

| Pantalla OLED | ESP32   |
| ------------- | ------- |
| VCC           | 3.3 V   |
| GND           | GND     |
| SDA           | GPIO 21 |
| SCL           | GPIO 22 |

La dirección utilizada para la pantalla es `0x3C`.

---

## Conexión del joystick

| Joystick | ESP32   |
| -------- | ------- |
| VCC      | 3.3 V   |
| GND      | GND     |
| VRx      | GPIO 34 |
| VRy      | GPIO 35 |
| SW       | GPIO 32 |

---

## Bibliotecas

Verificar que se encuentren instaladas las bibliotecas:

```text
Adafruit GFX Library
Adafruit SSD1306
```

La biblioteca `BluetoothSerial` ya se encuentra incluida en el paquete de tarjetas del ESP32.

---

## Procedimiento

1. Realizar las conexiones en los dos ESP32.
2. Cargar el mismo programa en ambos.
3. En el primer ESP32 seleccionar `SLAVE`.
4. Anotar la dirección MAC mostrada en la pantalla.
5. En el segundo ESP32 seleccionar `MASTER`.
6. Ingresar la dirección MAC del ESP32 configurado como `SLAVE`.
7. Mantener presionado el joystick para iniciar la conexión.
8. Cuando aparezca `CONECTADO`, escribir y enviar mensajes.
9. Probar la escritura, el borrado, el envío y el scroll.

---

## Código

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "BluetoothSerial.h"

/* Verificar Bluetooth Classic */

#if !defined(CONFIG_BT_ENABLED) || \
    !defined(CONFIG_BLUEDROID_ENABLED)
#error Bluetooth no se encuentra habilitado
#endif

#if !defined(CONFIG_BT_SPP_ENABLED)
#error BluetoothSerial solamente esta disponible para ESP32 clasico
#endif

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

/* Bluetooth */

BluetoothSerial SerialBT;

/* Pines del joystick */

const int PIN_X = 34;
const int PIN_Y = 35;
const int PIN_BOTON = 32;

/* Estados del sistema */

const int SELECCIONAR_ROL = 0;
const int INGRESAR_MAC = 1;
const int ESPERAR_CONEXION = 2;
const int CHAT = 3;

int estadoSistema = SELECCIONAR_ROL;

/* Rol seleccionado */

bool esMaster = false;
int opcionRol = 0;

String macLocal = "";

/* Dirección MAC destino */

char digitosMac[13] = "000000000000";

const char caracteresHex[] = "0123456789ABCDEF";

int posicionMac = 0;
bool errorConexion = false;

/* Grupos para escribir */

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
  "<>"
};

const int TOTAL_GRUPOS =
  sizeof(grupos) / sizeof(grupos[0]);

int grupoActual = 0;
int caracterActual = 0;

/* Mensajes */

String mensajeEscrito = "";
String mensajeRecibido = "";

const int MAXIMO_MENSAJE = 100;

/* Historial del chat */

const int MAXIMO_LINEAS_CHAT = 40;
const int CARACTERES_POR_LINEA = 20;
const int LINEAS_VISIBLES = 4;

String lineasChat[MAXIMO_LINEAS_CHAT];

int totalLineasChat = 0;
int primeraLineaChat = 0;

bool modoScroll = false;

/* Movimiento del joystick */

unsigned long ultimoMovimiento = 0;

const unsigned long TIEMPO_MOVIMIENTO = 180;

const int LIMITE_BAJO = 1000;
const int LIMITE_ALTO = 3000;

/* Botón del joystick */

int lecturaAnteriorBoton = HIGH;
int estadoBoton = HIGH;

unsigned long tiempoCambioBoton = 0;
unsigned long tiempoInicioBoton = 0;

const unsigned long ANTIRREBOTE = 30;
const unsigned long PRESION_LARGA = 800;


void setup()
{
  Serial.begin(115200);

  pinMode(PIN_BOTON, INPUT_PULLUP);

  Wire.begin(21, 22);

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

  mostrarSeleccionRol();
}


void loop()
{
  leerBoton();
  leerJoystick();

  revisarConexion();
  recibirBluetooth();
}


/* Lectura del joystick */

void leerJoystick()
{
  if (millis() - ultimoMovimiento <
      TIEMPO_MOVIMIENTO)
  {
    return;
  }

  int valorX = analogRead(PIN_X);
  int valorY = analogRead(PIN_Y);

  if (estadoSistema == SELECCIONAR_ROL)
  {
    leerJoystickRol(valorX);
  }
  else if (estadoSistema == INGRESAR_MAC)
  {
    leerJoystickMac(valorX, valorY);
  }
  else if (estadoSistema == CHAT)
  {
    leerJoystickChat(valorX, valorY);
  }
}


/* Selección de MASTER o SLAVE */

void leerJoystickRol(int valorX)
{
  if (valorX < LIMITE_BAJO ||
      valorX > LIMITE_ALTO)
  {
    opcionRol = !opcionRol;

    ultimoMovimiento = millis();

    mostrarSeleccionRol();
  }
}


/* Captura de la dirección MAC */

void leerJoystickMac(int valorX, int valorY)
{
  if (valorX < LIMITE_BAJO)
  {
    moverPosicionMac(-1);
    ultimoMovimiento = millis();
  }
  else if (valorX > LIMITE_ALTO)
  {
    moverPosicionMac(1);
    ultimoMovimiento = millis();
  }
  else if (valorY < LIMITE_BAJO)
  {
    cambiarDigitoMac(1);
    ultimoMovimiento = millis();
  }
  else if (valorY > LIMITE_ALTO)
  {
    cambiarDigitoMac(-1);
    ultimoMovimiento = millis();
  }
}


/* Escritura y scroll */

void leerJoystickChat(int valorX, int valorY)
{
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


/* Lectura del botón */

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
          presionLarga();
        }
        else
        {
          presionCorta();
        }
      }
    }
  }
}


/* Acción de una presión corta */

void presionCorta()
{
  if (estadoSistema == SELECCIONAR_ROL)
  {
    confirmarRol();
  }
  else if (estadoSistema == INGRESAR_MAC)
  {
    moverPosicionMac(1);
  }
  else if (estadoSistema == CHAT)
  {
    if (modoScroll)
    {
      modoScroll = false;
      mostrarChat();
    }
    else
    {
      seleccionarCaracter();
    }
  }
}


/* Acción de una presión larga */

void presionLarga()
{
  if (estadoSistema == SELECCIONAR_ROL)
  {
    confirmarRol();
  }
  else if (estadoSistema == INGRESAR_MAC)
  {
    conectarMaster();
  }
  else if (estadoSistema == CHAT)
  {
    modoScroll = !modoScroll;
    mostrarChat();
  }
}


/* Confirmar MASTER o SLAVE */

void confirmarRol()
{
  esMaster = opcionRol == 1;

  pantalla.clearDisplay();
  pantalla.setCursor(0, 0);
  pantalla.print("Iniciando Bluetooth");
  pantalla.display();

  if (esMaster)
  {
    SerialBT.begin("ESP32_CHAT_MASTER", true);
  }
  else
  {
    SerialBT.begin("ESP32_CHAT_SLAVE");
  }

  macLocal = SerialBT.getBtAddressString();

  if (esMaster)
  {
    estadoSistema = INGRESAR_MAC;
    mostrarIngresoMac();
  }
  else
  {
    estadoSistema = ESPERAR_CONEXION;
    mostrarEspera();
  }
}


/* Mostrar selección de rol */

void mostrarSeleccionRol()
{
  pantalla.clearDisplay();

  pantalla.setCursor(0, 0);
  pantalla.print("SELECCIONAR MODO");

  pantalla.drawLine(
    0,
    10,
    ANCHO_PANTALLA - 1,
    10,
    SSD1306_WHITE
  );

  pantalla.setCursor(10, 24);

  if (opcionRol == 0)
  {
    pantalla.print("> SLAVE");
  }
  else
  {
    pantalla.print("  SLAVE");
  }

  pantalla.setCursor(10, 40);

  if (opcionRol == 1)
  {
    pantalla.print("> MASTER");
  }
  else
  {
    pantalla.print("  MASTER");
  }

  pantalla.setCursor(0, 56);
  pantalla.print("Presionar: aceptar");

  pantalla.display();
}


/* Cambiar posición de la MAC */

void moverPosicionMac(int cambio)
{
  posicionMac += cambio;

  if (posicionMac < 0)
  {
    posicionMac = 11;
  }

  if (posicionMac > 11)
  {
    posicionMac = 0;
  }

  mostrarIngresoMac();
}


/* Cambiar número hexadecimal */

void cambiarDigitoMac(int cambio)
{
  char actual = digitosMac[posicionMac];

  int posicionHex = 0;

  for (int i = 0; i < 16; i++)
  {
    if (caracteresHex[i] == actual)
    {
      posicionHex = i;
      break;
    }
  }

  posicionHex += cambio;

  if (posicionHex < 0)
  {
    posicionHex = 15;
  }

  if (posicionHex > 15)
  {
    posicionHex = 0;
  }

  digitosMac[posicionMac] =
    caracteresHex[posicionHex];

  errorConexion = false;

  mostrarIngresoMac();
}


/* Formar la dirección MAC */

String obtenerMacFormateada()
{
  String mac = "";

  for (int i = 0; i < 12; i++)
  {
    mac += digitosMac[i];

    if (i % 2 == 1 && i < 11)
    {
      mac += ':';
    }
  }

  return mac;
}


/* Convertir carácter hexadecimal */

int convertirHexadecimal(char caracter)
{
  if (caracter >= '0' && caracter <= '9')
  {
    return caracter - '0';
  }

  if (caracter >= 'A' && caracter <= 'F')
  {
    return caracter - 'A' + 10;
  }

  return 0;
}


/* Convertir la MAC a seis bytes */

void convertirMacABytes(uint8_t direccion[6])
{
  for (int i = 0; i < 6; i++)
  {
    int parteAlta =
      convertirHexadecimal(digitosMac[i * 2]);

    int parteBaja =
      convertirHexadecimal(digitosMac[i * 2 + 1]);

    direccion[i] =
      parteAlta * 16 + parteBaja;
  }
}


/* Mostrar captura de la MAC */

void mostrarIngresoMac()
{
  pantalla.clearDisplay();

  pantalla.setCursor(0, 0);
  pantalla.print("MODO: MASTER");

  pantalla.setCursor(0, 10);
  pantalla.print("MI MAC:");

  pantalla.setCursor(0, 18);
  pantalla.print(macLocal);

  pantalla.setCursor(0, 32);
  pantalla.print("MAC SLAVE:");

  String macDestino =
    obtenerMacFormateada();

  pantalla.setCursor(0, 40);
  pantalla.print(macDestino);

  int posicionPantalla =
    posicionMac + posicionMac / 2;

  int posicionX =
    posicionPantalla * 6;

  pantalla.drawLine(
    posicionX,
    49,
    posicionX + 4,
    49,
    SSD1306_WHITE
  );

  pantalla.setCursor(0, 56);

  if (errorConexion)
  {
    pantalla.print("Error. Largo: reint.");
  }
  else
  {
    pantalla.print("Largo: conectar");
  }

  pantalla.display();
}


/* Conectar el MASTER */

void conectarMaster()
{
  uint8_t direccionRemota[6];

  convertirMacABytes(direccionRemota);

  pantalla.clearDisplay();

  pantalla.setCursor(0, 0);
  pantalla.print("Conectando...");

  pantalla.setCursor(0, 16);
  pantalla.print(obtenerMacFormateada());

  pantalla.display();

  bool conectado =
    SerialBT.connect(direccionRemota);

  if (!conectado)
  {
    conectado = SerialBT.connected(5000);
  }

  if (conectado)
  {
    estadoSistema = CHAT;

    agregarMensajeChat(
      "",
      "Conexion establecida"
    );

    mostrarChat();
  }
  else
  {
    errorConexion = true;
    mostrarIngresoMac();
  }
}


/* Mostrar espera del SLAVE */

void mostrarEspera()
{
  pantalla.clearDisplay();

  pantalla.setCursor(0, 0);
  pantalla.print("MODO: SLAVE");

  pantalla.setCursor(0, 12);
  pantalla.print("MI MAC:");

  pantalla.setCursor(0, 22);
  pantalla.print(macLocal);

  pantalla.setCursor(0, 40);
  pantalla.print("Esperando MASTER");

  pantalla.setCursor(0, 52);
  pantalla.print("Use esta direccion");

  pantalla.display();
}


/* Revisar conexión */

void revisarConexion()
{
  if (estadoSistema == ESPERAR_CONEXION)
  {
    if (SerialBT.hasClient())
    {
      estadoSistema = CHAT;

      agregarMensajeChat(
        "",
        "Conexion establecida"
      );

      mostrarChat();
    }
  }
  else if (estadoSistema == CHAT)
  {
    if (!SerialBT.hasClient())
    {
      mensajeRecibido = "";

      if (esMaster)
      {
        estadoSistema = INGRESAR_MAC;
        errorConexion = true;
        mostrarIngresoMac();
      }
      else
      {
        estadoSistema = ESPERAR_CONEXION;
        mostrarEspera();
      }
    }
  }
}


/* Cambiar grupo de caracteres */

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

  mostrarChat();
}


/* Cambiar carácter */

void cambiarCaracter(int cambio)
{
  int cantidadCaracteres =
    strlen(grupos[grupoActual]);

  caracterActual += cambio;

  if (caracterActual < 0)
  {
    caracterActual =
      cantidadCaracteres - 1;
  }

  if (caracterActual >= cantidadCaracteres)
  {
    caracterActual = 0;
  }

  mostrarChat();
}


/* Seleccionar carácter */

void seleccionarCaracter()
{
  char caracter =
    grupos[grupoActual][caracterActual];

  if (caracter == '<')
  {
    borrarCaracter();
  }
  else if (caracter == '>')
  {
    enviarMensaje();
  }
  else
  {
    if (mensajeEscrito.length() <
        MAXIMO_MENSAJE)
    {
      mensajeEscrito += caracter;
    }
  }

  mostrarChat();
}


/* Borrar el último carácter */

void borrarCaracter()
{
  if (mensajeEscrito.length() > 0)
  {
    mensajeEscrito.remove(
      mensajeEscrito.length() - 1
    );
  }
}


/* Enviar mensaje */

void enviarMensaje()
{
  if (mensajeEscrito.length() == 0)
  {
    return;
  }

  SerialBT.println(mensajeEscrito);

  agregarMensajeChat(
    "YO: ",
    mensajeEscrito
  );

  mensajeEscrito = "";

  colocarUltimasLineas();
}


/* Recibir mensaje */

void recibirBluetooth()
{
  if (estadoSistema != CHAT)
  {
    return;
  }

  bool mensajeCompleto = false;

  while (SerialBT.available())
  {
    char caracter = SerialBT.read();

    if (caracter == '\n')
    {
      if (mensajeRecibido.length() > 0)
      {
        agregarMensajeChat(
          "OTRO: ",
          mensajeRecibido
        );

        mensajeRecibido = "";
        mensajeCompleto = true;
      }
    }
    else if (caracter != '\r')
    {
      if (mensajeRecibido.length() <
          MAXIMO_MENSAJE)
      {
        mensajeRecibido += caracter;
      }
    }
  }

  if (mensajeCompleto)
  {
    colocarUltimasLineas();
    mostrarChat();
  }
}


/* Agregar un mensaje al historial */

void agregarMensajeChat(
  String prefijo,
  String mensaje)
{
  String mensajeCompleto =
    prefijo + mensaje;

  while (mensajeCompleto.length() > 0)
  {
    String parte =
      mensajeCompleto.substring(
        0,
        CARACTERES_POR_LINEA
      );

    agregarLineaChat(parte);

    mensajeCompleto.remove(
      0,
      parte.length()
    );
  }

  colocarUltimasLineas();
}


/* Agregar una línea al historial */

void agregarLineaChat(String linea)
{
  if (totalLineasChat <
      MAXIMO_LINEAS_CHAT)
  {
    lineasChat[totalLineasChat] = linea;
    totalLineasChat++;
  }
  else
  {
    for (int i = 0;
         i < MAXIMO_LINEAS_CHAT - 1;
         i++)
    {
      lineasChat[i] =
        lineasChat[i + 1];
    }

    lineasChat[MAXIMO_LINEAS_CHAT - 1] =
      linea;
  }
}


/* Mostrar las últimas líneas */

void colocarUltimasLineas()
{
  primeraLineaChat =
    totalLineasChat - LINEAS_VISIBLES;

  if (primeraLineaChat < 0)
  {
    primeraLineaChat = 0;
  }
}


/* Mover el scroll */

void moverScroll(int movimiento)
{
  int ultimaPosicion =
    totalLineasChat - LINEAS_VISIBLES;

  if (ultimaPosicion < 0)
  {
    ultimaPosicion = 0;
  }

  primeraLineaChat += movimiento;

  if (primeraLineaChat < 0)
  {
    primeraLineaChat = 0;
  }

  if (primeraLineaChat > ultimaPosicion)
  {
    primeraLineaChat = ultimaPosicion;
  }

  mostrarChat();
}


/* Mostrar la pantalla del chat */

void mostrarChat()
{
  pantalla.clearDisplay();

  pantalla.setCursor(0, 0);

  if (esMaster)
  {
    pantalla.print("MASTER CONECTADO");
  }
  else
  {
    pantalla.print("SLAVE CONECTADO");
  }

  pantalla.setCursor(0, 8);

  if (modoScroll)
  {
    pantalla.print("SCROLL ");

    pantalla.print(
      primeraLineaChat + 1
    );

    pantalla.print("/");

    pantalla.print(totalLineasChat);
  }
  else
  {
    mostrarCaracterSeleccionado();
  }

  pantalla.drawLine(
    0,
    17,
    ANCHO_PANTALLA - 1,
    17,
    SSD1306_WHITE
  );

  mostrarHistorial();

  mostrarMensajeEscrito();

  pantalla.display();
}


/* Mostrar carácter seleccionado */

void mostrarCaracterSeleccionado()
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
  else if (caracter == '>')
  {
    pantalla.print("ENVIAR");
  }
  else
  {
    pantalla.print(caracter);
  }
}


/* Mostrar historial */

void mostrarHistorial()
{
  if (totalLineasChat == 0)
  {
    pantalla.setCursor(0, 24);
    pantalla.print("Sin mensajes");
    return;
  }

  for (int linea = 0;
       linea < LINEAS_VISIBLES;
       linea++)
  {
    int posicion =
      primeraLineaChat + linea;

    if (posicion >= totalLineasChat)
    {
      break;
    }

    pantalla.setCursor(
      0,
      20 + linea * 8
    );

    pantalla.print(
      lineasChat[posicion]
    );
  }
}


/* Mostrar mensaje que se está escribiendo */

void mostrarMensajeEscrito()
{
  String textoVisible = mensajeEscrito;

  if (textoVisible.length() > 19)
  {
    textoVisible =
      textoVisible.substring(
        textoVisible.length() - 19
      );
  }

  pantalla.setCursor(0, 56);
  pantalla.print(">");
  pantalla.print(textoVisible);
}
```

---

## Actividades

* Investigar qué es una dirección MAC.
* Investigar la diferencia entre `MASTER` y `SLAVE`.
* Investigar el funcionamiento de `SerialBT.begin()`.
* Investigar el funcionamiento de `SerialBT.connect()`.
* Investigar el funcionamiento de `SerialBT.available()`.
* Investigar el funcionamiento de `SerialBT.println()`.
* Explicar cómo se convierte una dirección MAC de texto a seis bytes.
* Modificar el programa para mostrar el nombre del alumno al enviar un mensaje.
* Realizar una biblioteca a partir de las funciones de escritura y chat.
