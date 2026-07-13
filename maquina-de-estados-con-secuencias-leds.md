# Práctica: Máquina de estados con secuencias de LED

En esta práctica se implementa una máquina de estados controlada mediante un botón. Una pulsación corta permite avanzar entre diferentes secuencias de iluminación, mientras que una pulsación larga regresa al estado inicial. Cada estado ejecuta un patrón diferente sobre cuatro LED: caminata de uno, encendido y apagado simultáneo, caminata de cero, mitades alternadas y encendido aleatorio normal e inverso.

```cpp
uint8_t estadoMaquina =             0;
uint8_t ultimoEstadoBoton =         0;
uint32_t tiempoFlancoBoton =        0;
uint32_t tiempoInicioPresionado =   0;
bool boton =                        HIGH;
bool estadoAnteriorBoton =          HIGH;
bool botonPresionado =              false;
bool botonPresionadoLargo =         false;

const uint8_t TAMANO_MAQUINA =      5;

uint8_t leds[] =                    {32, 33, 25, 26};
bool estadosLeds[] =                {LOW, LOW, LOW, LOW};

const uint8_t TOTAL_LEDS =          4;
const uint8_t BOTON_ENTRADA =       15;


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


void setup()
{
  Serial.begin(115200);

  pinMode(BOTON_ENTRADA, INPUT_PULLUP);

  for(uint8_t i = 0; i < TOTAL_LEDS; i++)
  {
    pinMode(leds[i], OUTPUT);
  }
}


void loop()
{
  actualizarBoton();
  actualizarMaquina();

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
