# Práctica: Creación de una biblioteca para una máquina de estados

## Pasos

1. Crear un nuevo programa en Arduino IDE.
2. Abrir el menú de los tres puntos y seleccionar **New Tab**.
3. Crear el archivo `MaquinaEstados.h`.
4. Crear otra pestaña llamada `MaquinaEstados.cpp`.
5. Colocar el programa principal en el archivo `.ino`.
6. Como los tres archivos se encuentran en la misma carpeta, importar la biblioteca con:

```cpp
#include "MaquinaEstados.h"
```

7. Compilar y cargar el programa al ESP32.
8. Para instalarla como biblioteca, crear la carpeta:

```text
Documentos/Arduino/libraries/MaquinaEstados
```

9. Copiar dentro de esa carpeta los archivos:

```text
MaquinaEstados.h
MaquinaEstados.cpp
```

10. Reiniciar Arduino IDE. En un nuevo programa, la biblioteca se podrá importar con:

```cpp
#include <MaquinaEstados.h>
```

---

## Archivo `ejemplo_maquina.ino`

```cpp
#include "MaquinaEstados.h"

uint8_t leds[] = {32, 33, 25, 26};

void setup()
{
  iniciarMaquina(15, leds, 4);
}

void loop()
{
  actualizarBoton();
  actualizarMaquina();

  uint8_t estadoMaquina = optenerEstadoMaquina();

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

---

## Archivo `MaquinaEstados.h`

```cpp
#ifndef MAQUINA_ESTADOS_H
#define MAQUINA_ESTADOS_H

#include <Arduino.h>

void iniciarMaquina(
  uint8_t pinEntrada,
  uint8_t pinesLeds[],
  uint8_t totalLeds
);

uint8_t optenerEstadoMaquina();

void actualizarBoton();
uint8_t obtenerUltimoEventoBoton();
void actualizarMaquina();

void prenderApagarTodos();
void caminataDeUno();
void caminataDeCero();
void prenderApagarMitadAlternada();
void prenderAleatorios();
void prenderAleatoriosInversa();

void prenderTodos();
void apagarTodos();
void actualizarLeds();

#endif
```

---

## Archivo `MaquinaEstados.cpp`

```cpp
#include "MaquinaEstados.h"


uint8_t estadoMaquina =             0;
uint8_t ultimoEstadoBoton =         0;

uint32_t tiempoFlancoBoton =        0;
uint32_t tiempoInicioPresionado =   0;

bool boton =                        HIGH;
bool estadoAnteriorBoton =          HIGH;
bool botonPresionado =              false;
bool botonPresionadoLargo =         false;

const uint8_t TAMANO_MAQUINA =      5;

static uint8_t leds[] =             {32, 33, 25, 26};
static bool estadosLeds[] =         {LOW, LOW, LOW, LOW};

static uint8_t TOTAL_LEDS =         4;
static uint8_t BOTON_ENTRADA =      15;


void iniciarMaquina(
  uint8_t pinEntrada,
  uint8_t pinesLeds[],
  uint8_t totalLeds
)
{
  BOTON_ENTRADA = pinEntrada;
  TOTAL_LEDS = totalLeds;

  pinMode(BOTON_ENTRADA, INPUT_PULLUP);

  for(uint8_t i = 0; i < TOTAL_LEDS; i++)
  {
    leds[i] = pinesLeds[i];
    pinMode(leds[i], OUTPUT);
  }
}


uint8_t optenerEstadoMaquina()
{
  return estadoMaquina;
}


void actualizarBoton()
{
  boton = digitalRead(BOTON_ENTRADA);

  if(boton != estadoAnteriorBoton)
  {
    tiempoFlancoBoton = millis();
  }

  if(millis() - tiempoFlancoBoton > 50)
  {
    if(boton == LOW && botonPresionado == false)
    {
      tiempoInicioPresionado = millis();
      botonPresionado = true;
    }

    if(boton == LOW &&
       millis() - tiempoInicioPresionado >= 2000 &&
       botonPresionadoLargo == false)
    {
      ultimoEstadoBoton = 2;
      botonPresionadoLargo = true;
    }

    if(boton == HIGH && botonPresionado == true)
    {
      if(botonPresionadoLargo == false)
      {
        ultimoEstadoBoton = 1;
      }

      botonPresionado = false;
      botonPresionadoLargo = false;
    }
  }

  estadoAnteriorBoton = boton;
}


