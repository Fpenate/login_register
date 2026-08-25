--
-- query
SELECT id, location_code, location_type
	FROM (
	    SELECT id, location_code, location_type,
	           ROW_NUMBER() OVER (PARTITION BY location_type ORDER BY id DESC) as rn
	    FROM conformal_inventory_db.locatios
	) as sub
	WHERE rn = 1



--codigo:
<script>
    
if (document.readyState === 'complete') {
    
        // Array con los datos de ubicaciones pasado desde PHP
    const lastLocationCode = <?= $last_location_json ?>;
    //console.log("lastLocationCode");
    function asignarCodUbicacion(tipoSeleccionado) {
        const campoCodigo = document.getElementById('cod_ubicacion');

        if (!tipoSeleccionado) {
            campoCodigo.value = '';
            return;
        }

        // Buscar en el array el registro cuyo location_type coincida con lo seleccionado
        const ubicacion = lastLocationCode.find(item => item.location_type === tipoSeleccionado);

        if (ubicacion) {
            const codigoActual = ubicacion.location_code; // Ej: "CFB-240"
            const partes = codigoActual.split('-');        // ["CFB", "240"]
            const prefijo = partes[0];                     // "CFB"
            const correlativo = parseInt(partes[1]);       // 240
            const nuevoCorrelativo = correlativo + 1;      // 241

            campoCodigo.value = prefijo + '-' + nuevoCorrelativo; // "CFB-241"
        } else {
            campoCodigo.value = '';
        }
    }
    
    function validarFormulario() {

        if (RecorrerForm("formCapturaDataNewUbication")) {
            document.getElementById("mensaje").style.cssText += "display:none";
            return true;
        } else {
            document.getElementById("mensaje").style.display = "";
            document.getElementsByClassName('is-invalid')[0].focus();
            return false;
        }
    }
    
    function guardarUbicacion() {
        // Primero validar el formulario
        if (!validarFormulario()) {
            return false;
        }

        const form = document.getElementById("formCapturaDataNewUbication");
        const formData = new FormData(form);
        const url = form.getAttribute("action");

        fetch(url, {
            method: "POST",
            body: formData
        })
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                // Notificar al padre (ventana que contiene el modal) y cerrar
                window.parent.mostrarMensajeYCerrarModal('success', data.message);
            } else {
                window.parent.mostrarMensajeYCerrarModal('error', data.message);
            }
        })
        .catch(error => {
            window.parent.mostrarMensajeYCerrarModal('error', 'Error inesperado al guardar.');
        });

        return false; // Evita el submit normal del formulario
    }
}

</script>


------------------------------
-- Table structure for table `locatios`
--

DROP TABLE IF EXISTS `locatios`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `locatios` (
  `id` int NOT NULL AUTO_INCREMENT,
  `location_code` varchar(20) NOT NULL,
  `location_type` enum('BANDEJA','CAJON') NOT NULL,
  `state` enum('AVAILABLE','OCCUPIED') NOT NULL DEFAULT 'AVAILABLE',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=901 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
WITH numbered AS (
    SELECT
        id,
        location_code,
        location_type,
        SUBSTRING_INDEX(location_code, '-', 1) AS prefijo,
        CAST(SUBSTRING_INDEX(location_code, '-', -1) AS UNSIGNED) AS num
    FROM conformal_inventory_db.locatios
    WHERE location_type IN ('BANDEJA','CAJON')
),
prefijos AS (
    -- Obtiene el prefijo (CFB, CFC) usado por cada location_type
    SELECT location_type, MIN(prefijo) AS prefijo
    FROM numbered
    GROUP BY location_type
),
virtual AS (
    -- Fila ficticia con correlativo 000 para forzar el barrido desde el inicio
    SELECT
        0 AS id,
        CONCAT(prefijo, '-000') AS location_code,
        location_type,
        0 AS num
    FROM prefijos
),
combined AS (
    SELECT id, location_code, location_type, num FROM numbered
    UNION ALL
    SELECT id, location_code, location_type, num FROM virtual
),
with_next AS (
    SELECT
        id, location_code, location_type, num,
        LEAD(num) OVER (PARTITION BY location_type ORDER BY num) AS next_num
    FROM combined
),
candidatos AS (
    -- Caso 1: hay un salto en la secuencia -> nos quedamos con el primero que aparece
    SELECT
        id, location_code, location_type, num,
        0 AS prioridad,
        ROW_NUMBER() OVER (PARTITION BY location_type ORDER BY num) AS rn
    FROM with_next
    WHERE next_num IS NOT NULL AND next_num <> num + 1

    UNION ALL

    -- Caso 2 (fallback): no hay saltos -> tomamos el último registro real (ignorando el virtual)
    SELECT
        id, location_code, location_type, num,
        1 AS prioridad,
        ROW_NUMBER() OVER (PARTITION BY location_type ORDER BY num DESC) AS rn
    FROM with_next
    WHERE next_num IS NULL AND id <> 0
)
SELECT id, location_code, location_type
FROM (
    SELECT
        id, location_code, location_type,
        ROW_NUMBER() OVER (PARTITION BY location_type ORDER BY prioridad, num) AS final_rn
    FROM candidatos
    WHERE rn = 1
) t
WHERE final_rn = 1;