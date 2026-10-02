# OrbViSense — Guía de calibración cámara-IMU

Procedimiento para obtener los parámetros de calibración de una cámara monocular y un IMU utilizando **ROS 1 Noetic**, **allan_variance_ros** y **Kalibr**, y posteriormente trasladarlos a la configuración mono-inercial de **ORB-SLAM3**.

---

## 0. Descargar el repositorio

El procedimiento de calibración y el script de conversión se encuentran en el siguiente repositorio:

**GitHub:** https://github.com/HeyItsLuan/OrbVIsense-calibration

Para descargar el repositorio directamente en el Escritorio:

```bash
cd "$HOME/Escritorio"

git clone https://github.com/HeyItsLuan/OrbVIsense-calibration
```
---

## 1. Flujo general de calibración

La calibración se realiza utilizando **dos grabaciones independientes**:

```text
                            ┌──────────────────────┐
                            │   dataset_IMU        │
                            │                      │
┌──────────────────────┐    │Teléfono completamente│
│ dataset_CAMIMU       │    │inmóvil               │
│                      │    └──────────┬───────────┘
│ Teléfono moviéndose  │               │
│ frente al AprilGrid  │               ▼
└──────────┬───────────┘          Allan variance
           │                           │
┌──────────────────────┐               ▼
│    aprilgrid.yaml    │            imu.yaml
│                      │               │
│ Descripción física   │               │
│ del AprilGrid        │               ▼
└──────────┬───────────┘               │
           └────────────┬──────────────┘
                        ▼
                     Kalibr
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
      Calibración cámara    Calibración cámara-IMU
             │                     │
             └──────────┬──────────┘
                        ▼
             Parámetros para ORB-SLAM3
                        │
                        ▼
                    EuRoC.yaml
```

El resultado final de esta guía son los parámetros necesarios para construir el archivo de configuración mono-inercial utilizado por ORB-SLAM3.

---

# 2. Requisitos

## Sistema

* Ubuntu 20.04
* ROS 1 Noetic
* Python 3
* `catkin`
* `catkin_tools`

## Herramientas

* `allan_variance_ros`
* Kalibr
* Git

## Hardware

* Teléfono o cámara con IMU
* AprilGrid

---

# 3. Preparación de los datasets

Los datasets deben tener inicialmente la estructura:

```text
dataset_#######/
├── cam0/
│   ├── data/
│   └── data.csv
└── imu0/
    └── data.csv
```

El archivo:

```text
imu0/data.csv
```

debe contener:

```text
timestamp,wx,wy,wz,ax,ay,az
```

con los timestamps expresados en **nanosegundos**.

Las imágenes de:

```text
cam0/data/
```

deben estar nombradas utilizando su timestamp.

Se necesitan dos datasets.

---

## 3.1. Dataset para Allan variance

Nombre:

```text
$HOME/Escritorio/dataset_IMU/
```

Estructura inicial:

```text
dataset_IMU/
├── cam0/
│   ├── data/
│   └── data.csv
└── imu0/
    └── data.csv
```

Durante esta grabación:

> El teléfono debe permanecer completamente inmóvil durante aproximadamente 30 minutos o más.

Este dataset se utiliza exclusivamente para caracterizar el ruido del IMU mediante **Allan variance**.

La cámara no participa en el cálculo de Allan variance, aunque puede formar parte de la estructura original del dataset.

---

## 3.2. Dataset para calibración cámara-IMU

Nombre:

```text
$HOME/Escritorio/dataset_CAMIMU/
```

Estructura inicial:

```text
dataset_CAMIMU/
├── cam0/
│   ├── data/
│   └── data.csv
└── imu0/
    └── data.csv
```

Durante esta grabación:

> El teléfono se mueve frente al AprilGrid, observándolo desde diferentes posiciones y orientaciones.

Este dataset se utiliza para:

* calibración intrínseca de la cámara;
* calibración conjunta cámara-IMU;
* estimación de la transformación entre cámara e IMU;
* estimación del desplazamiento temporal entre ambos sensores.

---

# 4. Descripción del AprilGrid

El AprilGrid utilizado debe describirse mediante un archivo YAML.

En nuestro caso, el tablero tiene:

```text
4 columnas
5 filas
```

y:

```text
tagSize = 0.0439 m
```

El archivo debe crearse dentro del dataset de calibración:

