# GLM-5.3 Flash EXL3: preparación para cuatro Sparks sin switch

Estado: adaptación experimental; no desplegada ni medida en los Sparks.
Modelo **Flash**, no GLM-5.3 completo. Base Mia `8f29c6dd42da945884d087f1b98ffd4850bb7bd8`.
Rama: `codex/tp4-switchless-ring`. No modifica los launchers TP2/TP3.

## Qué copiar ahora

En un Spark (Linux ARM64), descargar la imagen oficial que usa Mia:

```bash
IMAGE=ghcr.io/miaai-lab/glm-5.3-flash-2x-dgx-sparks:exl3-instanttensor
PIN=ghcr.io/miaai-lab/glm-5.3-flash-2x-dgx-sparks@sha256:447114ee77d14c9b4732ee23978ada2a0ee9027868a231d6fd42700a8b25be1d
docker pull --platform linux/arm64 "$PIN"
docker tag "$PIN" "$IMAGE"
docker image inspect --format '{{.Id}} {{json .RepoDigests}}' "$IMAGE"
docker save -o glm53-flash-instanttensor.tar "$IMAGE"
sha256sum glm53-flash-instanttensor.tar
```

Copiar ese mismo archivo a los cuatro Sparks y ejecutar en cada uno:

```bash
docker load -i glm53-flash-instanttensor.tar
docker image inspect --format '{{.Id}} {{.Architecture}}' \
  ghcr.io/miaai-lab/glm-5.3-flash-2x-dgx-sparks:exl3-instanttensor
```

Esto NO para DeepSeek. Comparar el SHA256 del archivo después de copiarlo.
El tag es mutable: conservar el digest impreso; tras cargar, no volver a hacer
pull individualmente. El preflight exige el mismo ID de imagen en los cuatro.
La imagen no contiene los pesos ni convierte NCCL automáticamente a switchless.
Manifest y configuración consultados el 2026-09-16: linux/arm64, capas comprimidas
9.80 GB (no es el tamaño descomprimido ni el espacio total necesario).

También copiar a cada nodo **la misma biblioteca NCCL parcheada y de confianza
que ya usa vuestro DeepSeek del PR #19**, a:
`/home/incoxe/nccl-2.30.7/libnccl.so.2.30.7` (ajustar si la cuenta difiere).
No copiar a ciegas el NCCL stock del contenedor. Conservar la biblioteca original
y su SHA256; no se sobrescribe ningún NCCL del sistema ni de la imagen.

```bash
sha256sum "$HOME/nccl-2.30.7/libnccl.so.2.30.7"
grep -qa SWITCHLESS_RING_ONLY "$HOME/nccl-2.30.7/libnccl.so.2.30.7"
```

El marcador y la igualdad de hashes comprueban consistencia, no autenticidad,
ABI efectiva ni funcionamiento de collectives. Si no disponemos de esa biblioteca,
queda pendiente compilarla; no hay una imagen propia switchless publicada.
Procedencia documentada por PR #19: NCCL commit
`73cf112295c33aee2b895f329f592f2a9b4b0f97`; parche sparkring blob
`f4853e84334eaa3f980dce69a12660d8f1774d7c`, archivo
`spark_transport/nccl/nccl-2.30.7-dual-pci-domain.patch`.

Pesos (independientes de Docker): `Mia-AiLab/GLM-5.3-Flash-EXL3-TR3-4bpw`,
revisión `25a44fdbf16862a46b7cc9921142c6c81350af2f`.
El launcher mantiene la descarga/sincronización upstream; no asume que los pesos
de DeepSeek sean reutilizables. Reservar disco para imagen, tar y cachés/pesos.

## Configurar cuando conozcamos las IP reales

Copiar el repositorio al nodo principal. Conservar cualquier `.env` existente;
en una copia nueva:

```bash
cp -n .env.example .env
cp -n .env.tp4.ring.example .env.tp4
# Editar .env.tp4: cuatro IP de administración, interfaces, HCAs y GID por nodo.
bash start-tp4.sh doctor-ring
```

