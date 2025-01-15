
## Steinhart Hart equation
The Steinhart-Hart equation provides a more accurate temperature-resistance relationship over a wide temperature range. 
\\[
\frac{1}{T} = A + B \ln R + C (\ln R)^3
\\]

Where:
- T is the temperature in **Kelvins**. (Formula to calculate kelvin from degree Celsius, K = °C + 273.15)
- R is the resistance at temperature T in **Ohms**.
- A, B, and C are constants specific to the thermistor's material, often provided by the manufacturer. For better accuracy, you may need to calibrate and determine these values yourself. Some datasheets provide resistance values at various temperatures, which can also be used to calculate this.

**Note**:

We won't use this equation in our exercise because it takes more effort to find the A, B, and C constants. The B equation provides sufficient accuracy for our purposes, so feel free to skip this chapter if you prefer.


### Calibration
To determine the accurate values for A, B, and C, place the thermistor in three temperature conditions: room temperature, ice water, and boiling water. For each condition, measure the thermistor's resistance using the ADC value and use a reliable thermometer to record the actual temperature. Using the resistance values and corresponding temperatures, calculate the coefficients:
- Assign A to the ice water temperature,
- B to the room temperature, and
- C to the boiling water temperature.

### Calculating Steinhart-Hart Coefficients

With three resistance and temperature data points, we can find the A, B and C.

$$
\begin{bmatrix}
    1 & \ln R_1 & \ln^3 R_1 \\\\
    1 & \ln R_2 & \ln^3 R_2 \\\\
    1 & \ln R_3 & \ln^3 R_3
\end{bmatrix}\begin{bmatrix}
    A \\\\
    B \\\\
    C
\end{bmatrix} = \begin{bmatrix}
    \frac{1}{T_1} \\\\
    \frac{1}{T_2} \\\\
    \frac{1}{T_3}
\end{bmatrix}
$$

Where:
- \\( R_1, R_2, R_3 \\) are the resistance values at temperatures \\( T_1, T_2, T_3 \\).

**Let's calculate the coefficients**

Compute the natural logarithms of resistances:
$$
L_1 = \ln R_1, \quad L_2 = \ln R_2, \quad L_3 = \ln R_3
$$

Intermediate calculations:
$$
Y_1 = \frac{1}{T_1}, \quad Y_2 = \frac{1}{T_2}, \quad Y_3 = \frac{1}{T_3}
$$

$$
\gamma_2 = \frac{Y_2 - Y_1}{L_2 - L_1}, \quad \gamma_3 = \frac{Y_3 - Y_1}{L_3 - L_1}
$$

So, finally:
$$
C = \left( \frac{ \gamma_3 - \gamma_2 }{ L_3 - L_2} \right) \left(L_1 + L_2 + L_3\right)^{-1} \\
$$
$$
B = \gamma_2 - C \left(L_1^2 + L_1 L_2 + L_2^2\right) \\
$$
$$
A = Y_1 - \left(B + L_1^2 C\right) L_1
$$


<span style="color: green;">Good news, Everyone!</span> You don't need to calculate the coefficients manually. Simply provide the resistance and temperature values for cold, room, and hot environments, and use the form below to determine A, B and C

### ADC value and Resistance Calculation
<span style="color: orange;">Note:</span> if you already have the temperature and corresponding resistance, you can directly use the second table to input those values.

If you have the ADC value and want to calculate the resistance, use this table to find the corresponding resistance at different temperatures. As you enter the ADC value for each temperature, the calculated resistance will be automatically updated in the second table.

To perform this calculation, you'll need the base resistance of the thermistor, which is essential for determining the resistance at a given temperature based on the ADC value.

Please note that the ADC bits may need to be adjusted if you're using a different microcontroller. In our case, for the ESP32, the ADC resolution is 12 bits.

<form id="adcForm">
  <label for="baseResistance">Base Resistance (Ω): </label>
  <input type="number" id="baseResistance" name="baseResistance" step="any" value="10000" oninput="updateResistance()">
    <br/>
    <label for="adcBits">ADC Bits: </label>
  <input type="number" id="adcBits" name="adcBits" step="any" value="12" oninput="updateResistance()">
  <br/>
  <br/>

  <table>
    <thead>
      <tr>
         <th>Environment</th>
        <th>ADC value</th>
      </tr>
    </thead>
    <tbody>
      <!-- Cold Water Row -->
      <tr>
        <td>Cold Water</td>
        <td><input type="number" id="adcColdCount" name="adcColdCount" step="any" oninput="updateResistance()"></td>
      </tr>
      <!-- Room Temperature Row -->
      <tr>
        <td>Room Temperature</td>
        <td><input type="number" id="adcRoomCount" name="adcRoomCount" step="any" oninput="updateResistance()"></td>
      </tr>
      <!-- Boiling Water Row -->
      <tr>
      <td>Boiling Water</td>
        <td><input type="number" id="adcBoilCount" name="adcBoilCount" step="any" oninput="updateResistance()"></td>
      </tr>
    </tbody>
  </table>
