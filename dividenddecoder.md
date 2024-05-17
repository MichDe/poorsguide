
<h2>Quote and Dividend Data</h2>
<form id="dataForm">
    <label for="symbol">Symbol:</label>
    <input type="text" id="symbol" name="symbol" required>
    <label for="from">From:</label>
    <input type="date" id="from" name="from" required>
    <label for="to">To:</label>
    <input type="date" id="to" name="to" required>
    <input type="submit" value="Get Data">
</form>
<br>
<div id="dataTable"></div>

<script>
document.getElementById("dataForm").addEventListener("submit", function(event) {
    event.preventDefault();
    const symbol = document.getElementById("symbol").value;
    const from = document.getElementById("from").value;
    const to = document.getElementById("to").value;

    // API call for quote data
    const quoteApiUrl = `https://financialmodelingprep.com/api/v3/quote/${symbol}?apikey=5MI4XUXoEzLMprJ9gJXaxIXYGjLwEAW4`;

    // API call for dividend data
    const dividendApiUrl = `https://financialmodelingprep.com/api/v3/historical-price-full/stock_dividend/${symbol}?from=${from}&to=${to}&apikey=5MI4XUXoEzLMprJ9gJXaxIXYGjLwEAW4`;

    // Fetch quote data
    fetch(quoteApiUrl)
        .then(response => response.json())
        .then(quoteData => {
            // Fetch dividend data
            fetch(dividendApiUrl)
                .then(response => response.json())
                .then(dividendData => {
                    // Construct the table
                    let tableHtml = "<table>";
                    // First row for quote data
                    tableHtml += "<tr><th>Symbol</th><th>Price</th><th>Change (%)</th><th>Change</th><th>Year High</th><th>Year Low</th><th>Price Avg 50</th><th>Price Avg 200</th></tr>";
                    tableHtml += `<tr><td>${quoteData[0].symbol}</td><td>${quoteData[0].price}</td><td>${quoteData[0].changesPercentage}</td><td>${quoteData[0].change}</td><td>${quoteData[0].yearHigh}</td><td>${quoteData[0].yearLow}</td><td>${quoteData[0].priceAvg50}</td><td>${quoteData[0].priceAvg200}</td></tr>`;
                    // Second row for additional quote data
                    tableHtml += "<tr><th>Name</th><th>Day Low</th><th>Day High</th><th>Exchange</th><th>Volume</th><th>Avg Volume</th><th>Market Cap</th><th>Shares Outstanding</th></tr>";
                    tableHtml += `<tr><td>${quoteData[0].name}</td><td>${quoteData[0].dayLow}</td><td>${quoteData[0].dayHigh}</td><td>${quoteData[0].exchange}</td><td>${quoteData[0].volume}</td><td>${quoteData[0].avgVolume}</td><td>${quoteData[0].marketCap}</td><td>${quoteData[0].sharesOutstanding}</td></tr>`;
                    // Third row for averaged dividend data
                    tableHtml += "<tr><th>Avg Ex Date</th><th>Avg Pay Date</th><th>Avg Declaration Date</th><th>Avg Record Date</th><th>Avg Amount</th><th>Avg Annual Yield (%)</th><th>Avg Monthly Yield (%)</th><th>Avg Estimated Annual Amount</th></tr>";
                    // Initialize variables for calculating averages
                    let totalAmount = 0;
                    let totalAnnualYield = 0;
                    let totalMonthlyYield = 0;
                    let totalDays = 0;
                    let rowCount = 0;
                    // Iterate through dividend data to calculate totals
                    dividendData.historical.forEach(entry => {
                        const annualYield = (entry.dividend * 12 / quoteData[0].price) * 100;
                        const monthlyYield = annualYield / 12;
                        const estimatedAnnualAmount = (entry.dividend * 12 / quoteData[0].price) * quoteData[0].price;
                        // Add to totals
                        totalAmount += entry.dividend;
                        totalAnnualYield += annualYield;
                        totalMonthlyYield += monthlyYield;
                        const dateParts = entry.date.split("-");
                        totalDays += parseInt(dateParts[2]); // Add day number to total days
                        rowCount++;
                    });
                    // Calculate averages
                    const avgAmount = totalAmount / rowCount;
                    const avgAnnualYield = totalAnnualYield / rowCount;
                    const avgMonthlyYield = totalMonthlyYield / rowCount;
                    const avgDay = Math.round(totalDays / rowCount); // Calculate average day
                    const avgEstimatedAnnualAmount = totalAmount / rowCount * 12; // Calculate average estimated annual amount
                    // Append averaged data row to the table
                    tableHtml += `<tr><td>${avgDay}</td><td>${avgDay}</td><td>${avgDay}</td><td>${avgDay}</td><td>${avgAmount.toFixed(2)}</td><td>${avgAnnualYield.toFixed(2)}</td><td>${avgMonthlyYield.toFixed(2)}</td><td>${avgEstimatedAnnualAmount.toFixed(2)}</td></tr>`;
                    // Fourth row for dividend data
                    tableHtml += "<tr><th>Ex Date</th><th>Pay Date</th><th>Declaration Date</th><th>Record Date</th><th>Amount</th><th>Annual Yield (%)</th><th>Monthly Yield (%)</th><th>Estimated Annual Amount</th></tr>";
                    dividendData.historical.forEach(entry => {
                        const annualYield = (entry.dividend * 12 / quoteData[0].price) * 100;
                        const monthlyYield = annualYield / 12;
                        const estimatedAnnualAmount = (entry.dividend * 12 / quoteData[0].price) * quoteData[0].price;
                        // Append dividend data to the table
                        tableHtml += `<tr><td>${entry.date}</td><td>${entry.paymentDate}</td><td>${entry.declarationDate}</td><td>${entry.recordDate}</td><td>${entry.dividend}</td><td>${annualYield.toFixed(2)}</td><td>${monthlyYield.toFixed(2)}</td><td>${estimatedAnnualAmount.toFixed(2)}</td></tr>`;
                    });
                    // Close the table
                    tableHtml += "</table>";
                    // Display the table
                    document.getElementById("dataTable").innerHTML = tableHtml;
                })
                .catch(error => {
                    console.error('Error fetching dividend data:', error);
                    document.getElementById("dataTable").innerHTML = "An error occurred while fetching dividend data.";
                });
        })
        .catch(error => {
            console.error('Error fetching quote data:', error);
            document.getElementById("dataTable").innerHTML = "An error occurred while fetching quote data.";
        });
});
</script>
