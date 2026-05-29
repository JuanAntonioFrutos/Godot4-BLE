# Godot4-BLE

Plugin para conectar Godot v4 utilizando BLE.

---

# Pasos a seguir

# FASE 1: Creación del Proyecto en Android Studio

1. Abre Android Studio y selecciona **New Project**.

2. En la lista de plantillas, selecciona **No Activity** (o **Empty Views Activity**) y pulsa **Next**.

3. Configuración general:

* **Name:** `GodotBluetoothProject`
* **Language:** `Java`
* **Minimum SDK:** `API 24 (Android 7.0)`

Pulsa **Finish**.

4. Crear el módulo del Plugin:

* Ve al menú superior:

```text
File -> New -> New Module...
```

* Selecciona **Android Library**

* Configura:

```text
Module Name: GodotBluetooth
Package Name: com.ksk.godotbluetooth
```

> ⚠️ ¡Muy importante mantener exactamente ese package name!

* Pulsa **Finish**

---

# FASE 2: Conectar Android Studio con Godot (Gradle)

Necesitamos decirle al proyecto que vamos a usar las herramientas de Godot.

1. En la barra izquierda (vista Android), despliega la carpeta:

```text
GodotBluetooth
```

2. Abre el archivo:

```text
build.gradle.kts
```

(El que está dentro del módulo `GodotBluetooth`).

3. Ve al bloque:

```kotlin
dependencies {
}
```

y añade esta línea:

```kotlin
compileOnly("org.godotengine:godot:4.2.1.stable")
```

> ⚠️ Asegúrate de que la versión coincide aproximadamente con tu versión de Godot.

4. Pulsa el botón:

```text
Sync Now
```

(icono del elefante arriba a la derecha).

---

# FASE 3: El Código Java (El Motor Bluetooth)

Aquí configuramos la lectura de datos BLE "en el aire" (*Raw Hex Data*), ideal para sensores que emiten datos sin necesidad de conexión.

## Crear la clase Java

Ruta:

```text
GodotBluetooth -> src -> main -> java -> com.ksk.godotbluetooth
```

1. Haz clic derecho sobre la carpeta:

```text
com.ksk.godotbluetooth
```

2. Selecciona:

```text
New -> Java Class
```

3. Nombre:

```text
GodotBluetooth
```

4. Pega este código:

```java
package com.ksk.godotbluetooth;

import android.annotation.SuppressLint;
import android.bluetooth.BluetoothAdapter;
import android.bluetooth.BluetoothManager;
import android.bluetooth.le.BluetoothLeScanner;
import android.bluetooth.le.ScanCallback;
import android.bluetooth.le.ScanResult;
import android.bluetooth.le.ScanSettings;
import android.content.Context;
import android.util.Log;
import androidx.annotation.NonNull;

import org.godotengine.godot.Godot;
import org.godotengine.godot.plugin.GodotPlugin;
import org.godotengine.godot.plugin.SignalInfo;

import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.util.Set;

public class GodotBluetooth extends GodotPlugin {
    private static final String TAG = "GodotBluetooth";
    private BluetoothAdapter bluetoothAdapter;
    private BluetoothLeScanner bluetoothLeScanner;
    private boolean scanning = false;

    public GodotBluetooth(Godot godot) {
        super(godot);
    }

    @NonNull
    @Override
    public String getPluginName() {
        return "GodotBluetooth";
    }

    @NonNull
    @Override
    public List<String> getPluginMethods() {
        return Arrays.asList("scan", "stopScan");
    }

    @NonNull
    @Override
    public Set<SignalInfo> getPluginSignals() {
        return Collections.singleton(new SignalInfo("bluetooth_log", String.class));
    }

    private boolean initBluetooth() {
        BluetoothManager bluetoothManager = (BluetoothManager) getActivity().getSystemService(Context.BLUETOOTH_SERVICE);
        if (bluetoothManager != null) {
            bluetoothAdapter = bluetoothManager.getAdapter();
            if (bluetoothAdapter != null) {
                bluetoothLeScanner = bluetoothAdapter.getBluetoothLeScanner();
                return true;
            }
        }
        emitSignal("bluetooth_log", "Error: No se pudo inicializar Bluetooth");
        return false;
    }

    @SuppressLint("MissingPermission")
    public void scan() {
        if (!initBluetooth()) return;
        if (scanning) {
            stopScan();
        }

        emitSignal("bluetooth_log", "Escanear iniciado (Modo: Broadcast)...");

        ScanSettings settings = new ScanSettings.Builder()
                .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
                .build();

        scanning = true;
        bluetoothLeScanner.startScan(null, settings, leScanCallback);
    }

    @SuppressLint("MissingPermission")
    public void stopScan() {
        if (scanning && bluetoothLeScanner != null) {
            bluetoothLeScanner.stopScan(leScanCallback);
            scanning = false;
            emitSignal("bluetooth_log", "Escaneo finalizado.");
        }
    }

    // 🟢 ESTA ES LA NUEVA FUNCIÓN TOTALMENTE ABIERTA Y REEMPLAZADA
    private final ScanCallback leScanCallback = new ScanCallback() {
        @Override
        public void onScanResult(int callbackType, ScanResult result) {
            super.onScanResult(callbackType, result);
            if (result.getDevice() == null) return;

            @SuppressLint("MissingPermission")
            String address = result.getDevice().getAddress();

            // Si el dispositivo no trae datos extras (longitud cero), mandamos cadena vacía en vez de ignorarlo
            String hexData = "";
            if (result.getScanRecord() != null && result.getScanRecord().getBytes() != null) {
                hexData = bytesToHex(result.getScanRecord().getBytes());
            }

            int rssi = result.getRssi();

            // Enviamos SIEMPRE los datos limpios a Godot para que pinte el sensor
            String msg = "SCAN_DATA|" + address + "|" + hexData + "|" + rssi;
            emitSignal("bluetooth_log", msg);
        }

        @Override
        public void onScanFailed(int errorCode) {
            // Si hay un error de hardware en Android, nos enteraremos en la pantalla
            emitSignal("bluetooth_log", "ERROR_JAVA_SCAN|" + errorCode);
        }
    };

    private String bytesToHex(byte[] bytes) {
        if (bytes == null) return "";
        StringBuilder sb = new StringBuilder();
        for (byte b : bytes) {
            sb.append(String.format("%02X", b));
        }
        return sb.toString();
    }
}

```