```bash
gedit "$HOME/Escritorio/dataset_CAMIMU/aprilgrid.yaml"
```

Contenido:

```yaml
target_type: 'aprilgrid'
tagCols: 4
tagRows: 5
tagSize: 0.0439
tagSpacing: 0.1
```

### Significado de los parámetros

| Parámetro     | Significado                             |
| ------------- | --------------------------------------- |
| `target_type` | Tipo de objetivo utilizado por Kalibr   |
| `tagCols`     | Número de tags en X / columnas          |
| `tagRows`     | Número de tags en Y / filas             |
| `tagSize`     | Tamaño del lado del AprilTag, en metros |
| `tagSpacing`  | Separación relativa entre tags          |

`tagSpacing` es una **proporción respecto a `tagSize`**, no una distancia expresada en metros.

Este archivo únicamente describe la geometría física del AprilGrid. No contiene parámetros de la cámara ni del IMU.

---

# 5. Conversión del dataset a ROS bag

El repositorio de calibración contiene el script:

```text
dataset_to_rosbag.py
```

El script recibe como argumento únicamente la carpeta del dataset.

Desde la carpeta donde se encuentra el script:

```bash
cd "$HOME/Escritorio/orbvisense-calibration"
python3 ./dataset_to_rosbag.py \
    "$HOME/Escritorio/dataset_CAMIMU"
```

Esto genera automáticamente:

```text
$HOME/Escritorio/dataset_CAMIMU/camera_imu.bag
```

Para el dataset estático:

```bash
cd "$HOME/Escritorio/orbvisense-calibration"
python3 ./dataset_to_rosbag.py \
    "$HOME/Escritorio/dataset_IMU"
```

Esto genera:

```text
$HOME/Escritorio/dataset_IMU/imu.bag
```

La selección del tipo de bag se realiza automáticamente según el nombre del dataset:

```text
dataset_CAMIMU → camera_imu.bag
dataset_IMU    → imu.bag
```

---

# 6. Allan variance

## 6.1. Instalar `allan_variance_ros`

Entramos al workspace de ROS:

```bash
cd "$HOME/ros1_ws/src"
```

Clonamos el paquete:

```bash
git clone https://github.com/ori-drs/allan_variance_ros.git
```

Compilamos:

```bash
cd "$HOME/ros1_ws"

catkin_make -j1
```

Cargamos el workspace:

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"
```

Comprobamos que ROS pueda localizar el paquete:

```bash
roscd allan_variance_ros
```

Y comprobamos su contenido:

```bash
ls
```

---

# 7. Inspección del IMU bag

El archivo generado se encuentra en:

```text
$HOME/Escritorio/dataset_IMU/imu.bag
```

Ejecutamos:

```bash
rosbag info "$HOME/Escritorio/dataset_IMU/imu.bag"
```

De esta salida necesitamos comprobar:

1. duración de la grabación;
2. cantidad de mensajes;
3. tópico del IMU;
4. frecuencia aproximada del IMU.

El tópico utilizado por el procedimiento es:

```text
/imu0
```

---

## 7.1. Medición de la frecuencia del IMU

Utilizamos tres terminales.

### Terminal 1

```bash
roscore
```

### Terminal 2

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"

rosbag play "$HOME/Escritorio/dataset_IMU/imu.bag" --pause
```

El bag comenzará en pausa.

Presionamos:

```text
Espacio
```

para iniciar la reproducción.

### Terminal 3

```bash
rostopic hz /imu0
```

Dejamos reproducir el bag durante unos segundos hasta que la frecuencia mostrada se estabilice.

La frecuencia medida será utilizada como `imu_rate` y `measure_rate`.

---

# 8. Configuración de Allan variance

Abrimos:

```bash
gedit "$HOME/ros1_ws/src/allan_variance_ros/config/imu_phone.yaml"
```

La estructura debe ser:

```yaml
imu_topic: "/imu0"
imu_rate: <FRECUENCIA_IMU>
measure_rate: <FRECUENCIA_IMU>
sequence_time: <DURACION_EN_SEGUNDOS>
```

Por ejemplo:

```yaml
imu_topic: "/imu0"
imu_rate: 232
measure_rate: 232
sequence_time: 3407
```

Los valores de `imu_rate` y `measure_rate` corresponden a la frecuencia obtenida con:

```bash
rostopic hz /imu0
```

`sequence_time` corresponde a la duración real de la grabación obtenida mediante:

```bash
rosbag info "$HOME/Escritorio/dataset_IMU/imu.bag"
```

---

# 9. Preparación del bag para Allan variance

Ejecutamos:

```bash
python3 "$HOME/ros1_ws/src/allan_variance_ros/scripts/cookbag.py" \
    --input "$HOME/Escritorio/dataset_IMU/imu.bag" \
    --output "$HOME/Escritorio/dataset_IMU/imu_cooked.bag"
```

Comprobamos el resultado:

```bash
rosbag info "$HOME/Escritorio/dataset_IMU/imu_cooked.bag"
```

Debemos comprobar que continúe existiendo:

```text
/imu0
```

y que las muestras del IMU se hayan conservado.

---

# 10. Cálculo de Allan variance

Cargamos ROS:

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"
```

Ejecutamos:

```bash
rosrun allan_variance_ros allan_variance \
    "$HOME/Escritorio/dataset_IMU" \
    "$HOME/ros1_ws/src/allan_variance_ros/config/imu_phone.yaml"
```

El proceso genera:

```text
$HOME/Escritorio/dataset_IMU/allan_variance.csv
```

---

# 11. Generación de `imu.yaml`

Ejecutamos:

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"

rosrun allan_variance_ros analysis.py \
    --data "$HOME/Escritorio/dataset_IMU/allan_variance.csv" \
    --config "$HOME/ros1_ws/src/allan_variance_ros/config/imu_phone.yaml"
```

`analysis.py` genera:

```text
imu.yaml
```

como archivo de salida relativo.

Por ello, el archivo se genera en:

```text
$HOME/ros1_ws/src/allan_variance_ros/imu.yaml
```

Lo copiamos al dataset:

```bash
cp "$HOME/ros1_ws/src/allan_variance_ros/imu.yaml" \
   "$HOME/Escritorio/dataset_IMU/imu.yaml"
```

El resultado queda organizado como:

```text
dataset_IMU/
├── cam0/
├── imu0/
├── imu.bag
├── imu_cooked.bag
├── allan_variance.csv
└── imu.yaml
```

El archivo:

```text
imu.yaml
```

contiene los parámetros de ruido del IMU que posteriormente utilizará Kalibr.

---

# 12. Instalación de Kalibr

Creamos el workspace:

```bash
mkdir -p "$HOME/kalibr_workspace/src"
```

Entramos en `src`:

```bash
cd "$HOME/kalibr_workspace/src"
```

Clonamos Kalibr:

```bash
git clone https://github.com/ethz-asl/kalibr.git
```

Comprobamos:

```bash
ls "$HOME/kalibr_workspace/src/kalibr"
```

---

## 12.1. Dependencias

Instalamos las dependencias del sistema:

```bash
sudo apt-get update

sudo apt-get install -y \
    git \
    wget \
    autoconf \
    automake \
    nano \
    libeigen3-dev \
    libboost-all-dev \
    libsuitesparse-dev \
    doxygen \
    libopencv-dev \
    libpoco-dev \
    libtbb-dev \
    libblas-dev \
    liblapack-dev \
    libv4l-dev
```

Instalamos las dependencias de Python:

```bash
sudo apt-get install -y \
    python3-dev \
    python3-pip \
    python3-scipy \
    python3-matplotlib \
    ipython3 \
    python3-wxgtk4.0 \
    python3-tk \
    python3-igraph \
    python3-pyx
```

---

# 13. Configuración del workspace de Kalibr

Cargamos ROS Noetic:

```bash
source /opt/ros/noetic/setup.bash
```

Entramos al workspace:

```bash
cd "$HOME/kalibr_workspace"
```

Inicializamos `catkin_tools`:

```bash
catkin config --init
```

Indicamos que el workspace se extiende sobre ROS Noetic:

```bash
catkin config --extend /opt/ros/noetic
```

Comprobamos la configuración:

```bash
catkin config
```

Debe aparecer:

```text
Workspace:  $HOME/kalibr_workspace
Source:     $HOME/kalibr_workspace/src
Extending:  /opt/ros/noetic
```

---

# 14. Compilación de Kalibr

Compilamos utilizando un solo núcleo:

```bash
cd "$HOME/kalibr_workspace"

catkin build -j1
```

La compilación puede tardar bastante.

El uso de:

```text
-j1
```

reduce el consumo de recursos y evita sobrecargar el sistema durante la compilación.

