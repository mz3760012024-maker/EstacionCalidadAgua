# Estación inteligente de monitoreo de calidad de agua

Proyecto de la cátedra **Programación III** (eje: medio ambiente). Mide temperatura, sólidos disueltos (TDS) y
turbidez del agua con sensores de bajo costo, los procesa en un Arduino, los envía por Wi-Fi con un ESP32 a una
aplicación **Java / Spring Boot** que los guarda en PostgreSQL, los clasifica con un modelo de aprendizaje
automático (k-NN) y detecta cambios bruscos. El resultado se ve en un panel web, en una pantalla LCD 16x2 y en un
semáforo de LEDs.

> **Alcance honesto:** los sensores económicos sirven para **comparar muestras y detectar cambios**, no para certificar
> potabilidad. Los resultados dependen de la calibración y de los datos que usted capture.

## Arquitectura

```
 Sensores ──► Arduino (hub) ──UART──► ESP32 ──Wi-Fi/HTTP──► Backend Spring Boot ──► PostgreSQL
 DS18B20        mediana, TDS          reenvía JSON          clasificador k-NN
 TDS            compensado,           y devuelve la         detector de anomalías
 Turbidez       LCD 16x2 + LEDs       respuesta             panel web (http://IP:8080)
                  ▲                                              │
                  └──────── "R,estado,ETIQUETA,anomalia" ◄───────┘
```

| Capa | Tecnología | Carpeta |
|---|---|---|
| Nodo de sensores | Arduino (C++) | `firmware/arduino_hub` |
| Puerta de enlace | ESP32 (C++) | `firmware/esp32_gateway` |
| Servidor, IA y panel | Java 17 + Spring Boot, JPA, PostgreSQL | `backend` |
| Datos de entrenamiento | CSV exportado desde el panel | `data` |
| Pruebas sin hardware | Java (un solo archivo) | `scripts/SimuladorNodo.java` |

## Estructura del repositorio

```
estacion-calidad-agua/
├── README.md
├── .gitignore
├── .env.example               # copiar a .env (no se sube a GitHub)
├── docker-compose.yml         # base de datos PostgreSQL
├── backend/
│   ├── pom.xml
│   ├── Dockerfile             # opcional, fase nube
│   └── src/
│       ├── main/java/com/estacionagua/
│       │   ├── EstacionAguaApplication.java
│       │   ├── model/Lectura.java
│       │   ├── repo/LecturaRepository.java
│       │   ├── ml/ClasificadorKnn.java
│       │   ├── ml/DetectorAnomalias.java
│       │   ├── service/LecturaService.java
│       │   └── web/{LecturaController,LecturaRequest}.java
│       ├── main/resources/
│       │   ├── application.properties
│       │   └── static/{index.html, css/, js/}
│       └── test/java/com/estacionagua/ml/   # pruebas JUnit
├── firmware/
│   ├── arduino_hub/arduino_hub.ino
│   └── esp32_gateway/{esp32_gateway.ino, secrets.example.h}
├── hardware/
│   ├── cableado/              # fotos y diagramas propios (Fritzing, PNG)
│   └── maqueta/               # fotos, planos y medidas de la maqueta
├── data/                      # CSV de muestras REALES etiquetadas
├── scripts/SimuladorNodo.java
└── docs/
    ├── cableado.md
    ├── bom.md                 # lista de materiales y presupuesto
    ├── calibracion.md
    ├── maqueta.md             # guía de maqueta y guion de demostración
    ├── propuesta/             # documento Word entregado al docente
    └── imagenes/
```

## Requisitos

- JDK 17 o superior (verifique con `java -version`) y Maven 3.9+ (NetBeans e IntelliJ lo incluyen).
- Docker (para PostgreSQL) o una instalación propia de PostgreSQL.
- Arduino IDE 2.x.
- Hardware: ver `docs/bom.md`.

## Inicio rápido SIN hardware (haga esto primero)

1. Clone el repositorio y entre en la carpeta.
2. Cree el archivo de configuración y levante la base de datos:
   ```bash
   cp .env.example .env
   docker compose up -d
   ```
3. Ejecute el backend. Para probar con el simulador, active el entrenamiento con datos simulados **solo en desarrollo**:
   ```bash
   cd backend
   ENTRENAR_CON_SIMULADOS=true mvn spring-boot:run
   ```
   (En Windows PowerShell: `$env:ENTRENAR_CON_SIMULADOS="true"; mvn spring-boot:run`.
   En NetBeans puede abrir la carpeta `backend` como proyecto Maven y ejecutar `EstacionAguaApplication`.)
4. Abra `http://localhost:8080` en el navegador.
5. En otra terminal, desde la raíz del repositorio, envíe datos simulados con captura de muestras:
   ```bash
   java scripts/SimuladorNodo.java http://localhost:8080 clave-dev-123 --captura
   ```
   Verá las lecturas en el panel; al terminar, el modelo queda entrenado y puede ejecutar el simulador sin `--captura` para
   ver la clasificación.
6. **Antes de usar sensores reales**, limpie los datos de prueba para que el modelo no se entrene con datos inventados:
   ```bash
   docker compose down -v && docker compose up -d
   ```
   y reinicie el backend **sin** `ENTRENAR_CON_SIMULADOS`. Las lecturas simuladas se marcan siempre como tales y el panel muestra
   un aviso; nunca las presente como mediciones reales.

## Montaje con hardware real

