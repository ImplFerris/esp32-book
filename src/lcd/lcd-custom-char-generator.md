## LCD Custom Character Generator (5x8 Grid)

Select the grids to create a symbol or character. As you select the grid, the corresponding bits in the byte array will be updated.

<style>
    .container {
        display: flex;
        align-items: flex-start;
        gap: 20px;
    }

    .grid-wrapper {
        background-color: #0076CE;
        padding: 10px;
        width: 250px;
        height: auto;
        box-sizing: border-box;
        /* display: flex; */
        justify-content: center;
        align-items: center;
        margin-bottom: 10px;
    }

    .grid {
        display: grid;
        grid-template-columns: repeat(5, 40px);
        grid-template-rows: repeat(8, 40px);
        gap: 5px;
        margin-bottom: 10px;
    }

    .cell {
        width: 40px;
        height: 40px;
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
        margin-left: 100px;
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
        gap: 10px;
        margin-top: 10px;
    }

    button {
        padding: 10px;
        margin: 5px;
        background-color: #0076CE;
        color: white;
        border: none;
        border-radius: 5px;
        cursor: pointer;
    }

    button:hover {
        background-color: #005f8c;
    }

    .text-left {
        text-align: left;
    }
</style>

<div class="container">
    <div class="grid-wrapper">
        <div class="grid" id="grid"></div>
    </div>
    <div class="output" id="output">
        <h4 class="text-left">Generated Array</h4>
        <code id="rust-code"></code>
        <div class="button-wrapper">
            <button id="copy-btn">Copy</button>
        </div>
    </div>
</div>

<div class="button-wrapper">
    <button id="clear-btn">Clear</button>
    <button id="invert-btn">Invert</button>
</div>

<script>
    const gridContainer = document.getElementById('grid');
    const outputContainer = document.getElementById('rust-code');
    const clearButton = document.getElementById('clear-btn');
    const invertButton = document.getElementById('invert-btn');
    const copyButton = document.getElementById('copy-btn');
    const rows = 8;
    const cols = 5;

    for (let i = 0; i < rows * cols; i++) {
        const cell = document.createElement('button');
        cell.classList.add('cell');
        cell.dataset.row = Math.floor(i / cols);
        cell.dataset.col = i % cols;
        cell.addEventListener('click', () => {
            cell.classList.toggle('selected');
            updateOutput();
        });
        gridContainer.appendChild(cell);
    }

    function updateOutput() {
        let rustArray = [];
        for (let r = 0; r < rows; r++) {
            let rowArray = [];
            for (let c = 0; c < cols; c++) {
                const cell = document.querySelector(`.cell[data-row="${r}"][data-col="${c}"]`);
                rowArray.push(cell.classList.contains('selected') ? '1' : '0');
            }
            rustArray.push(`0b${rowArray.join('')},`);
        }
        let tc = "[\n" + `    ${rustArray.join('\n    ')}` + "\n]";
        outputContainer.textContent = tc;
    }

    updateOutput();

    clearButton.addEventListener('click', () => {
        const cells = document.querySelectorAll('.cell');
        cells.forEach(cell => cell.classList.remove('selected'));
        updateOutput();
    });

    invertButton.addEventListener('click', () => {
        const cells = document.querySelectorAll('.cell');
        cells.forEach(cell => {
            cell.classList.toggle('selected');
        });
        updateOutput();
    });

    copyButton.addEventListener('click', () => {
        const trimmedCode = outputContainer.textContent
            .replace(/\s+/g, ' ')
            .replace(/,\s+/g, ', ')
            .trim();
        navigator.clipboard.writeText(trimmedCode).then(() => {
            alert('Code copied to clipboard!');
        }).catch(err => {
            console.error('Error copying code: ', err);
        });
    });

</script>
