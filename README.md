<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Calculator</title>
<style>
  :root {
    --bg: #121212;
    --calc-bg: #1e1e1e;
    --display-bg: #0d0d0d;
    --btn-bg: #2a2a2a;
    --btn-hover: #383838;
    --btn-op: #ff9500;
    --btn-op-hover: #ffab33;
    --btn-clear: #a83232;
    --btn-clear-hover: #c23e3e;
    --text: #f5f5f5;
    --text-dim: #9a9a9a;
  }

  * {
    box-sizing: border-box;
  }

  body {
    margin: 0;
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg);
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  }

  .calculator {
    width: 320px;
    max-width: 92vw;
    background: var(--calc-bg);
    border-radius: 20px;
    padding: 20px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  }

  .display {
    background: var(--display-bg);
    border-radius: 12px;
    padding: 20px 16px;
    margin-bottom: 16px;
    text-align: right;
    overflow: hidden;
  }

  .display .previous {
    color: var(--text-dim);
    font-size: 1rem;
    min-height: 1.2rem;
    word-wrap: break-word;
  }

  .display .current {
    color: var(--text);
    font-size: 2.2rem;
    font-weight: 500;
    margin-top: 4px;
    white-space: nowrap;
    overflow-x: auto;
  }

  .buttons {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 10px;
  }

  button {
    border: none;
    outline: none;
    border-radius: 12px;
    padding: 18px 0;
    font-size: 1.2rem;
    font-weight: 500;
    color: var(--text);
    background: var(--btn-bg);
    cursor: pointer;
    transition: background 0.15s ease, transform 0.05s ease;
  }

  button:hover {
    background: var(--btn-hover);
  }

  button:active {
    transform: scale(0.96);
  }

  .btn-operator {
    background: var(--btn-op);
    color: #1a1a1a;
  }

  .btn-operator:hover {
    background: var(--btn-op-hover);
  }

  .btn-clear {
    background: var(--btn-clear);
  }

  .btn-clear:hover {
    background: var(--btn-clear-hover);
  }

  .btn-equals {
    background: #34a853;
    color: #0d1f12;
  }

  .btn-equals:hover {
    background: #45c265;
  }

  .span-two {
    grid-column: span 2;
  }
</style>
</head>
<body>

<div class="calculator">
  <div class="display">
    <!-- Shows the pending expression, e.g. "12 +" -->
    <div class="previous" id="previous-display"></div>
    <!-- Shows the number currently being typed / the result -->
    <div class="current" id="current-display">0</div>
  </div>

  <div class="buttons">
    <button class="btn-clear span-two" data-action="clear">C</button>
    <button data-action="delete">DEL</button>
    <button class="btn-operator" data-operator="÷">÷</button>

    <button data-number>7</button>
    <button data-number>8</button>
    <button data-number>9</button>
    <button class="btn-operator" data-operator="×">×</button>

    <button data-number>4</button>
    <button data-number>5</button>
    <button data-number>6</button>
    <button class="btn-operator" data-operator="−">−</button>

    <button data-number>1</button>
    <button data-number>2</button>
    <button data-number>3</button>
    <button class="btn-operator" data-operator="+">+</button>

    <button class="span-two" data-number>0</button>
    <button data-number>.</button>
    <button class="btn-equals" data-action="equals">=</button>
  </div>
</div>