Requiere SSH por clave desde el principal a los otros tres nodos y acceso Docker.
No guardar contraseñas en los archivos. Los sockets de bootstrap usan la interfaz
de administración alcanzable por los cuatro; el tráfico RDMA usa ambas CX7.
Reutilizar el anillo ya operativo de DeepSeek: un subnet por cable y la misma
longitud de prefijo que su configuración (ejemplo /24). No cambiar Netplan,
direcciones, MTU, rutas ni cableado remotamente sin vía de administración estable.

`doctor-ring` verifica ambos HCAs activos, IPv4 RoCE v2 en ambos puertos,
marcador NCCL, SHA256 idéntico de la biblioteca, imagen idéntica y ruta pip NCCL
real dentro del contenedor en los cuatro nodos. Usa contenedores CPU efímeros
sin red, sin GPU y con raíz de solo lectura. No prueba conectividad RDMA extremo
a extremo. No para el modelo existente. El launcher puede crear los archivos
`.env` por defecto si no existen.

## Lanzar después, durante mantenimiento

No se ejecuta automáticamente. Primero realizar un all-reduce de cuatro rangos
con el mismo NCCL y verificar `NET/IB` y el ciclo físico en logs. Después detener
DeepSeek mediante su propio launcher, conservando sus configuraciones y pesos.

```bash
bash start-tp4.sh start
bash start-tp4.sh status
bash start-tp4.sh logs
# Parada explícita solo de los contenedores GLM:
bash start-tp4.sh stop
```

El arranque exige GPU libre en todos los nodos; falla antes de reemplazar
contenedores ante cualquier error de preflight. `restart` se rechaza en modo
anillo: usar doctor, parada explícita y arranque. No es un despliegue transaccional:
un fallo después de las comprobaciones puede dejar un arranque parcial.
Volver a DeepSeek: parar GLM y arrancar con su receta anterior, sin desinstalar nada.

NCCL se monta sobre la biblioteca pip descubierta en la imagen, sin `LD_PRELOAD`.
Se activan ring-only, subnet-aware routing, skip-tree y cuatro canales. Esto
porta la capa de transporte del PR #19, no sus optimizaciones SGLang a vLLM.

## Ajustar con medidas, no con promesas

Inicio: 128k contexto, cuatro secuencias, memoria 0.75, lote 2048, draft apagado.
Es un perfil de puesta en marcha, no el máximo de contexto. Ensayar después:

1. Generación corta, tool calls y una tarea de programación de Hermes.
2. C1/C2/C4/C8 con prompts fríos de 8k/32k/128k: TTFT p50/p95, tok/s por
   sesión, agregado, errores y memoria libre por nodo. Separar caché fría/caliente.
3. DFlash2 k=3 frente a `SPEC_METHOD=none`; después canales 4 frente a 8.
4. Subir a 256k/512k/1M solo tras estabilidad; contexto configurado no equivale
   a capacidad simultánea para todas las sesiones. Medir presión del KV y colas.
5. Soak prolongado con prefill largo y decode concurrente antes de uso habitual.

Mia documenta bloqueos históricos en un kit TP4 con contexto largo; no atribuirlos
automáticamente al anillo ni considerar que este parche los resuelve. Revisar
los comentarios de `.env.tp4.example` y sus referencias antes de activar máximos.
No hay todavía cifras reproducidas de velocidad, TTFT o concurrencia de este fork.

## Créditos

Validación local: 14 tests de contrato CPU/Bash, incluidos GID del segundo puerto,
rechazo de biblioteca/imagen ausente, GPU ocupada, diferencias entre rangos y
sintaxis de launcher/helper/config. No equivalen a una prueba GPU/RDMA.

```bash
python3 -m unittest discover -s tests -p test_switchless_ring.py -v
```

- [Mia: receta GLM](https://github.com/MiaAI-Lab/GLM-5.3-Flash-EXL3-2x-DGX-Sparks).
- [Carlos: DeepSeek PR #19](https://github.com/MiaAI-Lab/DeepSeek-v4.1-Flash-DGX-Sparks/pull/19), referencia `8d2cdf9`.
- [Saolence: PR #3](https://github.com/MiaAI-Lab/DeepSeek-v4.1-Flash-DGX-Sparks/pull/3), integración original del anillo.
- [FujitsuPolycom/sparkring](https://github.com/FujitsuPolycom/sparkring), parche NCCL.

Se conserva la licencia upstream; no se redistribuye aquí el binario NCCL.