1. Arme el circuito siguiendo `docs/cableado.md` (incluye el divisor de voltaje Arduino → ESP32; **no lo omita**).
2. Arduino: instale las bibliotecas *OneWire*, *DallasTemperature* y *LiquidCrystal I2C* (Administrador de bibliotecas),
   abra `firmware/arduino_hub/arduino_hub.ino`, seleccione su placa y puerto y cargue el sketch.
3. ESP32: en el Gestor de placas instale **esp32 (Espressif Systems)**; seleccione *ESP32 Dev Module*.
   En `firmware/esp32_gateway/` copie `secrets.example.h` a `secrets.h` y complete la red Wi-Fi, la IP de su computadora
   (`ipconfig` en Windows, `ip a` en Linux) y la `API_KEY` (igual a la de `.env`). Cargue `esp32_gateway.ino`.
4. La computadora y el ESP32 deben estar en la **misma red**. El ESP32 clásico solo usa Wi-Fi de 2,4 GHz. Permita el puerto
   8080 en el firewall de su sistema operativo.
5. Abra el Monitor Serie (115200 en el ESP32; 9600 en el Arduino) para ver el flujo `RX <- {...}` / `OK -> 0,LIMPIA,0`.

## Captura de datos y entrenamiento

Siga `docs/calibracion.md`. En resumen: en el panel elija la muestra (agua limpia, salada, turbia, muy sucia), sumerja
el sensor en ella, deje capturar al menos 30 lecturas, repita en varias sesiones y pulse **Entrenar con las muestras**.
Exporte el CSV desde el panel y guárdelo en `data/` (solo datos reales).

## API del backend

| Método y ruta | Uso |
|---|---|
| `POST /api/lecturas` | El nodo envía `{"t":24.5,"tds":312,"mv":2870}` con cabecera `X-API-KEY`; responde `estado,ETIQUETA,anomalia`. |
| `GET /api/lecturas/ultimas?n=60` | Últimas lecturas (para el panel). |
| `GET /api/captura` · `POST /api/captura?etiqueta=LIMPIA` | Consulta o fija la etiqueta de captura (`NINGUNA` la desactiva). |
| `POST /api/modelo/reentrenar` | Reentrena el clasificador con las muestras etiquetadas. |
| `GET /api/datos.csv` | Exporta las muestras etiquetadas. |

Los endpoints del panel no tienen autenticación: úselos en red local. Si publica el servidor en internet, agregue
autenticación antes.

## Demostración en la feria

Ver `docs/maqueta.md` (maqueta, guion de 3 minutos y plan de contingencia). Recomendación: use un hotspot del celular
o un router propio, porque las redes de eventos suelen aislar los dispositivos entre sí.

## Fase opcional: servidor en la nube

El proyecto funciona completo en red local, que es la opción más segura para la feria. Si desean publicarlo:

- `backend/Dockerfile` empaqueta el backend (no probado). Debe verificar los planes y condiciones vigentes del servicio
  de alojamiento que elija; no los he verificado.
- Use HTTPS. En el ESP32 deberá cambiar `HTTPClient` para usar `WiFiClientSecure` (con validación de certificado en
  producción; `setInsecure()` solo sirve para pruebas y no es seguro).
- Cambie `API_KEY` y la contraseña de la base de datos, y proteja los endpoints del panel.

## Solución de problemas

| Síntoma | Causa probable y qué hacer |
|---|---|
| LCD sin texto | Ajuste el potenciómetro de contraste del módulo I2C; cambie `LCD_DIRECCION` a `0x3F`; revise SDA/SCL. |
| LCD dice "Sin conexion IA" | Revise TX/RX cruzados, GND común, divisor de voltaje, 9600 baudios, y el Monitor Serie del ESP32. |
| ESP32: `Error HTTP: -1` | IP o puerto incorrectos, firewall, o redes distintas. Pruebe abrir `http://IP:8080` desde el celular. |
| ESP32: `Error HTTP: 401` | `API_KEY` del ESP32 distinta a la del backend. |
| ESP32: `Error HTTP: 400` | Valores fuera de rango (sensor desconectado o cableado erróneo). |
| Temperatura `--.-` | DS18B20 sin resistencia pull-up de 4,7 kΩ o mal conectado. |
| TDS en 0 o muy alto | Sensor fuera del agua, mal cableado o sin calibrar. |
| Siempre "Sin entrenar" | Capture al menos 5 muestras de 2 tipos distintos y pulse *Entrenar*. |
| El backend no arranca | Verifique `docker compose ps`, el puerto 5432 y las variables de `.env`. |

## Estado de verificación

- **Probado:** `ClasificadorKnn` y `DetectorAnomalias` (compilación y 9 comprobaciones con JDK 21); el simulador contra un
  servidor HTTP de prueba; sintaxis de `panel.js`.
- **No probado por quien redactó el repositorio:** compilación Maven y arranque de Spring Boot (sin acceso a Maven Central),
  conexión a PostgreSQL, el panel en un navegador, los sketches de Arduino y ESP32 sobre hardware, y la imagen Docker.
  Es probable que la primera ejecución requiera pequeños ajustes; anótelos en este README cuando los resuelva.
- Los valores de calibración, la fórmula de TDS y los pines están tomados de documentación general y deben validarse con sus módulos.

## Créditos y licencias

El circuito de lectura de sensores y la comunicación serie con Java se apoyan en proyectos y tutoriales públicos
(ver `docs/propuesta/` y la sección de referencias del documento entregado). Revise la licencia de cada uno antes de
reutilizar su contenido. **Elijan una licencia para este repositorio** (por ejemplo MIT) y agréguela como `LICENSE`.
Chart.js (incluido en `backend/src/main/resources/static/js/vendor/`) se distribuye bajo licencia MIT.