Una vez terminada:

```bash
source "$HOME/kalibr_workspace/devel/setup.bash"
```

Los ejecutables principales utilizados en este procedimiento se encuentran en:

```text
$HOME/kalibr_workspace/devel/.private/kalibr/lib/kalibr/
```

---

# 15. Calibración intrínseca de la cámara

La primera calibración realizada por Kalibr es la calibración de la cámara.

Utilizamos:

```bash
"$HOME/kalibr_workspace/devel/.private/kalibr/lib/kalibr/kalibr_calibrate_cameras" \
    --models pinhole-radtan \
    --target "$HOME/Escritorio/dataset_CAMIMU/aprilgrid.yaml" \
    --bag "$HOME/Escritorio/dataset_CAMIMU/camera_imu.bag" \
    --topics /cam0/image_raw
```

El modelo:

```text
pinhole-radtan
```

corresponde a:

```text
Proyección: pinhole
Distorsión: radtan
```

Si la calibración finaliza correctamente, Kalibr genera archivos dentro de:

```text
$HOME/Escritorio/dataset_CAMIMU/
```

Entre ellos:

```text
camera_imu-camchain.yaml
camera_imu-results-cam.txt
camera_imu-report-cam.pdf
```

El archivo principal de esta etapa es:

```text
camera_imu-camchain.yaml
```

---

# 16. Calibración conjunta cámara-IMU

En esta etapa se utiliza:

* el bag de cámara + IMU;
* la calibración de cámara obtenida anteriormente;
* el `imu.yaml` generado mediante Allan variance;
* el `aprilgrid.yaml`.

Ejecutamos:

```bash
"$HOME/kalibr_workspace/devel/.private/kalibr/lib/kalibr/kalibr_calibrate_imu_camera" \
    --bag "$HOME/Escritorio/dataset_CAMIMU/camera_imu.bag" \
    --cam "$HOME/Escritorio/dataset_CAMIMU/camera_imu-camchain.yaml" \
    --imu "$HOME/Escritorio/dataset_IMU/imu.yaml" \
    --target "$HOME/Escritorio/dataset_CAMIMU/aprilgrid.yaml"
```

El resultado queda dentro de:

```text
$HOME/Escritorio/dataset_CAMIMU/
```

Los archivos principales son:

```text
camera_imu-camchain-imucam.yaml
camera_imu-imu.yaml
camera_imu-results-imucam.txt
camera_imu-report-imucam.pdf
```

---

# 17. Interpretación de los resultados de Kalibr

## 17.1. `camera_imu-camchain.yaml`

Contiene la calibración intrínseca de la cámara.

Incluye parámetros como:

```text
camera_model
intrinsics
distortion_coeffs
distortion_model
resolution
rostopic
```

---

## 17.2. `camera_imu-camchain-imucam.yaml`

Contiene el resultado de la calibración conjunta cámara-IMU.

Además de los parámetros de cámara, contiene información de la transformación cámara-IMU y del desplazamiento temporal.

---

## 17.3. `camera_imu-results-imucam.txt`

Es el reporte detallado de la calibración conjunta.

Entre sus resultados aparecen:

```text
T_ci: (imu0 to cam0)
T_ic: (cam0 to imu0)
```

y:

```text
timeshift cam0 to imu0
```

Este archivo se utiliza para identificar la transformación que debe trasladarse a la configuración de ORB-SLAM3.

---

## 17.4. `camera_imu-imu.yaml`

Contiene los parámetros del IMU utilizados durante la calibración de Kalibr.

Los parámetros de ruido utilizados originalmente provienen del `imu.yaml` generado mediante Allan variance.

---

# 18. Conversión de los resultados a `EuRoC.yaml`

Los resultados de Kalibr y Allan no utilizan los mismos nombres de variables que ORB-SLAM3.

Por ello, los parámetros deben trasladarse al formato utilizado por ORB-SLAM3.

La siguiente tabla sirve como referencia para realizar esta conversión.

