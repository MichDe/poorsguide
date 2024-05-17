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

    const quoteApiUrl = `https://financialmodelingprep.com/api/v3/quote/${symbol}?apikey=5MI4XUXoEzLMprJ9gJXaxIXYGjLwEAW4`;
    const dividendApiUrl = `https://financialmodelingprep.com/api/v3/historical-price-full/stock_dividend/${symbol}?from=${from}&to=${to}&apikey=5MI4XUXoEzLMprJ9gJXaxIXYGjLwEAW4`;

    fetch(quoteApiUrl)
        .then(response => response.json())
        .then(quoteData => {
            fetch(dividendApiUrl)
                .then(response => response.json())
                .then(dividendData => {
                    let tableHtml = "<table>";
                    tableHtml += "<tr><th>Symbol</th><th>Price</th><th>Change (%)</th><th>Change</th><th>Year High</th><th>Year Low</th><th>Price Avg 50</th><th>Price Avg 200</th></tr>";
                    tableHtml += `<tr><td data-label="Symbol">${quoteData[0].symbol}</td><td data-label="Price">${quoteData[0].price}</td><td data-label="Change (%)">${quoteData[0].changesPercentage}</td><td data-label="Change">${quoteData[0].change}</td><td data-label="Year High">${quoteData[0].yearHigh}</td><td data-label="Year Low">${quoteData[0].yearLow}</td><td data-label="Price Avg 50">${quoteData[0].priceAvg50}</td><td data-label="Price Avg 200">${quoteData[0].priceAvg200}</td></tr>`;
                    tableHtml += "<tr><th>Regularity</th><th>Day Low</th><th>Day High</th><th>Exchange</th><th>Volume</th><th>Avg Volume</th><th>Market Cap</th><th>Shares Outstanding</th></tr>";

                    let dividendDates = [];

                    dividendData.historical.forEach(entry => {
                        dividendDates.push(new Date(entry.date));
                    });

                    function determineRegularity(dates) {
                        dates.sort((a, b) => a - b);
                        let intervals = [];
                        for (let i = 1; i < dates.length; i++) {
                            let months = (dates[i].getFullYear() - dates[i - 1].getFullYear()) * 12 + (dates[i].getMonth() - dates[i - 1].getMonth());
                            intervals.push(months);
                        }
                        let frequency = intervals.reduce((acc, interval) => {
                            acc[interval] = (acc[interval] || 0) + 1;
                            return acc;
                        }, {});
                        let mostCommonInterval = Object.keys(frequency).reduce((a, b) => frequency[a] > frequency[b] ? a : b);
                        switch (parseInt(mostCommonInterval)) {
                            case 1: return "Monthly";
                            case 3: return "Quarterly";
                            case 6: return "Semi-Annual";
                            case 12: return "Annual";
                            default: return "Irregular";
                        }
                    }

                    let regularity = determineRegularity(dividendDates);

                    tableHtml += `<tr><td data-label="Regularity">${regularity}</td><td data-label="Day Low">${quoteData[0].dayLow}</td><td data-label="Day High">${quoteData[0].dayHigh}</td><td data-label="Exchange">${quoteData[0].exchange}</td><td data-label="Volume">${quoteData[0].volume}</td><td data-label="Avg Volume">${quoteData[0].avgVolume}</td><td data-label="Market Cap">${quoteData[0].marketCap}</td><td data-label="Shares Outstanding">${quoteData[0].sharesOutstanding}</td></tr>`;

                    tableHtml += "<tr><th>Ex Date</th><th>Pay Date</th><th>Declaration Date</th><th>Record Date</th><th>Amount</th><th>Annual Yield (%)</th><th>Monthly Yield (%)</th><th>Estimated Annual Amount</th></tr>";
                    dividendData.historical.forEach(entry => {
                        const annualYield = (entry.dividend * 12 / quoteData[0].price) * 100;
                        const monthlyYield = annualYield / 12;
                        const estimatedAnnualAmount = (entry.dividend * 12 / quoteData[0].price) * quoteData[0].price;

                        tableHtml += `<tr><td data-label="Ex Date">${entry.date}</td><td data-label="Pay Date">${entry.paymentDate}</td><td data-label="Declaration Date">${entry.declarationDate}</td><td data-label="Record Date">${entry.recordDate}</td><td data-label="Amount">${entry.dividend}</td><td data-label="Annual Yield (%)">${annualYield.toFixed(2)}</td><td data-label="Monthly Yield (%)">${monthlyYield.toFixed(2)}</td><td data-label="Estimated Annual Amount">${estimatedAnnualAmount.toFixed(2)}</td></tr>`;
                    });

                    tableHtml += "</table>";
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
