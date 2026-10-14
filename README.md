# 🧟 Momias de Guanajuato: La Leyenda

> Un juego de plataformas con física real, construido **100 % con TDD** en Flutter/Dart.

🤠 Salta · 🧟 Esquiva momias · 💀 Colecciona calaveritas · 💋 Llega al Callejón del Beso

*Desarrollo Móvil Integral: proyecto complementario de práctica TDD.*
**Ingeniería en Desarrollo y Gestión de Software (GIDS6092), UTNG.**

---

## 📸 Capturas

| Inicio del nivel | Saltando a la plataforma (+50 💀) | Momia patrullando | Fin de la partida |
|:---:|:---:|:---:|:---:|
| ![Inicio](docs/capturas/01_inicio.png) | ![Salto](docs/capturas/02_salto.png) | ![Momias](docs/capturas/03_momias.png) | ![Resultado](docs/capturas/04_resultado.png) |

*Capturas tomadas en el navegador con `flutter run -d web-server` (vista de 390 × 760, tamaño de un celular).*

---

## 🎮 ¿Qué es?

Un explorador debe cruzar los callejones de Guanajuato saltando entre plataformas, esquivando o derrotando momias que patrullan de un lado a otro y recolectando calaveritas de azúcar, hasta llegar al legendario **Callejón del Beso**.

Es el proyecto más ambicioso de la serie: un plataformas de verdad, con gravedad, salto, colisiones y un enemigo con una inteligencia artificial sencilla.

A diferencia de los proyectos anteriores (por turnos o con movimientos discretos), un plataformas se actualiza unas 60 veces por segundo. Aun así, **toda la física y las reglas del juego son funciones puras de Dart, probadas con TDD**, escritas antes de tocar un solo widget de Flutter.

### Mecánicas

- El personaje cae por gravedad y aterriza sobre las plataformas.
- Los botones en pantalla lo mueven a la izquierda o a la derecha y lo hacen saltar.
- Las momias patrullan entre dos límites fijos e invierten su dirección al llegar a ellos.
- **Saltar encima** de una momia la derrota; **chocar de lado** con ella le quita una vida al jugador.
- Cada calaverita recolectada suma **50 puntos**.
- Llegar al final del nivel (el Callejón del Beso) gana la partida; quedarse sin vidas (empiezas con 3) la pierde.

### Controles

| Botón | Acción |
|:---:|---|
| ⬅️ / ➡️ (mantener presionado) | Caminar |
| ⬆️ | Saltar (solo si estás en el suelo) |

---

## 🗂️ Estructura del proyecto

```
lib/
  logica/                      ← Dart puro: sin Flutter, 100 % probado
    rectangulo.dart            colisión AABB
    personaje.dart             posición, velocidad vertical, si está en el suelo
    momia.dart                 patrullaje entre dos límites
    calaverita.dart            coleccionable
    motor_fisica.dart          gravedad, salto, aterrizaje, límites, meta
    motor_combate.dart         pisotón contra golpe lateral
    estado_juego.dart          jugando / ganado / perdido
    motor_juego.dart           orquesta todo en cada cuadro
  presentation/
    juego_screen.dart          cámara, sprites y controles
  main.dart
test/
  rectangulo_test.dart
  motor_fisica_test.dart
  momia_test.dart
  motor_combate_test.dart
  motor_juego_test.dart
```

> ✅ **Buena práctica:** toda la carpeta `lib/logica/` es Dart puro. Un juego con física a 60 cuadros por segundo es justo el tipo de código donde los errores son más difíciles de ver a simple vista, y donde las pruebas aportan más valor.

### Arquitectura

```
┌──────────────────────────┐   cada 16 ms    ┌─────────────────────────┐
│  JuegoScreen (Flutter)   │ ──────────────▶ │  MotorJuego.mover(dx)   │
│  cámara · sprites · UI   │ ◀────────────── │  (estado del juego)     │
└──────────────────────────┘   setState      └───────────┬─────────────┘
                                                         │
             ┌───────────────────┬───────────────────────┼──────────────────┐
             ▼                   ▼                       ▼                  ▼
       MotorFisica         MotorCombate            Momia.patrullar   recolectarSiCorresponde
  gravedad · salto · piso  pisotón vs golpe        IA de ida y vuelta     calaveritas
             └───────────────────┴──────── Rectangulo.intersectaCon (AABB) ─┘
```

La interfaz **no decide** colisiones ni puntajes: solo traduce el estado de `MotorJuego` a píxeles.

---

## 🔴🟢 Desarrollo con TDD: las 10 rondas

Cada ronda siguió el ciclo **Rojo → Verde**: primero una prueba que falla y después el mínimo código para que pase.

