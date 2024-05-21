# ./FINRAshortdata

<table id="data-table">
    <thead>
      <tr></tr>
    </thead>
    <tbody></tbody>
  </table>
  <script>
    document.addEventListener('DOMContentLoaded', function() {
      const dataUrl = 'https://cdn.finra.org/equity/regsho/daily/CNMSshvol20240520.txt'; // Update with your data file URL

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
    });

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
  </script>
