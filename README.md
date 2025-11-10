<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Editor de Plantas Baixas</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            display: flex;
            height: 100vh;
            background-color: #f0f0f0;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            padding: 20px;
            box-sizing: border-box;
            overflow-y: auto;
        }
        .sidebar h2 {
            margin-top: 0;
        }
        .sidebar button {
            display: block;
            width: 100%;
            margin-bottom: 10px;
            padding: 10px;
            background-color: #555;
            color: white;
            border: none;
            cursor: pointer;
        }
        .sidebar button:hover {
            background-color: #777;
        }
        .sidebar input[type="range"] {
            width: 100%;
        }
        .mini-menu {
            display: none;
            background-color: #444;
            padding: 10px;
            margin-top: 10px;
            border-radius: 5px;
        }
        .mini-menu label {
            display: block;
            margin-bottom: 5px;
        }
        .mini-menu select {
            width: 100%;
            margin-bottom: 10px;
        }
        .canvas-container {
            flex: 1;
            overflow: auto; 
            background-color: white;
        }
        canvas {
            border: 1px solid #ccc;
            display: block;
        }
        @media (max-width: 768px) {
            body {
                flex-direction: column;
            }
            .sidebar {
                width: 100%;
                height: auto;
            }
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <h2>Ferramentas</h2>
        <button id="loadFloorplan">Carregar Planta Baixa</button>
        <input type="file" id="floorplanInput" accept="image/*" style="display: none;">
        <button id="addCircleTable">Adicionar Mesa Redonda</button>
        <button id="addSquareTable">Adicionar Mesa Quadrada</button>
        <button id="addStage">Adicionar Palco</button>
        <button id="addItem">Adicionar Item Personalizado</button>
        <div id="miniMenu" class="mini-menu">
            <label>Formato:</label>
            <select id="shapeSelect">
                <option value="circle">Círculo</option>
                <option value="square">Quadrado</option>
                <option value="rectangle">Retângulo</option>
            </select>
            <label>Cor:</label>
            <select id="colorSelect">
                <option value="#8B4513">Marrom</option>
                <option value="#555">Cinza Escuro</option>
                <option value="#0000FF">Azul</option>
                <option value="#008000">Verde</option>
                <option value="#FF0000">Vermelho</option>
            </select>
            <button id="confirmAdd">Adicionar</button>
        </div>
        <button id="undoItem">Retroceder (Undo)</button>
        <label for="itemSizeSlider">Tamanho do Item (pixels):</label>
        <input type="range" id="itemSizeSlider" min="5" max="100" step="1" value="30">
        <label for="scaleSlider">Escala (Zoom):</label>
        <input type="range" id="scaleSlider" min="0.5" max="3" step="0.1" value="1">
        <button id="saveCanvas">Salvar Canvas</button>
        <p>Mesas marrom, palco cinza escuro, itens personalizáveis. Clique no canvas para adicionar ou selecionar. Arraste o item selecionado para movê-lo. Use as barras de rolagem para navegar na planta.</p>
    </div>
    <div class="canvas-container">
        <canvas id="editorCanvas"></canvas>
    </div>

    <script>
        const canvas = document.getElementById('editorCanvas');
        const ctx = canvas.getContext('2d');
        const loadButton = document.getElementById('loadFloorplan');
        const fileInput = document.getElementById('floorplanInput');
        const addCircleButton = document.getElementById('addCircleTable');
        const addSquareButton = document.getElementById('addSquareTable');
        const addStageButton = document.getElementById('addStage');
        const addItemButton = document.getElementById('addItem');
        const miniMenu = document.getElementById('miniMenu');
        const shapeSelect = document.getElementById('shapeSelect');
        const colorSelect = document.getElementById('colorSelect');
        const confirmAddButton = document.getElementById('confirmAdd');
        const undoButton = document.getElementById('undoItem');
        const itemSizeSlider = document.getElementById('itemSizeSlider');
        const scaleSlider = document.getElementById('scaleSlider');
        const saveButton = document.getElementById('saveCanvas');

        let floorplanImage = null;
        let scale = 1; 
        let items = []; 
        let isAddingItem = false;
        let selectedItemIndex = -1; 
        let currentItemType = 'circle'; 
        let currentItemColor = '#8B4513'; 
        let itemSize = 30; 
        let isDragging = false;
        let dragIndex = -1;
        let dragOffsetX = 0;
        let dragOffsetY = 0;

  
        loadButton.addEventListener('click', () => {
            fileInput.click();
        });
        fileInput.addEventListener('change', (e) => {
            const file = e.target.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = (event) => {
                    const img = new Image();
                    img.onload = () => {
                        floorplanImage = img;
                        
                        canvas.width = img.width * scale;
                        canvas.height = img.height * scale;
                        drawCanvas();
                    };
                    img.src = event.target.result;
                };
                reader.readAsDataURL(file);
            }
        });

       
        addCircleButton.addEventListener('click', () => {
            isAddingItem = true;
            currentItemType = 'circle';
            currentItemColor = '#8B4513'; // Marrom padrão
            selectedItemIndex = -1;
            canvas.style.cursor = 'crosshair';
        });

       
        addSquareButton.addEventListener('click', () => {
            isAddingItem = true;
            currentItemType = 'square';
            currentItemColor = '#8B4513'; // Marrom padrão
            selectedItemIndex = -1;
            canvas.style.cursor = 'crosshair';
        });

      
        addStageButton.addEventListener('click', () => {
            isAddingItem = true;
            currentItemType = 'rectangle';
            currentItemColor = '#555'; // Cinza escuro padrão
            selectedItemIndex = -1;
            canvas.style.cursor = 'crosshair';
        });

       
        addItemButton.addEventListener('click', () => {
            miniMenu.style.display = miniMenu.style.display === 'block' ? 'none' : 'block';
        });

       
        confirmAddButton.addEventListener('click', () => {
            currentItemType = shapeSelect.value;
            currentItemColor = colorSelect.value;
            isAddingItem = true;
            selectedItemIndex = -1;
            canvas.style.cursor = 'crosshair';
            miniMenu.style.display = 'none';
        });

        
        undoButton.addEventListener('click', () => {
            if (items.length > 0) {
                items.pop(); // Remove o último item adicionado
                selectedItemIndex = -1; // Desseleciona
                drawCanvas();
            }
        });

       
        itemSizeSlider.addEventListener('input', (e) => {
            itemSize = parseInt(e.target.value);
            if (selectedItemIndex >= 0) {
                items[selectedItemIndex].size = itemSize; // Ajusta o item selecionado
                drawCanvas();
            }
        });

       
        scaleSlider.addEventListener('input', (e) => {
            scale = parseFloat(e.target.value);
            if (floorplanImage) {
                // Redimensiona o canvas baseado na imagem e zoom
                canvas.width = floorplanImage.width * scale;
                canvas.height = floorplanImage.height * scale;
                drawCanvas();
            }
        });

       
        canvas.addEventListener('mousedown', (e) => {
            const rect = canvas.getBoundingClientRect();
            const x = e.clientX - rect.left;
            const y = e.clientY - rect.top;

        
            let itemClicked = false;
            for (let i = 0; i < items.length; i++) {
                const item = items[i];
                let hit = false;
                if (item.type === 'circle') {
                    const distance = Math.sqrt((x - item.x) ** 2 + (y - item.y) ** 2);
                    hit = distance <= item.size;
                } else if (item.type === 'square') {
                    hit = x >= item.x - item.size / 2 && x <= item.x + item.size / 2 &&
                          y >= item.y - item.size / 2 && y <= item.y + item.size / 2;
                } else if (item.type === 'rectangle') {
                    hit = x >= item.x - item.size / 2 && x <= item.x + item.size / 2 &&
                          y >= item.y - item.size / 4 && y <= item.y + item.size / 4;
                }
                if (hit) {
                    selectedItemIndex = i;
                    itemSizeSlider.value = item.size; 
                   
                    isDragging = true;
                    dragIndex = i;
                    dragOffsetX = x - item.x;
                    dragOffsetY = y - item.y;
                    canvas.style.cursor = 'move';
                    itemClicked = true;
                    break;
                }
            }

           
            if (!itemClicked && isAddingItem) {
                items.push({ type: currentItemType, x, y, size: itemSize, color: currentItemColor });
                isAddingItem = false;
                canvas.style.cursor = 'default';
                drawCanvas();
            }
        });

        canvas.addEventListener('mousemove', (e) => {
            if (isDragging && dragIndex >= 0) {
                const rect = canvas.getBoundingClientRect();
                const x = e.clientX - rect.left;
                const y = e.clientY - rect.top;
                items[dragIndex].x = x - dragOffsetX;
                items[dragIndex].y = y - dragOffsetY;
                drawCanvas();
            }
        });

        canvas.addEventListener('mouseup', () => {
            if (isDragging) {
                isDragging = false;
                dragIndex = -1;
                canvas.style.cursor = 'default';
            }
        });

        canvas.addEventListener('contextmenu', (e) => {
            e.preventDefault();
            const rect = canvas.getBoundingClientRect();
            const x = e.clientX - rect.left;
            const y = e.clientY - rect.top;
            items = items.filter(item => {
                if (item.type === 'circle') {
                    return !(Math.abs(item.x - x) < item.size && Math.abs(item.y - y) < item.size);
                } else if (item.type === 'square') {
                    return !(x >= item.x - item.size / 2 && x <= item.x + item.size / 2 &&
                             y >= item.y - item.size / 2 && y <= item.y + item.size / 2);
                } else if (item.type === 'rectangle') {
                    return !(x >= item.x - item.size / 2 && x <= item.x + item.size / 2 &&
                             y >= item.y - item.size / 4 && y <= item.y + item.size / 4);
                }
                return true;
            }); // Remove se clicar no item
            selectedItemIndex = -1; // Desseleciona
            drawCanvas();
        });

        // Desenhar canvas
        function drawCanvas() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            if (floorplanImage) {
                ctx.drawImage(floorplanImage, 0, 0, canvas.width, canvas.height);
            }
            // Desenhar itens
            items.forEach((item, index) => {
                ctx.beginPath();
                ctx.fillStyle = item.color;
                if (item.type === 'circle') {
                    ctx.arc(item.x, item.y, item.size, 0, Math.PI * 2);
                } else if (item.type === 'square') {
                    ctx.rect(item.x - item.size / 2, item.y - item.size / 2, item.size, item.size);
                } else if (item.type === 'rectangle') {
                    ctx.rect(item.x - item.size / 2, item.y - item.size / 4, item.size, item.size / 2);
                }
                ctx.fill();
                if (index === selectedItemIndex) {
                    // Destaca o item selecionado com borda branca
                    ctx.strokeStyle = 'white';
                    ctx.lineWidth = 2;
                    ctx.stroke();
                }
            });
        }

       
        saveButton.addEventListener('click', () => {
            const link = document.createElement('a');
            link.download = 'planta-editada.png';
            link.href = canvas.toDataURL();
            link.click();
        });

        
        drawCanvas();
    </script>
</body>
</html>