</form>


### Coefficients Finder
Adjust the temperature by entering a value in either Fahrenheit or Celsius; the form will automatically convert it to the other format. Provide the resistance corresponding to each temperature, and then click the "Calculate Coefficients" button.
<form id="steinhartForm" onsubmit="calcCoeffBtnClicked(event)">
<table>
<thead>
<tr>
<th>Environment</th>
<th>Resistance (Ohms)</th>
<th>Temperature (°F)</th>
<th>Temperature (°C)</th>
<th>Temperature (K)</th>
</tr>
</thead>
<tbody>
<tr>
<td>Cold Water</td>
<td><input type="number" id="resistanceCold" name="resistanceCold" step="any" oninput="validateInput()"></td>
<td><input type="number" id="coldTempF" name="coldTempF" step="any" oninput="calcTempFromFarenhit('coldTempC', 'coldTempF', 'coldTempK', 'resistanceCold')"></td>
<td><input type="number" id="coldTempC" name="coldTempC" step="any" oninput="calcTempFromCel('coldTempC', 'coldTempF', 'coldTempK', 'resistanceCold')"></td>

<td><input type="number" id="coldTempK" name="coldTempK" step="any" readonly></td>
</tr>
<tr>
<td>Room Temperature</td>
<td><input type="number" id="resistanceRoom" name="resistanceRoom" step="any" oninput="validateInput()"></td>
<td><input type="number" id="roomTempF" name="roomTempF" step="any" oninput="calcTempFromFarenhit('roomTempC', 'roomTempF', 'roomTempK', 'resistanceRoom')"></td>
<td><input type="number" id="roomTempC" name="roomTempC" step="any" value="25"  oninput="calcTempFromCel('roomTempC', 'roomTempF', 'roomTempK', 'resistanceRoom')"></td>
<td><input type="number" id="roomTempK" name="roomTempK" step="any" readonly></td>
</tr>
<tr>
<td>Boiling Water</td>
<td><input type="number" id="resistanceBoiling" name="resistanceBoiling"  step="any" oninput="validateInput()"></td>

<td><input type="number" id="boilTempF" name="boilTempF" step="any"  oninput="calcTempFromFarenhit('boilTempC', 'boilTempF', 'boilTempK', 'resistanceBoiling')"></td>
<td><input type="number" id="boilTempC" name="boilTempC" step="any" oninput="calcTempFromCel('boilTempC', 'boilTempF', 'boilTempK', 'resistanceBoiling')"></td>
<td><input type="number" id="boilTempK" name="boilTempK" step="any" readonly></td>
</tr>
</tbody>
</table>

<h3>Results</h3>
<p>
    A: <input type="text" id="resultA" readonly />
    <span id="actualA"></span> 
</p>
<p>
    B: <input type="text" id="resultB" readonly />
    <span id="actualB"></span> 
</p>
<p>
    C: <input type="text" id="resultC" readonly />
    <span id="actualC"></span> 
</p>


<button type="submit" id="submitBtn" >Calculate Coefficients</button>
</form>

<h3>Calculate Temperature from Resistance</h3>
<p>Now, with these coefficients, you can calculate the temperature for any given resistance:</p>

<label for="r">R (Ohms): </label>
<input type="number" name="r" value="10000" id="inputResistance" >

<button type="button" id="calculateBtn" onclick="calculateTemperatureFromResistance()" >Calculate Temperature</button>

<label for="tc">Result (°C): </label>
<input type="text" name="tc" id="resultCelsius" readonly>

<label for="tf">Result (°F): </label>
<input type="text" name="tf" id="resultFahrenheit" readonly>


<!-- Error Message Section -->
<p id="errorMessage" style="color: red; display: none;">Error: Please calculate the coefficients (A, B, C) first.</p>

<script>
window.onload = function() {
  // Default values for Cold Water
  document.getElementById("resistanceCold").value = 25000;
  document.getElementById("coldTempC").value = 5;
  calcTempFromCel('coldTempC', 'coldTempF', 'coldTempK', 'resistanceCold');
  
  // Default values for Room Temperature
  document.getElementById("resistanceRoom").value = 10000;
  document.getElementById("roomTempC").value = 25;
  calcTempFromCel('roomTempC', 'roomTempF', 'roomTempK', 'resistanceRoom');
  
  // Default values for Boiling Water
  document.getElementById("resistanceBoiling").value = 4000;
  document.getElementById("boilTempC").value = 45;
  calcTempFromCel('boilTempC', 'boilTempF', 'boilTempK', 'resistanceBoiling');

  calculateCoefficients();
};