<script>
  /*
    Calculator logic overview
    --------------------------
    We track three pieces of state:
      - previousOperand : the number entered before an operator was pressed
      - currentOperand   : the number currently being typed (shown large)
      - operator          : the pending operation (+, −, ×, ÷)

    The flow is:
      1. User types digits -> appended to currentOperand.
      2. User presses an operator -> if there's already a pending operation,
         compute it first (so "5 + 3 + " chains correctly), then store the
         current number as previousOperand and remember the new operator.
      3. User presses "=" -> compute previousOperand (operator) currentOperand
         and show the result as the new currentOperand.
      4. "C" resets everything. "DEL" removes the last typed character.
  */

  const previousDisplay = document.getElementById('previous-display');
  const currentDisplay = document.getElementById('current-display');

  let currentOperand = '0';
  let previousOperand = '';
  let operator = undefined;

  // Re-renders both display lines based on current state
  function updateDisplay() {
    currentDisplay.textContent = formatDisplayNumber(currentOperand);
    previousDisplay.textContent = operator
      ? `${formatDisplayNumber(previousOperand)} ${operator}`
      : '';
  }

  // Adds thousands separators for readability without altering the
  // underlying numeric value used in calculations.
  function formatDisplayNumber(value) {
    if (value === '' || value === undefined) return '';
    const stringValue = value.toString();
    const [integerPart, decimalPart] = stringValue.split('.');
    const formattedInteger = isNaN(parseFloat(integerPart))
      ? integerPart
      : parseFloat(integerPart).toLocaleString('en-US');
    return decimalPart !== undefined
      ? `${formattedInteger}.${decimalPart}`
      : formattedInteger;
  }

  // Appends a digit or decimal point to the number being typed
  function appendNumber(digit) {
    // Prevent multiple decimal points in the same number
    if (digit === '.' && currentOperand.includes('.')) return;

    // Replace a lone leading "0" (unless we're adding a decimal point)
    if (currentOperand === '0' && digit !== '.') {
      currentOperand = digit;
    } else {
      currentOperand += digit;
    }
  }

  // Stores the chosen operator and shifts the current number into "previous"
  function chooseOperator(selectedOperator) {
    if (currentOperand === '' ) return;

    // If an operation is already pending, resolve it first so operators
    // can be chained, e.g. "4 + 5 + 6" evaluates step by step.
    if (previousOperand !== '') {
      compute();
    }

    operator = selectedOperator;
    previousOperand = currentOperand;
    currentOperand = '';
  }

  // Performs the actual arithmetic based on the stored operator
  function compute() {
    const prev = parseFloat(previousOperand);
    const current = parseFloat(currentOperand);

    // Guard against incomplete input (e.g. pressing "=" with nothing typed)
    if (isNaN(prev) || isNaN(current)) return;

    let result;
    switch (operator) {
      case '+':
        result = prev + current;
        break;
      case '−':
        result = prev - current;
        break;
      case '×':
        result = prev * current;
        break;
      case '÷':
        // Handle division by zero gracefully instead of showing "Infinity"
        result = current === 0 ? 'Error' : prev / current;
        break;
      default:
        return;
    }

    // Round to avoid floating point artifacts (e.g. 0.1 + 0.2 = 0.30000000000000004)
    if (typeof result === 'number') {
      result = Math.round(result * 1e10) / 1e10;
    }

    currentOperand = result.toString();
    operator = undefined;
    previousOperand = '';
  }

  function clearAll() {
    currentOperand = '0';
    previousOperand = '';
    operator = undefined;
  }

  function deleteLastCharacter() {
    currentOperand = currentOperand.toString().slice(0, -1);
    if (currentOperand === '') currentOperand = '0';
  }

  // Wire up number buttons
  document.querySelectorAll('[data-number]').forEach(button => {
    button.addEventListener('click', () => {
      appendNumber(button.textContent);
      updateDisplay();
    });
  });

  // Wire up operator buttons
  document.querySelectorAll('[data-operator]').forEach(button => {
    button.addEventListener('click', () => {
      chooseOperator(button.dataset.operator);
      updateDisplay();
    });
  });

  // Wire up equals button
  document.querySelector('[data-action="equals"]').addEventListener('click', () => {
    compute();
    updateDisplay();
  });

  // Wire up clear (C) button
  document.querySelector('[data-action="clear"]').addEventListener('click', () => {
    clearAll();
    updateDisplay();
  });

  // Wire up delete (DEL) button
  document.querySelector('[data-action="delete"]').addEventListener('click', () => {
    deleteLastCharacter();
    updateDisplay();
  });

  // Optional: basic keyboard support for convenience
  document.addEventListener('keydown', (e) => {
    if (e.key >= '0' && e.key <= '9') appendNumber(e.key);
    else if (e.key === '.') appendNumber('.');
    else if (e.key === '+') chooseOperator('+');
    else if (e.key === '-') chooseOperator('−');
    else if (e.key === '*') chooseOperator('×');
    else if (e.key === '/') { e.preventDefault(); chooseOperator('÷'); }
    else if (e.key === 'Enter' || e.key === '=') compute();
    else if (e.key === 'Backspace') deleteLastCharacter();
    else if (e.key === 'Escape') clearAll();
    else return;

    updateDisplay();
  });

  updateDisplay();
</script>

</body>
</html>
