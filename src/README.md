Control software
====
/*
 * =====================================================================
 *  WRO FUTURE ENGINEERS - PROGRAMA DE COMPETENCIA v2.0
 * =====================================================================
 *  Robot aut贸nomo para WRO Future Engineers.
 *  Completa 3 vueltas detectando el sentido libre del pasillo,
 *  sigue la pared externa con control proporcional y se detiene
 *  en el punto de partida.
 *
 *  HARDWARE (NO MODIFICAR ESTOS PINES):
 *   - Arduino Mega 2560
 *   - Sensor frontal:    VL53L0X  (I2C: SDA=20, SCL=21) -> addr 0x30
 *   - Sensor izquierdo:  VL53L1X  (I2C, XSHUT=pin 4)    -> addr 0x2A
 *   - Sensor derecho:    VL53L1X  (I2C, XSHUT=pin 5)    -> addr 0x2B
 *   - Servo direcci贸n:   MG90S    (pin 9)
 *   - Motor tracci贸n:    Feetech STS3215 (Serial1 TX=18, RX=19)
 *
 *  INSTRUCCIONES DE USO:
 *   1. Ajustar los par谩metros de calibraci贸n de la secci贸n de abajo.
 *   2. Subir el programa al Arduino.
 *   3. Encender primero la bater铆a de sensores/motores.
 *   4. Encender la bater铆a del Arduino.
 *   5. El robot esperar谩 TIEMPO_INICIO ms y arrancar谩 solo.
 * =====================================================================
 */

#include <Wire.h>
#include <VL53L0X.h>
#include <VL53L1X.h>
#include <SCServo.h>
#include <Servo.h>

// =====================================================================
// *** ZONA DE CALIBRACI脫N - SOLO MODIFIQUEN ESTOS VALORES ***
// =====================================================================

// --- 1. DIRECCI脫N (Servo MG90S, pin 9) ---
const int PIN_SERVO_DIRECCION = 9;
const int STEER_CENTER    = 58;   // 脕ngulo para ir RECTO. Calibrar hasta que avance sin desviarse.
const int STEER_LEFT_MAX  = 100;  // 脕ngulo de giro m谩ximo a la IZQUIERDA.
const int STEER_RIGHT_MAX = 25;   // 脕ngulo de giro m谩ximo a la DERECHA.

// --- 2. MOTOR DE TRACCI脫N (Feetech STS3215, ID=1) ---
const int SERVO_ID           = 1;
const int MOTOR_SPEED_CRUISE = 3000;  // Velocidad de crucero (0 a 7000 aprox).
const int MOTOR_SPEED_TURN   = 2000;  // Velocidad m谩s lenta usada durante giros de 90掳.
const int MOTOR_SPEED_BACKUP = -2000; // Velocidad de retroceso (valor NEGATIVO).
const int MOTOR_ACCEL        = 50;    // Suavidad de aceleraci贸n (0=brusco, 100=suave).

// --- 3. DISTANCIAS DE DETECCI脫N (en mil铆metros) ---
// DIST_FIRST_WALL: el robot avanza y frena al detectar una pared a esta distancia.
//   Subir si frena muy lejos. Bajar si choca antes de frenar.
const int DIST_FIRST_WALL  = 400;

// DIST_CORNER_STOP: a qu茅 distancia frenar en las esquinas normales de cada vuelta.
//   Si el robot choca antes de girar, subir este valor.
const int DIST_CORNER_STOP = 300;

// TARGET_DISTANCE: distancia ideal a mantener de la pared externa durante la navegaci贸n (mm).
//   Si el robot va muy pegado a la pared, subir. Si se aleja mucho, bajar.
const int TARGET_DISTANCE  = 250;

// MARGEN_PARED: zona muerta (mm). El robot NO corrige si el error es menor a este valor.
//   Esto evita oscilaciones peque帽as. Si el robot zigzaguea, subir este valor.
const int MARGEN_PARED     = 60;

// --- 4. TIEMPOS DE MANIOBRA ---
// *** TIME_TURN_90: PAR脕METRO M脕S IMPORTANTE ***
// Tiempo en milisegundos que el motor empuja durante un giro de 90掳.
// Subir si el robot gira menos de 90掳. Bajar si gira m谩s de 90掳.
// Usar la Prueba 3 del programa pruebas_inge.ino para calibrar.
const unsigned long TIME_TURN_90 = 1500;

