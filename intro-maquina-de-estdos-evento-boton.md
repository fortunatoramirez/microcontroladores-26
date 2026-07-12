# Práctica: Introducción a una máquina de estados controlada por eventos de un botón

## Objetivo

Observar cómo un botón puede generar eventos y cómo esos eventos pueden modificar de manera persistente el estado de una máquina.

En esta práctica todavía no se ejecutarán acciones específicas en cada estado. Únicamente se observará, mediante el monitor serial, cómo cambia la variable `estadoMaquina`.

---

## Material necesario

* ESP32.
* Botón pulsador.
* Cables de conexión.
* Protoboard.
* Arduino IDE.
* Cable USB.

---

## Conexión del botón

El botón utilizará la resistencia interna `INPUT_PULLUP` del ESP32.

Conectar:

* Una terminal del botón al pin GPIO 15.
* La otra terminal del botón a GND.

Debido al uso de `INPUT_PULLUP`:

* Botón liberado: `HIGH`.
* Botón presionado: `LOW`.

---

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

## Explicación general del programa

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

---

## Procedimiento

1. Realizar la conexión del botón al GPIO 15 y a GND.
2. Copiar el programa en Arduino IDE.
3. Seleccionar la tarjeta ESP32 correspondiente.
4. Compilar y cargar el programa.
5. Abrir el monitor serial a `115200` baudios.
6. Realizar varias pulsaciones cortas.
7. Observar cómo aumenta el valor de `estadoMaquina`.
8. Mantener presionado el botón durante al menos 2 segundos.
9. Observar cómo la máquina regresa inmediatamente al estado `0`.
10. Mantener el botón presionado durante más de 2 segundos y comprobar que el evento largo solamente se genera una vez.

Una posible salida es:

```text
Estado Máquina: 0
Estado botón: 0
Estado botón: 1
Estado Máquina: 1
Estado botón: 1
Estado Máquina: 2
Estado botón: 1
Estado Máquina: 3
Estado botón: 2
Estado Máquina: 0
```

---

## Ejercicio único

Modifique la constante:

```cpp
const uint8_t TAMANO_MAQUINA = 6;
```

para que la máquina utilice únicamente los estados:

```text
0, 1, 2 y 3
```

Después:

1. Cargue nuevamente el programa.
2. Realice cinco pulsaciones cortas.
3. Anote la secuencia de estados mostrada en el monitor serial.
4. Compruebe que, después de llegar al último estado, la máquina regresa al estado `0`.
5. Realice una pulsación larga desde cualquier estado y compruebe que la máquina regresa al estado inicial.

### Resultado esperado

La secuencia debe ser:

```text
0 → 1 → 2 → 3 → 0 → 1
```

La pulsación larga debe producir:

```text
Cualquier estado → 0
```

---

## Conclusión

En esta práctica se observó que el botón no modifica directamente las salidas del sistema. Primero genera un evento y posteriormente la máquina procesa dicho evento.

Esta separación permite que:

* Cada pulsación se procese una sola vez.
* El estado de la máquina permanezca almacenado.
* La detección del botón sea independiente de las acciones realizadas por la máquina.
* Posteriormente se puedan asignar diferentes funciones a cada estado.