| Ronda | Tema | Prueba(s) | Implementación |
|:---:|---|---|---|
| 1 | Colisión AABB | `rectangulo_test.dart` | `Rectangulo.intersectaCon` |
| 2 | Gravedad y salto | `motor_fisica_test.dart` | `MotorFisica.aplicarGravedad`, `saltar` |
| 3 | Aterrizar sobre una plataforma | `motor_fisica_test.dart` | `MotorFisica.resolverColisionConPiso` |
| 4 | Movimiento horizontal con límites | `motor_fisica_test.dart` | `MotorFisica.moverHorizontal` |
| 5 | La momia patrulla | `momia_test.dart` | `Momia.patrullar` |
| 6 | Combate: pisotón contra golpe | `motor_combate_test.dart` | `MotorCombate.resolverColision` |
| 7 | Recolectar calaveritas | `motor_juego_test.dart` | `MotorJuego.recolectarSiCorresponde` |
| 8 | Detectar la meta | `motor_fisica_test.dart` | `MotorFisica.haLlegadoALaMeta` |
| 9 | Vidas y Game Over | `motor_juego_test.dart` | `MotorJuego._perderVida` |
| 10 | El bucle que integra todo | `motor_juego_test.dart` (integración) | `MotorJuego.mover` |

### Ronda 1: colisión entre rectángulos (AABB)

Dos rectángulos se traslapan si lo hacen **en el eje X y en el eje Y al mismo tiempo**. Este algoritmo (*Axis-Aligned Bounding Box*) es la base de casi toda la detección de colisiones en 2D, y aquí se usa para **todas** las colisiones: piso, momias y calaveritas.

```dart
bool intersectaCon(Rectangulo otro) {
  return x < otro.x + otro.ancho &&
      x + ancho > otro.x &&
      y < otro.y + otro.alto &&
      y + alto > otro.y;
}
```

### Ronda 2: gravedad y salto

`gravedad = 0.8` se suma a la velocidad vertical en cada cuadro. `impulsoSalto = -12` solo se aplica si el personaje está en el suelo, para que no pueda saltar en el aire.

### Ronda 3: aterrizar sobre una plataforma

`resolverColisionConPiso` recibe `yAnterior`, la posición del cuadro anterior.

> ⚠️ **Evita esto:** sin saber dónde estaba el personaje un instante antes, no hay forma de distinguir *"viene cayendo desde arriba"* de *"ya estaba atravesando la plataforma por dentro"*. `yAnterior` es un **parámetro-costura**, igual que el `Random` inyectado de Nieves 2048.

### Ronda 4: movimiento horizontal con límites

`moverHorizontal` desplaza al personaje y lo mantiene dentro de `[0, anchoNivel - ancho]`.

### Ronda 5: la momia patrulla entre dos límites

Es la "inteligencia artificial" más simple que existe, un patrullaje de ida y vuelta. Aun así merece pruebas: un error de signo (`+` en vez de `-`) pasa desapercibido a simple vista y solo se nota después de jugar varios minutos.

### Ronda 6: combate

```dart
final cayendoSobreLaMomia = personaje.velocidadY > 0 &&
    (personaje.y + personaje.alto) <= (momia.y + momia.alto / 2);
```

> ✅ **Buena práctica:** para que cuente como pisotón no basta con estar cayendo (`velocidadY > 0`). También se exige que el pie del personaje esté en la **mitad superior** de la momia. Sin esa segunda condición, un choque lateral mientras el personaje ya iba cayendo contaría como una momia derrotada. Al pisarla, el personaje rebota con la mitad del impulso de salto.

### Ronda 7: recolectar calaveritas

Cada calaverita da 50 puntos y se marca como `recolectada`, así que no vuelve a sumar.

### Ronda 8: detectar la meta

`haLlegadoALaMeta` es verdadero cuando `personaje.x >= posicionMeta`.

### Ronda 9: vidas

Un golpe lateral resta una vida y regresa al personaje a su posición inicial. Al llegar a 0 vidas, el estado cambia a `EstadoJuego.perdido`.

### Ronda 10: `MotorJuego.mover()`

```dart
void mover(double deltaX) {
  if (estado != EstadoJuego.jugando) return;
  final yAnterior = personaje.y;
  MotorFisica.moverHorizontal(personaje, deltaX, anchoNivel);   // 1. física
  MotorFisica.aplicarGravedad(personaje);
  MotorFisica.resolverColisionConPiso(personaje, plataformas, yAnterior);
  for (final momia in momias) {                                 // 2. combate
    momia.patrullar();
    if (MotorCombate.resolverColision(personaje, momia) ==
        ResultadoColisionMomia.jugadorGolpeado) {
      _perderVida();
      return;
    }
  }
  puntaje += recolectarSiCorresponde(personaje, calaveritas);   // 3. recolección
  if (MotorFisica.haLlegadoALaMeta(personaje, posicionMeta)) {  // 4. meta
    estado = EstadoJuego.ganado;
  }
}
```