---

# FASE 4: Compilar el Plugin (.aar)

1. Abre la pestaña **Terminal** en Android Studio.

2. Ejecuta:

```bash
./gradlew :GodotBluetooth:assemble
```

3. Si todo va bien aparecerá:

```text
BUILD SUCCESSFUL
```

4. El archivo generado estará en:

```text
/ GodotBluetoothProject / GodotBluetooth / build / outputs / aar /
```

Archivo esperado:

```text
GodotBluetooth-release.aar
```

---

# FASE 5: Inyectar el Plugin en Godot

## Instalar plantilla Android

En Godot:

```text
Proyecto -> Instalar plantilla de compilación de Android...
```

Esto creará:

```text
android/build/
```

---

## Copiar archivos

Copia:

```text
GodotBluetooth-release.aar
```

hacia:

```text
android/plugins/
```

---

## Crear archivo `.gdap`

Dentro de:

```text
android/plugins/
```

crea:

```text
GodotBluetooth.gdap
```

Contenido:

```toml
[config]
name="GodotBluetooth"
binary_type="local"
binary="GodotBluetooth-release.aar"

[dependencies]
local=[]
remote=[]
```

---

## Forzar el AndroidManifest

Abre:

```text
android/build/AndroidManifest.xml
```

y añade esta línea justo antes de:

```xml
</application>
```

```xml
<meta-data
    android:name="org.godotengine.plugin.v1.GodotBluetooth"
    android:value="com.ksk.godotbluetooth.GodotBluetooth" />
```

---

# FASE 6: Configurar Permisos en Godot y Exportar

1. Ve a:

```text
Proyecto -> Exportar... -> Android
```

2. Marca:

```text
Use Gradle Build
```

3. En la lista de Plugins activa:

```text
GodotBluetooth
```

4. En Permissions marca:

* Bluetooth
* Bluetooth Admin
* Access Fine Location
* Access Coarse Location

> ⚠️ Android requiere permisos de ubicación para utilizar Bluetooth BLE.

5. Exporta o instala directamente en el móvil.

---

# Resultado Final

Con estos pasos tendrás un plugin BLE completamente funcional para Godot 4 utilizando Android Studio y Java.

El plugin:

* Escanea dispositivos BLE
* Lee broadcasts BLE
* Obtiene:

  * MAC Address
  * RSSI
  * Datos HEX RAW
* Envía toda la información a Godot mediante señales

Ideal para:

* Sensores BLE
* Wearables
* ESP32
* Beacons
* IoT
* Hardware personalizado

---

# Señal Emitida a Godot

Formato:

```text
SCAN_DATA|MAC_ADDRESS|HEX_DATA|RSSI
```

Ejemplo:

```text
SCAN_DATA|AA:BB:CC:DD:EE:FF|0201061AFF4C000215...|-65
```

---

# Notas

* Compatible con Godot 4.x
* Android SDK mínimo: API 24
* Lenguaje: Java
* Arquitectura: Android Library (.aar)
* Comunicación mediante señales Godot

---
