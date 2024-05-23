# ./FINRAshortdata

  <input type="date" id="date-selector" />
  <select id="file-type-selector">
    <option value="CNMSshvol" selected>Consolidated</option>
    <option value="FNSQshvol">NASDAQ Carteret</option>
    <option value="FNQCshvol">NASDAQ Chicago</option>
    <option value="FNYXshvol">NYSE</option>
    <option value="FNRAshvol">ADF</option>
    <option value="FORFshvol">ORF</option>
  </select>
  <table id="data-table" class="display">
    <thead>
      <tr></tr>
    </thead>
    <tbody></tbody>
  </table>

  <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
  <script src="https://cdn.datatables.net/1.11.3/js/jquery.dataTables.min.js"></script>
  <script>
    $(document).ready(function() {
      function isWeekday(date) {
        const day = date.getDay();
        return day >= 1 && day <= 5;
      }

      function getDefaultDate() {
        const today = new Date();
        let defaultDate = new Date(today);
        defaultDate.setDate(today.getDate() - 1);
        while (!isWeekday(defaultDate)) {
          defaultDate.setDate(defaultDate.getDate() - 1);
        }
        return defaultDate.toISOString().split('T')[0];
      }

      const defaultDate = getDefaultDate();
      $('#date-selector').val(defaultDate);

      function updateUrlAndReload() {
        const selectedDate = new Date($('#date-selector').val());
        const selectedFileType = $('#file-type-selector').val();

        if (isWeekday(selectedDate)) {
          const formattedDate = selectedDate.toISOString().split('T')[0].replace(/-/g, '');
          const newUrl = `${window.location.pathname}?date=${formattedDate}&filetype=${selectedFileType}`;
          window.location.href = newUrl;
        } else {
          alert('Please select a weekday.');
          $('#date-selector').val(defaultDate);
        }
      }

      $('#date-selector').change(updateUrlAndReload);
      $('#file-type-selector').change(updateUrlAndReload);

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

      const params = getQueryParams();
      const selectedDate = params.date || defaultDate.replace(/-/g, '');
      const selectedFileType = params.filetype || 'CNMSshvol';
      $('#date-selector').val(`${selectedDate.slice(0, 4)}-${selectedDate.slice(4, 6)}-${selectedDate.slice(6, 8)}`);
      $('#file-type-selector').val(selectedFileType);
      
      const dataUrl = `https://cdn.finra.org/equity/regsho/daily/${selectedFileType}${selectedDate}.txt`;

      function detectDelimiter(data) {
        const lines = data.split('\n');
        const sampleLine = lines[0];
        if (sampleLine.indexOf(',') !== -1) return ',';
        if (sampleLine.indexOf('\t') !== -1) return '\t';
        if (sampleLine.indexOf('|') !== -1) return '|';
        return ','; 
      }

      function parseData(str, delimiter) {
        const arr = [];
        let quote = false;

        for (let row = 0, col = 0, c = 0; c < str.length; c++) {
          const cc = str[c], nc = str[c + 1];
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

      fetch(dataUrl)
        .then(response => {
          if (!response.ok) throw new Error('No data available for the selected date');
          return response.text();
        })
        .then(data => {
          const delimiter = detectDelimiter(data);
          const rows = parseData(data, delimiter);
          const table = $('#data-table');

          // Populate headers
          let theadHTML = '';
          rows[0].forEach(header => {
            theadHTML += `<th>${header}</th>`;
          });
          table.find('thead tr').html(theadHTML);

          // Initialize DataTables with server-side processing
          table.DataTable({
            data: rows.slice(1),
            columns: rows[0].map(header => ({ title: header })),
            deferRender: true,
            scrollY: 400,
            scrollCollapse: true,
            scroller: true
          });
        })
        .catch(error => {
          console.error('Error fetching the data file:', error);
          alert('No data available for the selected date');
        });
    });
    $.fn.dataTable.ext.errMode = 'none';
  </script>