// TIME_BACKUP: tiempo que el robot retrocede antes del PRIMER giro para ganar espacio.
//   Si toca la pared durante el primer giro, subir este valor.
const unsigned long TIME_BACKUP  = 800;

// --- 5. CONTROL PROPORCIONAL DE DIRECCI脫N (Seguimiento de Pared) ---
// KP: agresividad de la correcci贸n. Empezar con 0.10 e ir ajustando.
//   Muy peque帽o (0.05): correcciones lentas, se aleja/acerca gradualmente.
//   Muy grande (0.30):  correcciones bruscas, puede zigzaguear.
const float KP = 0.10;

// --- 6. ALGORITMO DE TOMA DE DECISI脫N INICIAL ---
// Diferencia m铆nima entre los sensores lateral izquierdo y derecho para
// considerar que la decisi贸n es "clara" (en mm).
// Regla pr谩ctica: ~25% del ancho del pasillo de la pista.
// Pasillo de 400mm -> usar 100. Pasillo de 600mm -> usar 150.
const float UMBRAL_CLARIDAD = 100.0;

// --- 7. TIEMPO DE INICIO ---
// Cu谩ntos milisegundos espera el robot antes de arrancar (desde que enciende).
// Usar para posicionarlo en la pista sin apuros.
const unsigned long TIEMPO_INICIO_MS = 3000;

// =====================================================================
// PINES DE HARDWARE (No modificar - corresponde al cableado)
// =====================================================================
#define MOTOR_SERIAL Serial1 // TX=18 (va a RX del motor), RX=19 (va a TX del motor)
#define DEBUG_SERIAL Serial  // USB para monitoreo en computadora

const int xshutSensorIzq = 4; // Pin XSHUT del sensor VL53L1X izquierdo
const int xshutSensorDer = 5; // Pin XSHUT del sensor VL53L1X derecho

// =====================================================================
// OBJETOS Y VARIABLES GLOBALES (Internas del programa)
// =====================================================================
VL53L0X sensorFront;
VL53L1X sensorIzq;
VL53L1X sensorDer;

bool sensorFrontListo = false;
bool sensorIzqListo   = false;
bool sensorDerListo   = false;

SMS_STS sts;
Servo   servoDireccion;

// Estados de la m谩quina de navegaci贸n
enum EstadoRobot {
  INICIO,           // Cuenta regresiva antes de arrancar
  AVANZAR_INICIAL,  // Avanza en l铆nea recta buscando la primera esquina
  DECIDIR_SENTIDO,  // Muestrea sensores para decidir el sentido de la pista
  RETROCEDER,       // Retrocede para tener espacio en el primer giro
  GIRAR_PRIMERO,    // Ejecuta el primer giro de 90掳
  AVANZAR_PARED,    // Navegaci贸n normal: seguimiento de pared externa
  DETENER_ESQUINA,  // Frena al llegar a una esquina
  GIRAR_ESQUINA,    // Ejecuta un giro de 90掳 en esquina
  RECTA_FINAL,      // 脷ltimo pasillo antes de la llegada
  COMPLETADO        // Fin de carrera. Robot detenido.
};

EstadoRobot estadoActual = INICIO;

// Variables de control de misi贸n
bool girarALaDerecha    = true;  // true = circuito horario, false = antihorario
bool paredExternaEsDer  = false; // true = pared externa est谩 a la DERECHA del robot
int  totalTurns         = 0;
unsigned long stateTimer        = 0;
unsigned long startTime         = 0;
unsigned long timeToFirstCorner = 0; // Tiempo que tard贸 en llegar a la primera esquina

// =====================================================================
// SISTEMA DE RECUPERACI脫N AUTOM脕TICA DE SENSORES
// Reintenta la inicializaci贸n completa si alg煤n sensor falla.
// =====================================================================