// Function to calculate resistance based on base resistance and ADC value
function calculateResistance(baseResistance, adcCount, adcBits) {
  const maxADCValue = Math.pow(2, adcBits) - 1;  // Max ADC value for the given bits (e.g., 12 bits = 4095)
  
  const resistance = baseResistance * ((maxADCValue / adcCount)-1);
  
  return resistance;
}

function updateResistance() {
  const baseResistance = parseFloat(document.getElementById("baseResistance").value);
  const adcBits = parseInt(document.getElementById("adcBits").value);
  
  const adcColdCount = parseFloat(document.getElementById("adcColdCount").value);
  const adcRoomCount = parseFloat(document.getElementById("adcRoomCount").value);
  const adcBoilCount = parseFloat(document.getElementById("adcBoilCount").value);
  
  // Calculate resistance for each environment using the ADC counts
  if (!isNaN(baseResistance) && !isNaN(adcBits)) {
    const resistanceCold = calculateResistance(baseResistance, adcColdCount, adcBits);
    document.getElementById("resistanceCold").value = resistanceCold.toFixed(2);

    const resistanceRoom = calculateResistance(baseResistance, adcRoomCount, adcBits);
    document.getElementById("resistanceRoom").value = resistanceRoom.toFixed(2);

    const resistanceBoiling = calculateResistance(baseResistance, adcBoilCount, adcBits);
    document.getElementById("resistanceBoiling").value = resistanceBoiling.toFixed(2);
  }
}

function calcTempFromCel(celsiusId, fahrenheitId, kelvinId, resistanceId) {
    const tempC = parseFloat(document.getElementById(celsiusId).value);

    if (!isNaN(tempC)) {
        const tempF = (tempC * 9/5) + 32;
        const tempK = tempC + 273.15;
        document.getElementById(fahrenheitId).value = tempF.toFixed(2);
        document.getElementById(kelvinId).value = tempK.toFixed(2);
    } else{
        document.getElementById(fahrenheitId).value = "";
        document.getElementById(kelvinId).value = "";
    }
}

function calcTempFromFarenhit(celsiusId, fahrenheitId, kelvinId, resistanceId) {
    const tempF = parseFloat(document.getElementById(fahrenheitId).value);
    if (!isNaN(tempF)) {
        const tempC = (tempF - 32) * 5 / 9;
        const tempK = tempC + 273.15;
        document.getElementById(celsiusId).value = tempC.toFixed(2);
        document.getElementById(kelvinId).value = tempK.toFixed(2);
    } else{
        document.getElementById(celsiusId).value = "";
        document.getElementById(kelvinId).value = "";
    }
}


function validateInput() {
    const resistanceCold = document.getElementById("resistanceCold").value;
    const resistanceRoom = document.getElementById("resistanceRoom").value;
    const resistanceBoiling = document.getElementById("resistanceBoiling").value;
    const coldTempC = document.getElementById("coldTempC").value;
    const roomTempC = document.getElementById("roomTempC").value;
    const boilTempC = document.getElementById("boilTempC").value;
    const submitBtn = document.getElementById("submitBtn");
}

function calcCoeffBtnClicked(event){
    event.preventDefault();
    calculateCoefficients();
}

function calculateCoefficients() {
    // const coldTempC = parseFloat(document.getElementById("coldTempC").value);
    // const roomTempC = parseFloat(document.getElementById("roomTempC").value);
    // const boilTempC = parseFloat(document.getElementById("boilTempC").value);

    // const coldTempK = coldTempC + 273.15;
    // const roomTempK = roomTempC + 273.15;
    // const boilTempK = boilTempC + 273.15;
    const T1 = parseFloat(document.getElementById("coldTempK").value);
    const T2 = parseFloat(document.getElementById("roomTempK").value);
    const T3 = parseFloat(document.getElementById("boilTempK").value);

    const resistanceCold = parseFloat(document.getElementById("resistanceCold").value);
    const resistanceRoom = parseFloat(document.getElementById("resistanceRoom").value);
    const resistanceBoiling = parseFloat(document.getElementById("resistanceBoiling").value);

    const L1 = Math.log(resistanceCold); //natural logarithm
    const L2 = Math.log(resistanceRoom);
    const L3 = Math.log(resistanceBoiling);

    const Y1 = 1 / T1;
    const Y2 = 1 / T2;
    const Y3 = 1 / T3;

    const gamma2 = (Y2 - Y1) / (L2 - L1); //γ2
    const gamma3 = (Y3 - Y1) / (L3 - L1); //γ3

    // Calculate coefficients A, B, and C
    const C = ((gamma3 - gamma2) / (L3 - L2)) * (L1 + L2 + L3) ** -1;
    const B = gamma2 - C * (Math.pow(L1, 2) + L1 * L2 + Math.pow(L2, 2));
    const A = Y1 - (B + Math.pow(L1, 2) * C) * L1;

    document.getElementById("resultA").value = A.toExponential(8);
    document.getElementById("resultB").value = B.toExponential(8);
    document.getElementById("resultC").value = C.toExponential(8);

    document.getElementById("actualA").textContent = `(${A.toFixed(16)})`;
    document.getElementById("actualB").textContent = `(${B.toFixed(16)})`;
    document.getElementById("actualC").textContent = `(${C.toFixed(16)})`;
}

