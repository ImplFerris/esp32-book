## Multi Custom Character Generator

This is used when you want to combine multiple grids to create a symbol. You can utilize adjacent grids on the 16x2 LCD display to design a custom symbol or character.  You can view the example symbol created with this generator and how to use in Rust in the next page.
 
<style>
    .container {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap:0 10px;
    }

    .grid-wrapper {
        background-color: #0076CE;
        padding: 10px;
        display: flex;
        flex-wrap: wrap;
        gap: 10px;
        justify-content: center;
        align-items: center;
        margin-bottom: 10px;
    }

    .grid {
        display: grid;
        grid-template-columns: repeat(5, 20px);
        grid-template-rows: repeat(8, 20px);
        gap: 5px;
    }

    .cell {
        width: 20px;
        height: 20px;
        background-color: #0076CE;
        border: 1px solid white;
        outline: none;
        cursor: pointer;
    }

    .cell.selected {
        background-color: white;
    }

    .output {
        margin-top: 20px;
        font-family: monospace;
        text-align: center;
        width: 100%;
    }

    code {
        background-color: #eee;
        padding: 5px;
        border-radius: 5px;
        display: block !important;
        width: 100%;
        white-space: pre-wrap;
        text-align: left;
    }

    .button-wrapper {
        display: flex;
        justify-content: flex-start;
        gap: 4px;
        margin-top: 10px;
        margin-bottom: 10px;
    }

    button {
        padding: 8px;
        margin: 5px;
        background-color: #0076CE;
        color: white;
        border: none;
        /* border-radius: 5px; */
        cursor: pointer;
    }

    button:hover {
        background-color: #005f8c;
    }

    .text-left {
        text-align: left;
    }
    
   .custom-char-tbl {
    width: 100%;
    border-collapse: collapse;
     border: none; 
}

.custom-char-tbl td {
    width: auto;
    height: auto;
    text-align: center;
    vertical-align: middle;
    padding: 0; 
     border: none; 
}

.grid {
    display: grid;
    grid-template-columns: repeat(5, 20px);
    grid-template-rows: repeat(8, 20px);
    gap: 5px;
    margin: auto;
}

.cell {
    width: 20px;
    height: 20px;
    background-color: #0076CE;
    border: 1px solid white;
    outline: none;
    cursor: pointer;
}

.cell.selected {
    background-color: white;
}

.custom-char-tbl td div.grid {
    margin: auto; 
    width: calc(20px * 5 + 5px * 6); 
    height: calc(20px * 8 + 5px * 10); 
}

</style>

<div class="container">
    <div class="button-wrapper">
        <button id="clear-btn">Clear</button>
        <button id="invert-btn">Invert</button>
        <select id="grid-size">
            <option value="2">2 Grids</option>
            <option value="4">4 Grids</option>
            <option value="6">6 Grids</option>
            <option value="8">8 Grids</option>
        </select>
        <select id="grid-layout">
            <option value="single-row">Single Row</option>
            <option value="single-column">Single Column</option>
        </select>
    </div>
    <div class="grid-wrapper" id="grid-wrapper"></div>
    <div class="output" id="output">
        <h4 class="text-left">Generated Array</h4>
        <code id="rust-code"></code>
        <div class="button-wrapper">
            <button id="copy-btn">Copy</button>
        </div>
    </div>
</div>

