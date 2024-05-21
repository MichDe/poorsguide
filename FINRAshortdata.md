# ./FINRAshortdata

<input type="date" id="date-selector" />
  <table id="data-table">
    <thead>
      <tr></tr>
    </thead>
    <tbody></tbody>
  </table>
  <script>
    document.addEventListener('DOMContentLoaded', function() {
      // Function to check if a date is a weekday
      function isWeekday(date) {
        const day = date.getDay();
        return day >= 1 && day <= 5; // 1 = Monday, 5 = Friday
      }

      // Function to get the previous weekday if the current date is a weekend
      function getDefaultDate() {
        const today = new Date();
        const day = today.getDay();
        let defaultDate;

        if (day === 0) { // Sunday
          defaultDate = new Date(today);
          defaultDate.setDate(today.getDate() - 2); // Previous Friday
        } else if (day === 6) { // Saturday
          defaultDate = new Date(today);
          defaultDate.setDate(today.getDate() - 1); // Previous Friday
        } else {
          defaultDate = today; // Weekday
        }

        return defaultDate.toISOString().split('T')[0];
      }

      // Set default date
      const defaultDate = getDefaultDate();
      document.getElementById('date-selector').value = defaultDate;

      // Update URL and reload page with selected date
      document.getElementById('date-selector').addEventListener('change', function(event) {
        const selectedDate = new Date(event.target.value);
        if (isWeekday(selectedDate)) {
          const formattedDate = selectedDate.toISOString().split('T')[0].replace(/-/g, '');
          const newUrl = `${window.location.pathname}?date=${formattedDate}`;
          window.location.href = newUrl;
        } else {
          alert('Please select a weekday.');
          // Reset to default date if an invalid date is selected
          document.getElementById('date-selector').value = defaultDate;
        }
      });

      // Function to parse the URL query parameters
      function getQueryParams() {
        const params = {};
        const queryString = window.location.search.substring(1);
        const queryArray = queryString.split('&');
        queryArray.forEach(param => {
          const [key, value] = param.split('=');
          params[key] = decodeURIComponent(value);
        });
        return params;
      }

      // Fetch and display data based on the date parameter
      const params = getQueryParams();
      const selectedDate = params.date || defaultDate.replace(/-/g, '');
      if (selectedDate) {
        document.getElementById('date-selector').value = `${selectedDate.slice(0, 4)}-${selectedDate.slice(4, 6)}-${selectedDate.slice(6, 8)}`;
        const dataUrl = `https://cdn.finra.org/equity/regsho/daily/CNMSshvol${selectedDate}.txt`;

        fetch(dataUrl)
          .then(response => response.text())
          .then(data => {
            const delimiter = detectDelimiter(data);
            const rows = parseData(data, delimiter);
            const table = document.getElementById('data-table');

            // Populate headers
            let theadHTML = '';
            rows[0].forEach(header => {
              theadHTML += `<th>${header}</th>`;
            });
            table.querySelector('thead tr').innerHTML = theadHTML;

            // Populate data rows
            let tbodyHTML = '';
            rows.slice(1).forEach(row => {
              tbodyHTML += '<tr>';
              row.forEach(cell => {
                tbodyHTML += `<td>${cell}</td>`;
              });
              tbodyHTML += '</tr>';
            });
            table.querySelector('tbody').innerHTML = tbodyHTML;
          })
          .catch(error => console.error('Error fetching the data file:', error));
      }

      function detectDelimiter(data) {
        const lines = data.split('\n');
        const sampleLine = lines[0];
        if (sampleLine.indexOf(',') !== -1) return ',';
        if (sampleLine.indexOf('\t') !== -1) return '\t';
        if (sampleLine.indexOf('|') !== -1) return '|';
        return ','; // Default to CSV
      }

      function parseData(str, delimiter) {
        const arr = [];
        let quote = false;  // 'true' means we're inside a quoted field

        for (let row = 0, col = 0, c = 0; c < str.length; c++) {
          const cc = str[c], nc = str[c+1];  // Current character, next character
          arr[row] = arr[row] || [];
          arr[row][col] = arr[row][col] || '';

          if (cc === '"' && quote && nc === '"') {
            arr[row][col] += cc; ++c;
          } else if (cc === '"') {
            quote = !quote;
          } else if (cc === delimiter && !quote) {
            ++col;
          } else if (cc === '\n' && !quote) {
            ++row; col = 0;
          } else {
            arr[row][col] += cc;
          }
        }
        return arr;
      }
    });
  </script>