void inicializarTodosLosSensores() {
  // 1. Apagar sensores laterales para que el frontal pueda iniciar sin conflicto en 0x29
  digitalWrite(xshutSensorIzq, LOW);
  digitalWrite(xshutSensorDer, LOW);
  delay(150);

  // 2. Sensor Frontal VL53L0X (direcci贸n asignada: 0x30)
  sensorFrontListo = false;
  sensorFront.setTimeout(200);
  // Intentar en 0x30 (ya asignado si reinici贸 en caliente) y en 0x29 (direcci贸n de f谩brica)
  sensorFront.setAddress(0x30);
  if (sensorFront.init()) {
    sensorFrontListo = true;
  } else {
    sensorFront.setAddress(0x29);
    if (sensorFront.init()) {
      sensorFront.setAddress(0x30);
      sensorFrontListo = true;
    }
  }
  if (sensorFrontListo) sensorFront.startContinuous(50);

  // 3. Sensor Izquierdo VL53L1X (direcci贸n asignada: 0x2A)
  digitalWrite(xshutSensorIzq, HIGH);
  delay(150);
  sensorIzqListo = false;
  sensorIzq.setTimeout(200);
  if (sensorIzq.init()) {
    sensorIzq.setAddress(0x2A);
    sensorIzq.setDistanceMode(VL53L1X::Long);
    sensorIzq.setMeasurementTimingBudget(40000);
    sensorIzq.startContinuous(40);
    sensorIzqListo = true;
  }

  // 4. Sensor Derecho VL53L1X (direcci贸n asignada: 0x2B)
  digitalWrite(xshutSensorDer, HIGH);
  delay(150);
  sensorDerListo = false;
  sensorDer.setTimeout(200);
  if (sensorDer.init()) {
    sensorDer.setAddress(0x2B);
    sensorDer.setDistanceMode(VL53L1X::Long);
    sensorDer.setMeasurementTimingBudget(40000);
    sensorDer.startContinuous(40);
    sensorDerListo = true;
  }
}

// Verifica si alg煤n sensor tuvo un timeout o est谩 desconectado y lo recupera.
void verificarYRecuperarSensores() {
  // --- Frontal ---
  if (!sensorFrontListo || sensorFront.timeoutOccurred()) {
    // Apagar laterales brevemente para liberar el bus I2C
    digitalWrite(xshutSensorIzq, LOW);
    digitalWrite(xshutSensorDer, LOW);
    delay(50);
    sensorFrontListo = false;
    sensorFront.setTimeout(200);
    sensorFront.setAddress(0x30);
    if (sensorFront.init()) { sensorFrontListo = true; }
    else {
      sensorFront.setAddress(0x29);
      if (sensorFront.init()) { sensorFront.setAddress(0x30); sensorFrontListo = true; }
    }
    if (sensorFrontListo) sensorFront.startContinuous(50);
    if (sensorIzqListo) digitalWrite(xshutSensorIzq, HIGH);
    if (sensorDerListo) digitalWrite(xshutSensorDer, HIGH);
    delay(50);
  }

  // --- Izquierdo ---
  if (!sensorIzqListo || sensorIzq.timeoutOccurred()) {
    sensorIzqListo = false;
    digitalWrite(xshutSensorIzq, LOW); delay(50);
    digitalWrite(xshutSensorIzq, HIGH); delay(100);
    sensorIzq.setTimeout(200);
    if (sensorIzq.init()) {
      sensorIzq.setAddress(0x2A);
      sensorIzq.setDistanceMode(VL53L1X::Long);
      sensorIzq.setMeasurementTimingBudget(40000);
      sensorIzq.startContinuous(40);
      sensorIzqListo = true;
    }
  }

  // --- Derecho ---
  if (!sensorDerListo || sensorDer.timeoutOccurred()) {
    sensorDerListo = false;
    digitalWrite(xshutSensorDer, LOW); delay(50);
    digitalWrite(xshutSensorDer, HIGH); delay(100);
    sensorDer.setTimeout(200);
    if (sensorDer.init()) {
      sensorDer.setAddress(0x2B);
      sensorDer.setDistanceMode(VL53L1X::Long);
      sensorDer.setMeasurementTimingBudget(40000);
      sensorDer.startContinuous(40);
      sensorDerListo = true;
    }
  }
}

// =====================================================================
// LECTURAS R脕PIDAS DE SENSORES (Para el loop de navegaci贸n)
// =====================================================================

