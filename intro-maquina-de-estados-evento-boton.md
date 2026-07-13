# Práctica: Introducción a una máquina de estados controlada por eventos de un botón

## Eventos utilizados

El programa reconoce dos eventos:

| Evento          | Valor | Descripción                                               |
| --------------- | ----: | --------------------------------------------------------- |
| Pulsación corta |     1 | El botón se presiona y se libera antes de 2 segundos      |
| Pulsación larga |     2 | El botón permanece presionado durante al menos 2 segundos |

El valor `0` significa que no existe un evento pendiente por procesar.

---

## Comportamiento de la máquina

La máquina comienza en el estado `0`.

* Una pulsación corta avanza al siguiente estado.
* Una pulsación larga regresa la máquina al estado `0`.
* Cuando se alcanza el último estado, la siguiente pulsación corta regresa al estado inicial.

La variable `estadoMaquina` conserva su valor entre las diferentes ejecuciones de `loop()`.

---

## Código

```cpp
uint8_t estadoMaquina =             0;
uint8_t ultimoEstadoBoton =         0; 
uint32_t tiempoFlancoBoton =        0;
uint32_t tiempoInicioPresionado =   0;
bool boton =                        HIGH;
bool estadoAnteriorBoton =          HIGH;
bool botonPresionado =              false;
bool botonPresionadoLargo =         false;

const uint8_t TAMANO_MAQUINA =      6;


void actualizarBoton()
{
  boton = digitalRead(15);

  if(boton != estadoAnteriorBoton)
  {
    tiempoFlancoBoton = millis();
  }

  if(millis() - tiempoFlancoBoton > 50)
  {
    // Se reconoce el inicio de una pulsación
    if(boton == LOW && botonPresionado == false)
    {
      tiempoInicioPresionado = millis();
      botonPresionado = true;
    }

    // Se genera el evento de pulsación larga
    if(boton == LOW &&
       millis() - tiempoInicioPresionado >= 2000 &&
       botonPresionadoLargo == false)
    {
      ultimoEstadoBoton = 2;
      botonPresionadoLargo = true;
    }

    // Se reconoce la liberación del botón
    if(boton == HIGH && botonPresionado == true)
    {
      // Si no se produjo un evento largo,
      // la pulsación fue corta
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

  // El evento se elimina después de ser obtenido
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


void setup()
{
  Serial.begin(115200);

  pinMode(15, INPUT_PULLUP);

  Serial.print("Estado Máquina: ");
  Serial.println(estadoMaquina);

  Serial.print("Estado botón: ");
  Serial.println(ultimoEstadoBoton);
}


void loop()
{
  // Guardar el evento anterior para detectar cambios
  uint8_t estadoBotonAnterior = ultimoEstadoBoton;

  actualizarBoton();

  if(ultimoEstadoBoton != estadoBotonAnterior)
  {
    Serial.print("Estado botón: ");
    Serial.println(ultimoEstadoBoton);
  }

  // Guardar el estado anterior de la máquina
  uint8_t estadoMaquinaAnterior = estadoMaquina;

  actualizarMaquina();

  if(estadoMaquina != estadoMaquinaAnterior)
  {
    Serial.print("Estado Máquina: ");
    Serial.println(estadoMaquina);
  }
}
```

---

## Explicación

El programa se divide en tres partes principales.

### 1. Detección de eventos del botón

La función:

```cpp
actualizarBoton();
```

se ejecuta continuamente y determina si ocurrió:

* Una pulsación corta.
* Una pulsación larga.
* La liberación del botón.

Cuando ocurre un evento, se guarda en:

```cpp
ultimoEstadoBoton
```

El evento permanece almacenado hasta que la máquina lo obtiene.

---

### 2. Obtención y eliminación del evento

La función:

```cpp
obtenerUltimoEventoBoton();
```

realiza dos operaciones:

1. Regresa el último evento registrado.
2. Limpia el evento después de entregarlo.

```cpp
uint8_t obtenerUltimoEventoBoton()
{
  uint8_t ue = ultimoEstadoBoton;
  ultimoEstadoBoton = 0;
  return ue;
}
```

Esto evita que la máquina procese varias veces la misma pulsación.

---

### 3. Actualización de la máquina

La función:

```cpp
actualizarMaquina();
```

obtiene el evento del botón y modifica `estadoMaquina`.

```cpp
if(evento == 1)
{
  // Avanzar al siguiente estado
}
else if(evento == 2)
{
  // Regresar al estado inicial
}
```

La pulsación corta avanza un estado y la pulsación larga regresa al estado `0`.