<script>
    const gridWrapper = document.getElementById('grid-wrapper');
    const outputContainer = document.getElementById('rust-code');
    const clearButton = document.getElementById('clear-btn');
    const invertButton = document.getElementById('invert-btn');
    const copyButton = document.getElementById('copy-btn');
    const gridSizeSelector = document.getElementById('grid-size');
    const gridLayoutSelector = document.getElementById('grid-layout');

    let gridCount = parseInt(gridSizeSelector.value);
    let layout = gridLayoutSelector.value;

    // Function to create and validate grid layout
    function validateLayout() {
        const gridSize = parseInt(gridSizeSelector.value);

        if (gridSize === 4) {
            gridLayoutSelector.innerHTML = `
                <option value="single-row">Single Row</option>
                <option value="two-rows">2 Rows</option>
            `;
        } else if (gridSize === 6 || gridSize === 8) {
            gridLayoutSelector.innerHTML = `
                <option value="single-row">Single Row</option>
                <option value="two-rows">2 Rows</option>
            `;
        } else {
            gridLayoutSelector.innerHTML = `
                <option value="single-row">Single Row</option>
                <option value="single-column">Single Column</option>
            `;
        }

        // Always default to the first valid layout
        gridLayoutSelector.value = gridLayoutSelector.options[0].value;
    }

    // Function to adjust cell size based on grid count
    // function adjustCellSize(count) {
    //     const cells = document.querySelectorAll('.cell');
    //     let cellSize = 40; // Default size

    //     if (count === 6) {
    //         cellSize = 20; // Smaller size for 6 grids
    //     } else if (count === 8) {
    //         cellSize = 20; // Smaller size for 8 grids
    //     }

    //     cells.forEach(cell => {
    //         cell.style.width = `${cellSize}px`;
    //         cell.style.height = `${cellSize}px`;
    //     });
    // }

    // Function to create grids inside table
    function createGrids(count, layout) {
        gridWrapper.innerHTML = '';

        let rows, columns;

        if (layout === 'single-row') {
            rows = 1;
            columns = count;
        } else if (layout === 'single-column') {
            rows = count;
            columns = 1;
        } else if (layout === 'two-rows') {
            rows = 2;
            columns = count === 4 ? 2 : count === 6 ? 3 : 4;
        }

        const table = document.createElement('table');
        table.classList.add('custom-char-tbl');  

        for (let r = 0; r < rows; r++) {
            const tr = document.createElement('tr');
            for (let c = 0; c < columns; c++) {
                if ((r * columns + c) >= count) break;

                const td = document.createElement('td');
                const grid = document.createElement('div');
                grid.classList.add('grid');
                grid.dataset.gridId = r * columns + c;

                for (let i = 0; i < 5 * 8; i++) {
                    const cell = document.createElement('button');
                    cell.classList.add('cell');
                    cell.dataset.row = Math.floor(i / 5);
                    cell.dataset.col = i % 5;
                    cell.dataset.gridId = r * columns + c;
                    cell.addEventListener('click', () => {
                        cell.classList.toggle('selected');
                        updateOutput(count);
                    });
                    grid.appendChild(cell);
                }

                td.appendChild(grid);
                tr.appendChild(td);
            }
            table.appendChild(tr);
        }

        gridWrapper.appendChild(table);

        // Adjust cell sizes after grid creation
        // adjustCellSize(count);
    }

    // Function to update the output code
    function updateOutput(count) {
        let rustCode = '';

        for (let g = 0; g < count; g++) {
            let rustArrays = [];
            for (let r = 0; r < 8; r++) {
                let rowArray = [];
                for (let c = 0; c < 5; c++) {
                    const cell = document.querySelector(`.cell[data-grid-id="${g}"][data-row="${r}"][data-col="${c}"]`);
                    rowArray.push(cell.classList.contains('selected') ? '1' : '0');
                }
                rustArrays.push(`0b${rowArray.join('')},`);
            }
            rustCode += `const SYMBOL${g + 1}: [u8; 8] = [
    ${rustArrays.join('\n    ')}
];\n\n`;
        }

        outputContainer.textContent = rustCode;
    }

    // Event listeners
    gridSizeSelector.addEventListener('change', () => {
        gridCount = parseInt(gridSizeSelector.value);
        validateLayout(); // Apply layout restriction
        createGrids(gridCount, gridLayoutSelector.value);
        updateOutput(gridCount);
    });

    gridLayoutSelector.addEventListener('change', () => {
        createGrids(gridCount, gridLayoutSelector.value);
        updateOutput(gridCount);
    });

    clearButton.addEventListener('click', () => {
        const cells = document.querySelectorAll('.cell');
        cells.forEach(cell => cell.classList.remove('selected'));
        updateOutput(gridCount);
    });

    invertButton.addEventListener('click', () => {
        const cells = document.querySelectorAll('.cell');
        cells.forEach(cell => {
            cell.classList.toggle('selected');
        });
        updateOutput(gridCount);
    });

    copyButton.addEventListener('click', () => {
        navigator.clipboard.writeText(outputContainer.textContent).then(() => {
            alert('Code copied to clipboard!');
        }).catch(err => {
            console.error('Error copying code: ', err);
        });
    });

    // Initialize the grid and output
    createGrids(gridCount, layout);
    updateOutput(gridCount);
</script>