uint8_t obtenerUltimoEventoBoton()
{
  uint8_t ue = ultimoEstadoBoton;

  ultimoEstadoBoton = 0;

  return ue;
}


void actualizarMaquina()
{
  uint8_t evento = obtenerUltimoEventoBoton();

  if(evento == 1)
  {
    if(estadoMaquina < TAMANO_MAQUINA)
    {
      estadoMaquina++;
    }
    else
    {
      estadoMaquina = 0;
    }
  }
  else if(evento == 2)
  {
    estadoMaquina = 0;
  }
}


void prenderApagarTodos()
{
  static uint32_t ts = 0;
  static bool estado = LOW;

  if(millis() - ts >= 100)
  {
    ts = millis();
    estado = !estado;

    for(uint8_t i = 0; i < TOTAL_LEDS; i++)
    {
      estadosLeds[i] = !estado;
    }

    actualizarLeds();
  }
}


void caminataDeUno()
{
  static uint32_t ts = 0;
  static uint32_t c = 0;
  static uint32_t ca = TOTAL_LEDS - 1;

  if(millis() - ts >= 100)
  {
    ts = millis();

    estadosLeds[c % TOTAL_LEDS] = HIGH;
    estadosLeds[ca % TOTAL_LEDS] = LOW;

    actualizarLeds();

    c++;
    ca++;
  }
}


void caminataDeCero()
{
  static uint32_t ts = 0;
  static uint32_t c = 0;
  static uint32_t ca = TOTAL_LEDS - 1;

  if(millis() - ts >= 100)
  {
    ts = millis();

    estadosLeds[c % TOTAL_LEDS] = LOW;
    estadosLeds[ca % TOTAL_LEDS] = HIGH;

    actualizarLeds();

    c++;
    ca++;
  }
}


void prenderApagarMitadAlternada()
{
  static uint32_t ts = 0;
  static bool m = false;
  static uint32_t mitad = TOTAL_LEDS / 2;

  if(millis() - ts >= 100)
  {
    ts = millis();
    m = !m;

    for(uint8_t i = 0; i < mitad; i++)
    {
      estadosLeds[i] = m;
    }

    for(uint8_t i = mitad; i < TOTAL_LEDS; i++)
    {
      estadosLeds[i] = !m;
    }

    actualizarLeds();
  }
}


void prenderAleatorios()
{
  static uint32_t ts = 0;
  static uint32_t led = 0;
  static uint32_t ledAnterior = 0;
  static bool primera = true;

  if(primera)
  {
    apagarTodos();
    primera = false;
  }

  if(millis() - ts >= 50)
  {
    ts = millis();

    estadosLeds[ledAnterior] = LOW;

    led = random(TOTAL_LEDS);

    estadosLeds[led] = HIGH;
    ledAnterior = led;

    actualizarLeds();
  }
}


void prenderAleatoriosInversa()
{
  static uint32_t ts = 0;
  static uint8_t led = 0;
  static uint8_t ledAnterior = 0;
  static bool primera = true;

  if(primera)
  {
    prenderTodos();
    primera = false;
  }

  if(millis() - ts >= 50)
  {
    ts = millis();

    estadosLeds[ledAnterior] = HIGH;

    led = random(TOTAL_LEDS);

    estadosLeds[led] = LOW;
    ledAnterior = led;

    actualizarLeds();
  }
}


void prenderTodos()
{
  for(uint8_t i = 0; i < TOTAL_LEDS; i++)
  {
    estadosLeds[i] = HIGH;
  }

  actualizarLeds();
}


void apagarTodos()
{
  for(uint8_t i = 0; i < TOTAL_LEDS; i++)
  {
    estadosLeds[i] = LOW;
  }

  actualizarLeds();
}


void actualizarLeds()
{
  for(uint8_t i = 0; i < TOTAL_LEDS; i++)
  {
    digitalWrite(leds[i], estadosLeds[i]);
  }
}
```