int leerFrontal() {
  if (!sensorFrontListo) { verificarYRecuperarSensores(); if (!sensorFrontListo) return 9999; }
  int v = sensorFront.readRangeContinuousMillimeters();
  if (sensorFront.timeoutOccurred() || v == 8190 || v == 8191) {
    verificarYRecuperarSensores();
    return 9999;
  }
  return v;
}

int leerLateral(VL53L1X &sensor, bool &listo) {
  if (!listo) { verificarYRecuperarSensores(); if (!listo) return 9999; }
  int v = sensor.read();
  if (sensor.timeoutOccurred() || v <= 0 || v == 8191 || v == 8190 || v > 4000) {
    verificarYRecuperarSensores();
    return 9999;
  }
  return v;
}

// =====================================================================
// ALGORITMO ESTAD脥STICO DE DECISI脫N DE SENTIDO DE GIRO
// Toma 15 muestras de cada sensor lateral, aplica Promedio Recortado
// (Trimmed Mean: descarta el 20% superior e inferior para filtrar ruidos)
// y compara usando UMBRAL_CLARIDAD.
// Retorna: 1 = DERECHA  |  2 = IZQUIERDA
// =====================================================================
int decidirSentidoGiro() {
  const int N = 15;
  int bufIzq[N], bufDer[N];
  int nIzq = 0, nDer = 0;

  DEBUG_SERIAL.println("\n-> Muestreo estad铆stico lateral (15 muestras)...");
  DEBUG_SERIAL.println("=====================================");

  for (int i = 0; i < N; i++) {
    verificarYRecuperarSensores();
    int lI = -1, lD = -1;

    if (sensorIzqListo) {
      int v = sensorIzq.read();
      if (!sensorIzq.timeoutOccurred() && v > 0 && v < 4000 && v != 8191 && v != 8190) {
        bufIzq[nIzq++] = v;
        lI = v;
      }
    }
    if (sensorDerListo) {
      int v = sensorDer.read();
      if (!sensorDer.timeoutOccurred() && v > 0 && v < 4000 && v != 8191 && v != 8190) {
        bufDer[nDer++] = v;
        lD = v;
      }
    }

    // Imprimir cada muestra en tiempo real para verificaci贸n
    DEBUG_SERIAL.print("M"); if (i+1 < 10) DEBUG_SERIAL.print(" ");
    DEBUG_SERIAL.print(i + 1); DEBUG_SERIAL.print("/15  Izq: ");
    if (lI >= 0) { DEBUG_SERIAL.print(lI); DEBUG_SERIAL.print(" mm"); } else DEBUG_SERIAL.print("ERROR");
    DEBUG_SERIAL.print("  |  Der: ");
    if (lD >= 0) { DEBUG_SERIAL.print(lD); DEBUG_SERIAL.print(" mm"); } else DEBUG_SERIAL.print("ERROR");
    DEBUG_SERIAL.println();
    delay(50);
  }
  DEBUG_SERIAL.println("=====================================");

  // --- Ordenar buffers (Bubble Sort) para el promedio recortado ---
  for (int i = 0; i < nIzq - 1; i++)
    for (int j = i + 1; j < nIzq; j++)
      if (bufIzq[i] > bufIzq[j]) { int t = bufIzq[i]; bufIzq[i] = bufIzq[j]; bufIzq[j] = t; }

  for (int i = 0; i < nDer - 1; i++)
    for (int j = i + 1; j < nDer; j++)
      if (bufDer[i] > bufDer[j]) { int t = bufDer[i]; bufDer[i] = bufDer[j]; bufDer[j] = t; }

  // --- Calcular Promedio Recortado (quitar 20% de extremos) ---
  float mediaIzq = 9999.0f, mediaDer = 9999.0f;

  if (nIzq >= 5) {
    int ini = nIzq / 5, fin = nIzq - ini;
    long s = 0; for (int i = ini; i < fin; i++) s += bufIzq[i];
    mediaIzq = (float)s / (fin - ini);
  } else if (nIzq > 0) {
    long s = 0; for (int i = 0; i < nIzq; i++) s += bufIzq[i];
    mediaIzq = (float)s / nIzq;
  }

  if (nDer >= 5) {
    int ini = nDer / 5, fin = nDer - ini;
    long s = 0; for (int i = ini; i < fin; i++) s += bufDer[i];
    mediaDer = (float)s / (fin - ini);
  } else if (nDer > 0) {
    long s = 0; for (int i = 0; i < nDer; i++) s += bufDer[i];
    mediaDer = (float)s / nDer;
  }

  DEBUG_SERIAL.print("   V谩lidas Izq: "); DEBUG_SERIAL.print(nIzq);
  DEBUG_SERIAL.print(" | Der: "); DEBUG_SERIAL.println(nDer);
  DEBUG_SERIAL.print("   Media Recortada -> Izq: "); DEBUG_SERIAL.print(mediaIzq);
  DEBUG_SERIAL.print(" mm | Der: "); DEBUG_SERIAL.print(mediaDer); DEBUG_SERIAL.println(" mm");
  float diff = mediaIzq - mediaDer;
  DEBUG_SERIAL.print("   Diferencia Izq-Der: "); DEBUG_SERIAL.print(diff);
  DEBUG_SERIAL.print(" mm  | Umbral: "); DEBUG_SERIAL.print(UMBRAL_CLARIDAD); DEBUG_SERIAL.println(" mm");

  // --- Casos especiales: un sensor sin lecturas v谩lidas ---
  if (nIzq == 0 && nDer > 0) {
    DEBUG_SERIAL.println("   -> DERECHA (Izquierdo sin lecturas -> derecha libre)");
    return 1;
  }
  if (nDer == 0 && nIzq > 0) {
    DEBUG_SERIAL.println("   -> IZQUIERDA (Derecho sin lecturas -> izquierda libre)");
    return 2;
  }

  // --- Decisi贸n por umbral claro ---
  if (diff > UMBRAL_CLARIDAD) {
    DEBUG_SERIAL.print("   -> DECISION CLARA: IZQUIERDA (diff="); DEBUG_SERIAL.print(diff); DEBUG_SERIAL.println("mm)");
    return 2;
  } else if (diff < -UMBRAL_CLARIDAD) {
    DEBUG_SERIAL.print("   -> DECISION CLARA: DERECHA (diff="); DEBUG_SERIAL.print(-diff); DEBUG_SERIAL.println("mm)");
    return 1;
  } else {
    // Margen peque帽o: elegir el lado con mayor distancia
    if (mediaIzq >= mediaDer) {
      DEBUG_SERIAL.print("   -> Por margen: IZQUIERDA (Izq="); DEBUG_SERIAL.print(mediaIzq);
      DEBUG_SERIAL.print(" > Der="); DEBUG_SERIAL.print(mediaDer); DEBUG_SERIAL.println(")");
      return 2;
    } else {
      DEBUG_SERIAL.print("   -> Por margen: DERECHA (Der="); DEBUG_SERIAL.print(mediaDer);
      DEBUG_SERIAL.print(" > Izq="); DEBUG_SERIAL.print(mediaIzq); DEBUG_SERIAL.println(")");
      return 1;
    }
  }
}

