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


WITH parsed AS (
    SELECT
        id,
        location_code,
        location_type,
        state,
        SUBSTRING_INDEX(location_code, '-', 1) AS prefix,
        CAST(SUBSTRING_INDEX(location_code, '-', -1) AS UNSIGNED) AS num
    FROM locatios
),
gaps AS (
    SELECT
        prefix,
        num,
        location_code,
        LAG(num) OVER (PARTITION BY prefix ORDER BY num) AS prev_num,
        LAG(location_code) OVER (PARTITION BY prefix ORDER BY num) AS prev_code
    FROM parsed
),
first_gap AS (
    SELECT
        prefix,
        prev_code AS ultimo_correlativo_antes_del_hueco,
        num AS siguiente_encontrado,
        ROW_NUMBER() OVER (PARTITION BY prefix ORDER BY num) AS rn
    FROM gaps
    WHERE num - prev_num > 1
)
SELECT prefix, ultimo_correlativo_antes_del_hueco, siguiente_encontrado
FROM first_gap
WHERE rn = 1
ORDER BY prefix;
