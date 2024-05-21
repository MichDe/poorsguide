# ./datas 

<table id="data-table">
    <thead>
      <tr>
        <!-- Table headers will go here -->
      </tr>
    </thead>
    <tbody>
      <!-- Table data will go here -->
    </tbody>
  </table>
<script>
document.addEventListener('DOMContentLoaded', function() {
  const sheetUrl = 'https://www.nyse.com/api/trade-halts/current/download';

  fetch(sheetUrl)
    .then(response => response.text())
    .then(data => {
      const rows = parseCSV(data);
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
    .catch(error => console.error('Error fetching the Google Sheet:', error));
});

function parseCSV(str) {
  const arr = [];
  let quote = false;  // 'true' means we're inside a quoted field

  // Iterate over each character, keep track of quoted field status
  for (let row = 0, col = 0, c = 0; c < str.length; c++) {
    const cc = str[c], nc = str[c+1];  // Current character, next character
    arr[row] = arr[row] || [];
    arr[row][col] = arr[row][col] || '';

    if (cc === '"' && quote && nc === '"') {
      arr[row][col] += cc; ++c;
    } else if (cc === '"') {
      quote = !quote;
    } else if (cc === ',' && !quote) {
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