| Variable en `EuRoC.yaml` | Archivo de origen                 | Variable de origen              |
| ------------------------ | --------------------------------- | ------------------------------- |
| `Camera.type`            | `camera_imu-camchain-imucam.yaml` | `camera_model`                  |
| `Camera.width`           | `camera_imu-camchain-imucam.yaml` | `resolution[0]`                 |
| `Camera.height`          | `camera_imu-camchain-imucam.yaml` | `resolution[1]`                 |
| `Camera.fx`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[0]`                 |
| `Camera.fy`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[1]`                 |
| `Camera.cx`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[2]`                 |
| `Camera.cy`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[3]`                 |
| `Camera.k1`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[0]`          |
| `Camera.k2`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[1]`          |
| `Camera.p1`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[2]`          |
| `Camera.p2`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[3]`          |
| `Camera.fps`             | `camera_imu.bag`                  | frecuencia de `/cam0/image_raw` |
| `Camera.RGB`             | configuración de la cámara        | formato de imagen               |
| `IMU.Frequency`          | `dataset_IMU/imu.yaml`            | `update_rate`                   |
| `IMU.NoiseGyro`          | `dataset_IMU/imu.yaml`            | `gyroscope_noise_density`       |
| `IMU.NoiseAcc`           | `dataset_IMU/imu.yaml`            | `accelerometer_noise_density`   |
| `IMU.GyroWalk`           | `dataset_IMU/imu.yaml`            | `gyroscope_random_walk`         |
| `IMU.AccWalk`            | `dataset_IMU/imu.yaml`            | `accelerometer_random_walk`     |
| `IMU.T_b_c1`             | `camera_imu-results-imucam.txt`   | `T_ic` — `cam0 to imu0`         |

---

# 19. Interpretación de `IMU.T_b_c1`

En el fork de ORB-SLAM3 utilizado por OrbViSense, `IMU.T_b_c1` se carga como `Tbc`.

El código de ORB-SLAM3 mantiene internamente:

```text
mTbc
```

y calcula:

```text
mTcb = mTbc.inverse()
```

Por ello, para este flujo de calibración se utiliza como entrada de:

```text
IMU.T_b_c1
```

la transformación:

```text
T_ic
```

obtenida de Kalibr:

```text
T_ic: (cam0 to imu0)
```

No se debe volver a invertir esta matriz antes de colocarla en `IMU.T_b_c1`.

---

# 20. Desplazamiento temporal cámara-IMU

Kalibr también proporciona:

```text
timeshift cam0 to imu0
```

con la convención:

```text
t_imu = t_cam + shift
```

Este parámetro debe conservarse como parte de los resultados de calibración.

No se debe inventar una variable adicional en `EuRoC.yaml` si la versión de ORB-SLAM3 utilizada no proporciona un parámetro específico para introducir este desplazamiento temporal.

---

# 21. Estructura final de los datasets

Después de completar la calibración, `dataset_IMU` contiene:

```text
dataset_IMU/
├── cam0/
├── imu0/
├── imu.bag
├── imu_cooked.bag
├── allan_variance.csv
└── imu.yaml
```

Y `dataset_CAMIMU` contiene:

```text
dataset_CAMIMU/
├── cam0/
├── imu0/
├── camera_imu.bag
├── aprilgrid.yaml
├── camera_imu-camchain.yaml
├── camera_imu-camchain-imucam.yaml
├── camera_imu-imu.yaml
├── camera_imu-report-cam.pdf
├── camera_imu-report-imucam.pdf
├── camera_imu-results-cam.txt
└── camera_imu-results-imucam.txt
```

---

# 22. Resultado final

Al terminar el procedimiento se dispone de:

```text
Allan variance
      │
      └── imu.yaml
             │
             ▼
        Parámetros IMU
             
Kalibr cámara
      │
      └── camera_imu-camchain.yaml
             │
             ▼
        Parámetros cámara

Kalibr cámara-IMU
      │
      ├── T_ic
      ├── time shift
      └── resultados conjuntos
             │
             ▼
        Parámetros cámara-IMU
```

Estos resultados pueden utilizarse para construir el archivo de configuración **mono-inercial de ORB-SLAM3 (`EuRoC.yaml`)** utilizado posteriormente por OrbViSense.

El archivo `aprilgrid.yaml` se utiliza durante la calibración de Kalibr y **no forma parte del `EuRoC.yaml`**.

Los parámetros de persistencia del Atlas, como:

```yaml
#System.SaveAtlasToFile: "FileName"
#System.LoadAtlasFromFile: "FileName"
```

pertenecen a la configuración y operación de ORB-SLAM3/OrbViSense, no al procedimiento de calibración, por lo que deben documentarse en la documentación de OrbViSense y no como parte de esta guía de calibración.