function calculateTemperatureFromResistance() {

    const A = parseFloat(document.getElementById('resultA').value);
    const B = parseFloat(document.getElementById('resultB').value);
    const C = parseFloat(document.getElementById('resultC').value);

    if (isNaN(A) || isNaN(B) || isNaN(C)) {
        document.getElementById('errorMessage').style.display = 'block'; 
        document.getElementById('resultFahrenheit').value = '';
        document.getElementById('resultCelsius').value = '';
        return;
    } else{
            document.getElementById('errorMessage').style.display = 'none';
    }

    let resistance = parseFloat(document.getElementById('inputResistance').value);
    if (isNaN(resistance)) {
        alert("Please enter a valid resistance.");
        return;
    }

    // Calculate temperature in Kelvin using Steinhart-Hart equation: 
    // 1/T = A + B*ln(R) + C*(ln(R))^3
    let inverseTemperature = A + B * Math.log(resistance) + C * Math.pow(Math.log(resistance), 3);
    let temperatureKelvin = 1 / inverseTemperature; 

    // Convert to Celsius and Fahrenheit
    let temperatureCelsius = temperatureKelvin - 273.15;  
    let temperatureFahrenheit = (temperatureCelsius * 9/5) + 32;  

    document.getElementById('resultFahrenheit').value = temperatureFahrenheit.toFixed(2);
    document.getElementById('resultCelsius').value = temperatureCelsius.toFixed(2);

}
</script>

### Rust function
```rust
fn steinhart_temp_calc(
    resistance: f64, // Resistance in Ohms
    a: f64,          // Coefficient A
    b: f64,          // Coefficient B
    c: f64,          // Coefficient C
) -> Result<(f64, f64), String> {
    if resistance <= 0.0 {
        return Err("Resistance must be a positive number.".to_string());
    }

    // Calculate temperature in Kelvin using Steinhart-Hart equation:
    // 1/T = A + B*ln(R) + C*(ln(R))^3
    let ln_r = resistance.ln();
    let inverse_temperature = a + b * ln_r + c * ln_r.powi(3);

    if inverse_temperature == 0.0 {
        return Err("Invalid coefficients or resistance leading to division by zero.".to_string());
    }

    let temperature_kelvin = 1.0 / inverse_temperature;

    let temperature_celsius = temperature_kelvin - 273.15;
    let temperature_fahrenheit = (temperature_celsius * 9.0 / 5.0) + 32.0;

    Ok((temperature_celsius, temperature_fahrenheit))
}

fn main() {
    // Example inputs
     let a = 2.10850817e-3;
    let b = 7.97920473e-5;
    let c = 6.53507631e-7;
    let resistance = 10000.0;


    match steinhart_temp_calc(resistance, a, b, c) {
        Ok((celsius, fahrenheit)) => {
            println!("Temperature in Celsius: {:.2}", celsius);
            println!("Temperature in Fahrenheit: {:.2}", fahrenheit);
        }
        Err(e) => println!("Error: {}", e),
    }
}
```

### Referemce
- [Thermistor Calculator](https://www.thinksrs.com/downloads/programs/therm%20calc/ntccalibrator/ntccalculator.html) 
- [Thermistor Steinhart-Hart Coefficients for Calculating Motor Temperature](https://www.servo.jp/member/admin/document_upload/AN144-Thermistor-Steinhart-Hart-Coefficients.pdf) 
- [Calibrate Steinhart-Hart Coefficients for Thermistors](https://www.thinksrs.com/downloads/PDFs/ApplicationNotes/LDC%20Note%204%20NTC%20Calculatorold.pdf) 
- [Cooking Thermometer With Steinhart-Hart Correction](https://www.instructables.com/ESP32-NTP-Temperature-Probe-Cooking-Thermometer-Wi/)