// =====================================================================
// SEGUIMIENTO DE PARED EXTERNA (Control Proporcional)
// Ajusta el 谩ngulo del servo para mantener TARGET_DISTANCE.
// Aplica una zona muerta de MARGEN_PARED mm para no corregir cuando
// el error es peque帽o (evita zigzagueo en tramos rectos).
// =====================================================================
void seguirPared(int distParedExterna) {
  if (distParedExterna >= 9999) {
    // Sin lectura v谩lida: ir recto
    servoDireccion.write(STEER_CENTER);
    return;
  }

  int error = distParedExterna - TARGET_DISTANCE;

  // Zona muerta: si el error es menor al margen, no corregir
  if (abs(error) < MARGEN_PARED) {
    servoDireccion.write(STEER_CENTER);
    return;
  }

  // Correcci贸n proporcional
  int correccion = (int)(error * KP);
  int angulo;

  if (paredExternaEsDer) {
    // Pared externa a la DERECHA del robot (sentido antihorario)
    // error > 0: estoy lejos de la pared 鈫� corregir hacia la derecha (谩ngulo m谩s peque帽o)
    // error < 0: estoy muy cerca 鈫� alejarme hacia la izquierda (谩ngulo m谩s grande)
    angulo = STEER_CENTER - correccion;
  } else {
    // Pared externa a la IZQUIERDA del robot (sentido horario)
    // error > 0: estoy lejos 鈫� corregir hacia la izquierda (谩ngulo m谩s grande)
    // error < 0: estoy cerca 鈫� alejarme hacia la derecha (谩ngulo m谩s peque帽o)
    angulo = STEER_CENTER + correccion;
  }

  // Limitar al rango f铆sico del servo
  angulo = constrain(angulo,
                     min(STEER_RIGHT_MAX, STEER_LEFT_MAX),
                     max(STEER_RIGHT_MAX, STEER_LEFT_MAX));
  servoDireccion.write(angulo);
}

