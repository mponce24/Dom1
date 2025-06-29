<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Calculadora de Igualdades y Desigualdades</title>
    <!-- Incluye Tailwind CSS para estilos modernos y responsivos -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f0f4f8; /* Un fondo suave */
            display: flex;
            justify-content: center;
            align-items: flex-start; /* Alinea al inicio para dejar espacio en la parte inferior */
            min-height: 100vh;
            padding: 20px;
            box-sizing: border-box;
        }
        .calculator-container {
            @apply bg-white p-8 rounded-xl shadow-lg w-full max-w-2xl transform transition-all duration-300 ease-in-out;
            border: 1px solid #e2e8f0; /* Un borde sutil */
        }
        h1 {
            @apply text-3xl font-bold text-center mb-6 text-gray-800;
        }
        .input-group {
            @apply mb-4;
        }
        label {
            @apply block text-gray-700 text-sm font-semibold mb-2;
        }
        input[type="text"] {
            @apply w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-gray-800;
        }
        .button-group {
            @apply flex flex-col sm:flex-row gap-4 mt-6;
        }
        button {
            @apply flex-1 py-3 px-6 rounded-lg text-white font-semibold transition-all duration-300 ease-in-out shadow-md;
        }
        button:hover {
            @apply shadow-lg transform -translate-y-0.5;
        }
        .btn-primary {
            @apply bg-blue-600 hover:bg-blue-700;
        }
        .btn-secondary {
            @apply bg-green-600 hover:bg-green-700;
        }
        .result-area {
            @apply mt-6 p-4 bg-blue-50 rounded-lg border border-blue-200 text-blue-800 text-center font-medium;
            min-height: 60px;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .graph-area {
            @apply mt-6 bg-white border border-gray-300 rounded-lg overflow-hidden;
        }
        canvas {
            @apply w-full; /* Hace que el lienzo sea responsivo */
            background-color: #fdfdfd; /* Fondo del lienzo */
        }
        /* Estilos para el mensaje de información */
        .info-message {
            @apply mt-4 p-3 bg-yellow-50 rounded-lg border border-yellow-200 text-yellow-800 text-sm;
        }
    </style>
</head>
<body>
    <div class="calculator-container">
        <h1>Calculadora de Igualdades y Desigualdades</h1>

        <div class="info-message">
            <p><strong>Uso:</strong> Introduce ecuaciones o desigualdades lineales simples.</p>
            <p><strong>Ejemplos:</strong> <code>2x + 5 = 15</code>, <code>3x - 1 > 8</code>, <code>-4x <= 20</code>.</p>
            <p>Solo se admiten ecuaciones y desigualdades con una sola variable 'x'.</p>
        </div>

        <div class="input-group">
            <label for="equationInput">Introduce tu ecuación o desigualdad:</label>
            <input type="text" id="equationInput" placeholder="Ej: 2x + 3 = 7 o x > 5">
        </div>

        <div class="button-group">
            <button class="btn-primary" onclick="calculateAndGraph()">Calcular y Graficar</button>
            <button class="btn-secondary" onclick="speakResult()">Leer Resultado</button>
        </div>

        <div class="result-area" id="resultDisplay">
            El resultado aparecerá aquí.
        </div>

        <div class="graph-area">
            <canvas id="graphCanvas"></canvas>
        </div>
    </div>

    <script>
        // Obtiene referencias a elementos del DOM
        const equationInput = document.getElementById('equationInput');
        const resultDisplay = document.getElementById('resultDisplay');
        const graphCanvas = document.getElementById('graphCanvas');
        const ctx = graphCanvas.getContext('2d');

        // Configura el tamaño del lienzo de forma responsiva
        function resizeCanvas() {
            graphCanvas.width = graphCanvas.offsetWidth;
            graphCanvas.height = Math.min(graphCanvas.offsetWidth * 0.6, 400); // Mantiene una proporción 16:9 o un máximo de 400px de alto
            drawGraph(); // Redibuja el gráfico al cambiar el tamaño
        }

        window.addEventListener('resize', resizeCanvas);
        window.addEventListener('load', resizeCanvas); // Llama al cargar la página

        let solution = ''; // Almacena el resultado de la solución
        let graphType = ''; // 'equality' o 'inequality'
        let graphValue = null; // Valor para graficar (ej. x=5)
        let inequalityDirection = null; // '>' o '<' para desigualdades

        /**
         * Parsea y resuelve una ecuación o desigualdad lineal simple.
         * Formatos esperados: ax + b = c, ax + b > c, etc.
         * @param {string} input La cadena de la ecuación o desigualdad.
         * @returns {object|null} Un objeto con el tipo de operación, el resultado de x, y la dirección de la desigualdad si aplica.
         */
        function parseAndSolve(input) {
            // Elimina espacios en blanco para facilitar el parsing
            const cleanedInput = input.replace(/\s+/g, '');
            let match;
            let type;
            let leftSide, rightSide;

            // Define expresiones regulares para igualdades y desigualdades
            const equalityRegex = /(.+)=(.+)/;
            const inequalityRegex = /(.+)([<>]|<=|>=)(.+)/;

            if (match = cleanedInput.match(equalityRegex)) {
                type = 'equality';
                leftSide = match[1];
                rightSide = match[2];
            } else if (match = cleanedInput.match(inequalityRegex)) {
                type = 'inequality';
                leftSide = match[1];
                inequalityDirection = match[2];
                rightSide = match[3];
            } else {
                return null; // Formato no reconocido
            }

            // Función auxiliar para obtener el coeficiente y el término constante
            const parseSide = (side) => {
                let coefX = 0;
                let constant = 0;

                // Divide la cadena en términos (ej. "2x", "+5", "-3")
                const terms = side.match(/(\+|-)?\d*x|(\+|-)?\d+/g) || [];

                terms.forEach(term => {
                    if (term.includes('x')) {
                        const num = parseFloat(term.replace('x', ''));
                        coefX += isNaN(num) ? (term === 'x' || term === '+x' ? 1 : -1) : num;
                    } else {
                        constant += parseFloat(term);
                    }
                });
                return { coefX, constant };
            };

            const leftParsed = parseSide(leftSide);
            const rightParsed = parseSide(rightSide);

            // Reorganiza la ecuación a la forma Ax = B
            let A = leftParsed.coefX - rightParsed.coefX;
            let B = rightParsed.constant - leftParsed.constant;

            if (A === 0) {
                // Si el coeficiente de x es 0, la ecuación puede no tener solución o ser una identidad
                if (B === 0) {
                    return { type: 'identity', message: 'La expresión es una identidad (siempre verdadera).' };
                } else {
                    return { type: 'contradiction', message: 'La expresión es una contradicción (siempre falsa).' };
                }
            }

            let x = B / A;

            // Ajusta la dirección de la desigualdad si se multiplicó/dividió por un número negativo
            if (type === 'inequality' && A < 0) {
                if (inequalityDirection === '>') inequalityDirection = '<';
                else if (inequalityDirection === '<') inequalityDirection = '>';
                else if (inequalityDirection === '>=') inequalityDirection = '<=';
                else if (inequalityDirection === '<=') inequalityDirection = '>=';
            }

            return { type, value: x, direction: inequalityDirection };
        }

        /**
         * Dibuja el gráfico en el lienzo según el resultado.
         */
        function drawGraph() {
            ctx.clearRect(0, 0, graphCanvas.width, graphCanvas.height); // Limpia el lienzo

            // Configuración del eje y la escala
            const centerX = graphCanvas.width / 2;
            const centerY = graphCanvas.height / 2;
            const scale = 30; // Píxeles por unidad

            ctx.save(); // Guarda el estado actual del contexto
            ctx.translate(centerX, centerY); // Mueve el origen al centro del lienzo
            ctx.scale(1, -1); // Invierte el eje Y para que los valores positivos vayan hacia arriba

            // Dibujar el eje X
            ctx.beginPath();
            ctx.moveTo(-centerX, 0);
            ctx.lineTo(graphCanvas.width - centerX, 0);
            ctx.strokeStyle = '#333';
            ctx.lineWidth = 2;
            ctx.stroke();

            // Dibujar el eje Y
            ctx.beginPath();
            ctx.moveTo(0, -centerY);
            ctx.lineTo(0, graphCanvas.height - centerY);
            ctx.stroke();

            // Etiquetas de los ejes
            ctx.scale(1, -1); // Volver a invertir para que el texto no esté al revés
            ctx.fillStyle = '#555';
            ctx.font = '12px Inter';
            ctx.fillText('X', graphCanvas.width / 2 - 15, -10);
            ctx.fillText('Y', 5, -graphCanvas.height / 2 + 15);
            ctx.scale(1, -1); // Volver a invertir para el dibujo

            // Marcas en el eje X
            for (let i = -Math.floor(centerX / scale); i * scale <= centerX; i++) {
                if (i === 0) continue; // No dibujar en el origen
                ctx.beginPath();
                ctx.moveTo(i * scale, -5);
                ctx.lineTo(i * scale, 5);
                ctx.stroke();
                ctx.scale(1, -1); // Invertir para el texto
                ctx.fillText(i.toString(), i * scale - 7, 18);
                ctx.scale(1, -1); // Volver a invertir
            }

            // Dibujar la gráfica si hay una solución
            if (graphValue !== null) {
                const xPixel = graphValue * scale;

                if (graphType === 'equality') {
                    // Línea vertical para x = valor
                    ctx.strokeStyle = '#dc2626'; // Rojo
                    ctx.lineWidth = 3;
                    ctx.setLineDash([]); // Línea sólida
                    ctx.beginPath();
                    ctx.moveTo(xPixel, -centerY);
                    ctx.lineTo(xPixel, centerY);
                    ctx.stroke();

                    // Etiqueta de la línea
                    ctx.scale(1, -1);
                    ctx.fillStyle = '#dc2626';
                    ctx.fillText(`x = ${graphValue.toFixed(2)}`, xPixel + 10, -graphCanvas.height / 2 + 30);
                    ctx.scale(1, -1);

                } else if (graphType === 'inequality') {
                    // Línea punteada para desigualdades estrictas, sólida para no estrictas
                    ctx.strokeStyle = '#1e40af'; // Azul oscuro
                    ctx.lineWidth = 3;
                    if (inequalityDirection === '>' || inequalityDirection === '<') {
                        ctx.setLineDash([5, 5]); // Línea punteada
                    } else {
                        ctx.setLineDash([]); // Línea sólida
                    }
                    ctx.beginPath();
                    ctx.moveTo(xPixel, -centerY);
                    ctx.lineTo(xPixel, centerY);
                    ctx.stroke();
                    ctx.setLineDash([]); // Restablecer para evitar afectar otros dibujos

                    // Sombrear el área
                    ctx.fillStyle = 'rgba(30, 64, 175, 0.2)'; // Azul con transparencia
                    if (inequalityDirection === '>' || inequalityDirection === '>=') {
                        ctx.fillRect(xPixel, -centerY, graphCanvas.width - centerX - xPixel, graphCanvas.height);
                    } else if (inequalityDirection === '<' || inequalityDirection === '<=') {
                        ctx.fillRect(-centerX, -centerY, xPixel + centerX, graphCanvas.height);
                    }

                    // Etiqueta de la línea
                    ctx.scale(1, -1);
                    ctx.fillStyle = '#1e40af';
                    ctx.fillText(`x ${inequalityDirection} ${graphValue.toFixed(2)}`, xPixel + 10, -graphCanvas.height / 2 + 30);
                    ctx.scale(1, -1);

                    // Dibujar círculo abierto/cerrado en el eje X
                    ctx.fillStyle = '#1e40af';
                    ctx.beginPath();
                    if (inequalityDirection === '>' || inequalityDirection === '<') {
                        ctx.arc(xPixel, 0, 7, 0, Math.PI * 2, true); // Círculo abierto
                        ctx.strokeStyle = '#1e40af';
                        ctx.lineWidth = 2;
                        ctx.stroke();
                        ctx.fillStyle = 'white';
                        ctx.fill();
                    } else {
                        ctx.arc(xPixel, 0, 7, 0, Math.PI * 2, true); // Círculo cerrado
                        ctx.fill();
                    }
                }
            }

            ctx.restore(); // Restaura el estado del contexto
        }

        /**
         * Realiza el cálculo y actualiza el gráfico.
         */
        function calculateAndGraph() {
            const input = equationInput.value;
            resultDisplay.textContent = 'Calculando...';
            solution = ''; // Limpiar solución anterior
            graphType = '';
            graphValue = null;
            inequalityDirection = null;

            try {
                const result = parseAndSolve(input);

                if (result) {
                    if (result.type === 'equality') {
                        solution = `La solución es: x = ${result.value.toFixed(4)}`;
                        graphType = 'equality';
                        graphValue = result.value;
                    } else if (result.type === 'inequality') {
                        solution = `La solución es: x ${result.direction} ${result.value.toFixed(4)}`;
                        graphType = 'inequality';
                        graphValue = result.value;
                        inequalityDirection = result.direction;
                    } else if (result.type === 'identity' || result.type === 'contradiction') {
                        solution = result.message;
                        graphType = ''; // No hay gráfico para identidades/contradicciones
                        graphValue = null;
                    }
                } else {
                    solution = 'Formato de ecuación o desigualdad no reconocido. Por favor, revisa el ejemplo.';
                }
            } catch (error) {
                console.error("Error al calcular:", error);
                solution = 'Ocurrió un error al procesar la expresión. Asegúrate de que sea una expresión lineal válida.';
            }

            resultDisplay.textContent = solution;
            drawGraph(); // Dibuja el gráfico con la nueva solución
        }

        /**
         * Lee en voz alta el contenido del resultado.
         */
        function speakResult() {
            const textToSpeak = resultDisplay.textContent;

            if ('speechSynthesis' in window) {
                const utterance = new SpeechSynthesisUtterance(textToSpeak);
                utterance.lang = 'es-ES'; // Establece el idioma a español
                speechSynthesis.speak(utterance);
            } else {
                resultDisplay.textContent = 'Tu navegador no soporta la síntesis de voz.';
            }
        }
    </script>
</body>
</html>