> ✅ **Buena práctica:** `mover()` es el **único** lugar que decide el orden de las reglas: física y piso, luego momias, luego calaveritas y al final la meta. Ese orden importa, y el propio código lo documenta línea por línea, respaldado por las pruebas de las rondas anteriores.

### Prueba de integración

`llegar a la meta cambia el estado a ganado` confirma que moverse, aplicar la física y detectar la meta funcionan juntos, sin tener que simular cientos de cuadros de animación.

---

## 🖼️ La interfaz: cámara lateral y sprites

- **Cámara:** `camaraX` centra al personaje en la ventana visible y se limita con `clamp` para no mostrar zonas fuera del nivel. Cada objeto se dibuja en su posición real menos `camaraX`, así que todo el mundo se desliza cuando el personaje avanza.
- **Plataformas:** en el rosa típico de las fachadas de Guanajuato (`#E91E63`).
- **Sprites:** el personaje 🤠, las momias 🧟 y las calaveritas 💀 se dibujan con emojis.
- **Bucle:** un `Timer.periodic` de 16 ms (unos 60 FPS) llama a `MotorJuego.mover`.
- **Fin de partida:** un diálogo animado (`ScaleTransition` con `Curves.elasticOut`) muestra el puntaje y el botón *Jugar de nuevo*.

---

## 🚀 Cómo ejecutarlo

Requisitos: [Flutter](https://docs.flutter.dev/get-started/install) (SDK de Dart ^3.13).

```bash
git clone https://github.com/SekiroK5/momias_guanajuato.git
cd momias_guanajuato
flutter pub get

# 1. Corre las pruebas (23 pruebas)
flutter test

# 2. Ejecuta el juego
flutter run -d chrome      # en el navegador
flutter run                # en un emulador o dispositivo conectado
```

---

## 🛠️ Correcciones respecto al manual

Al ejecutar las pruebas tal como venían en el manual, dos de ellas fallaban por los **datos de la prueba**, no por la lógica del juego. Las corregí y dejé un comentario en el código de cada una:

| Archivo | Prueba | Problema | Corrección |
|---|---|---|---|
| `motor_fisica_test.dart` | *el personaje aterriza sobre una plataforma…* | Con `y: 90` y `yAnterior: 85`, el pie anterior (85 + 32 = 117) ya estaba **debajo** de la plataforma (y = 100), así que no "venía cayendo desde arriba". | `y: 70`, `yAnterior: 65`: el pie pasa de 97 a 102 y cruza el borde. |
| `motor_combate_test.dart` | *caer sobre la mitad superior de una momia la derrota* | Con `y: 68`, el pie queda exactamente en y = 100, el borde superior de la momia. AABB no cuenta como colisión dos bordes que solo se tocan. | `y: 70`: el pie (102) se traslapa con la momia y sigue en su mitad superior (≤ 116). |
| `juego_screen.dart` | — | `const [Calaverita(...)]` no compila, porque `Calaverita` tiene un campo mutable (`recolectada`) y no tiene constructor `const`. | Se quitó `const` de esa lista. |

---

## ✅ Checklist del proyecto TDD

- [x] La colisión AABB (`Rectangulo`) se probó de forma aislada antes de usarse en cualquier otra parte.
- [x] `resolverColisionConPiso` usa la posición **anterior** del personaje, no solo la actual.
- [x] El criterio de pisotón exige caer **y** estar en la mitad superior de la momia, no solo tocarla.
- [x] `patrullar()` y el resto de la lógica sensible al tiempo son deterministas en las pruebas.
- [x] `MotorJuego.mover()` es el único lugar que decide el orden de física, combate, recolección y meta.
- [x] La interfaz no decide colisiones ni puntajes: solo traduce el estado de `MotorJuego` a píxeles.

---

## 🎯 Retos de extensión

- [ ] Colisión **lateral** del personaje contra las plataformas (para no atravesarlas de costado), con el mismo patrón de costura testeable de `resolverColisionConPiso`.
- [ ] Vidas visuales (corazones ❤️) en el HUD en lugar de un número.
- [ ] Una momia que patrulle **verticalmente** reutilizando la clase `Momia` con `limiteArriba` / `limiteAbajo`.
- [ ] Guardar el mejor puntaje con `shared_preferences`.
- [ ] Un segundo nivel más largo, con más plataformas y momias, usando exactamente el mismo `MotorJuego`.

---

🏆 **Insignia desbloqueada: Cinturón Negro de Proyecto.** Un plataformas completo con física, colisiones e IA de enemigos, 100 % guiado por pruebas.

**Autor:** Noé Chavero · [@SekiroK5](https://github.com/SekiroK5)