// =====================================================================
// SETUP - Inicializaci贸n del sistema
// =====================================================================
void setup() {
  DEBUG_SERIAL.begin(115200);
  MOTOR_SERIAL.begin(1000000);
  sts.pSerial = &MOTOR_SERIAL;

  servoDireccion.attach(PIN_SERVO_DIRECCION);
  servoDireccion.write(STEER_CENTER);

  Wire.begin();
  Wire.setClock(400000);

  pinMode(xshutSensorIzq, OUTPUT);
  pinMode(xshutSensorDer, OUTPUT);
  digitalWrite(xshutSensorIzq, LOW);
  digitalWrite(xshutSensorDer, LOW);
  delay(300);

  DEBUG_SERIAL.println("=================================================");
  DEBUG_SERIAL.println("   WRO FUTURE ENGINEERS v2.0  -  COMPETENCIA     ");
  DEBUG_SERIAL.println("=================================================");
  DEBUG_SERIAL.println("Inicializando sensores...");
  DEBUG_SERIAL.println("(Si alguno falla, enciende la bateria de sensores)");

  // Bucle que reintenta hasta que todos los sensores respondan
  while (!sensorFrontListo || !sensorIzqListo || !sensorDerListo) {
    inicializarTodosLosSensores();
    if (sensorFrontListo && sensorIzqListo && sensorDerListo) break;

    DEBUG_SERIAL.print("[-] Estado: Frontal="); DEBUG_SERIAL.print(sensorFrontListo ? "OK" : "ERROR");
    DEBUG_SERIAL.print(" Izq=");               DEBUG_SERIAL.print(sensorIzqListo   ? "OK" : "ERROR");
    DEBUG_SERIAL.print(" Der=");               DEBUG_SERIAL.println(sensorDerListo ? "OK" : "ERROR");
    delay(2000); // Reintentar cada 2 segundos
  }
  DEBUG_SERIAL.println("[OK] Todos los sensores inicializados!");
  DEBUG_SERIAL.println("[OK] Frontal: 0x30 | Izquierdo: 0x2A | Derecho: 0x2B");

  sts.WriteSpe(SERVO_ID, 0, MOTOR_ACCEL);
  DEBUG_SERIAL.println("[OK] Motor detenido.");
  DEBUG_SERIAL.println("\n>>> COLOCA EL ROBOT EN EL PASILLO DE SALIDA <<<");
  DEBUG_SERIAL.print(">>> Arrancando en ");
  DEBUG_SERIAL.print(TIEMPO_INICIO_MS / 1000);
  DEBUG_SERIAL.println(" segundos... <<<\n");

  stateTimer = millis();
  estadoActual = INICIO;
}

// =====================================================================
// LOOP - M谩quina de estados de navegaci贸n aut贸noma
// =====================================================================
void loop() {
  unsigned long ahora = millis();

  // Lectura r谩pida de sensores en cada ciclo del loop
  int distFront = leerFrontal();
  int distIzq   = leerLateral(sensorIzq, sensorIzqListo);
  int distDer   = leerLateral(sensorDer,  sensorDerListo);

  // Telemetr铆a compacta en el monitor serial
  DEBUG_SERIAL.print("F:"); DEBUG_SERIAL.print(distFront);
  DEBUG_SERIAL.print(" I:"); DEBUG_SERIAL.print(distIzq);
  DEBUG_SERIAL.print(" D:"); DEBUG_SERIAL.print(distDer);
  DEBUG_SERIAL.print(" Giros:"); DEBUG_SERIAL.print(totalTurns);
  DEBUG_SERIAL.print("/12 Edo:");

  switch (estadoActual) {

    // ----------------------------------------------------------------