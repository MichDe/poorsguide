# @PoorsGuide ./Top100YieldETFs 

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
      const sheetUrl = 'https://docs.google.com/spreadsheets/d/e/2PACX-1vSm02-ccjaMk1wVhrlnlO5z9_3iavHhVB87VzO7NTCtlfhHfkDXyuLOmVLbgWOkggQlU59Mg8jdQjOj/pub?gid=1154281212&single=true&output=csv';

      fetch(sheetUrl)
        .then(response => response.text())
        .then(data => {
          const rows = data.split('\n');
          const table = document.getElementById('data-table');
          const headers = rows[0].split(',');

          // Populate headers
          let theadHTML = '';
          headers.forEach(header => {
            theadHTML += `<th>${header}</th>`;
          });
          table.querySelector('thead tr').innerHTML = theadHTML;

          // Populate data rows
          let tbodyHTML = '';
          rows.slice(1).forEach(row => {
            if (row) {
              const cells = row.split(',');
              tbodyHTML += '<tr>';
              cells.forEach(cell => {
                tbodyHTML += `<td>${cell}</td>`;
              });
              tbodyHTML += '</tr>';
            }
          });
          table.querySelector('tbody').innerHTML = tbodyHTML;
        })
        .catch(error => console.error('Error fetching the Google Sheet:', error));
    });
  </script>
