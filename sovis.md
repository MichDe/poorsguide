
  <link rel="stylesheet" href="https://cdn.datatables.net/1.11.3/css/jquery.dataTables.min.css">
  <label for="start-date-selector">Start Date:</label>
  <input type="date" id="start-date-selector" />
  
  <label for="end-date-selector">End Date:</label>
  <input type="date" id="end-date-selector" />
  
  <select id="file-type-selector">
    <option value="CNMSshvol" selected>Consolidated</option>
    <option value="FNSQshvol">NASDAQ Carteret</option>
    <option value="FNQCshvol">NASDAQ Chicago</option>
    <option value="FNYXshvol">NYSE</option>
    <option value="FNRAshvol">ADF</option>
    <option value="FORFshvol">ORF</option>
    <!-- Add more options as needed -->
  </select>
  <table id="data-table" class="display">
    <thead>
      <tr>
        <th>Date</th>
        <th>Symbol</th>
        <th>Short</th>
        <th>Sv%</th>
        <th>SvR</th>
        <th>Exempt</th>
        <th>Ev%</th>
        <th>EvR</th>
        <th>Total</th>
        <th>Ov%</th>
        <th>OvR</th>
        <th>Market</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
  <script src="https://cdn.datatables.net/1.11.3/js/jquery.dataTables.min.js"></script>
  <script>
    $(document).ready(function() {
      function getDefaultDate() {
        const today = new Date();
        let defaultDate = new Date(today);
        defaultDate.setDate(today.getDate() - 1);

        // Ensure the default date is a weekday (Mon-Fri)
        while (defaultDate.getDay() === 0 || defaultDate.getDay() === 6) { // Skip Sunday and Saturday
          defaultDate.setDate(defaultDate.getDate() - 1);
        }

        return defaultDate.toISOString().split('T')[0];
      }

      const defaultDate = getDefaultDate();
      $('#start-date-selector').val(defaultDate);
      $('#end-date-selector').val(defaultDate);
      $.fn.dataTable.ext.errMode = 'none';

      function updateUrlAndReload() {
        const startDate = new Date($('#start-date-selector').val());
        const endDate = new Date($('#end-date-selector').val());
        const selectedFileType = $('#file-type-selector').val();

        const formattedStartDate = startDate.toISOString().split('T')[0].replace(/-/g, '');
        const formattedEndDate = endDate.toISOString().split('T')[0].replace(/-/g, '');
        const newUrl = `${window.location.pathname}?startDate=${formattedStartDate}&endDate=${formattedEndDate}&filetype=${selectedFileType}`;
        window.location.href = newUrl;
      }

      $('#start-date-selector').change(updateUrlAndReload);
      $('#end-date-selector').change(updateUrlAndReload);
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

      function dateRange(startDate, endDate) {
        const dates = [];
        let currentDate = new Date(startDate);
        while (currentDate <= endDate) {
          dates.push(new Date(currentDate));
          currentDate.setDate(currentDate.getDate() + 1);
        }
        return dates;
      }

      function fetchDataAndCombine(dates, fileType) {
        const fetchPromises = dates.map(date => {
          const formattedDate = date.toISOString().split('T')[0].replace(/-/g, '');
          const dataUrl = `https://cdn.finra.org/equity/regsho/daily/${fileType}${formattedDate}.txt`;
          return fetch(dataUrl).then(response => {
            if (!response.ok) throw new Error(`No data for ${formattedDate}`);
            return response.text();
          }).catch(error => {
            console.error(error);
            return null;
          });
        });

        return Promise.all(fetchPromises).then(dataArray => {
          const validData = dataArray.filter(data => data !== null);
          if (validData.length === 0) throw new Error('No data available for the selected date range');

          const delimiter = detectDelimiter(validData[0]);
          const allRows = validData.flatMap(data => parseData(data, delimiter));
          const headers = allRows[0];
          const combinedRows = allRows.slice(1);
          return { headers, combinedRows };
        });
      }

      const params = getQueryParams();
      const startDate = params.startDate || defaultDate.replace(/-/g, '');
      const endDate = params.endDate || defaultDate.replace(/-/g, '');
      const selectedFileType = params.filetype || 'CNMSshvol';
      $('#start-date-selector').val(`${startDate.slice(0, 4)}-${startDate.slice(4, 6)}-${startDate.slice(6, 8)}`);
      $('#end-date-selector').val(`${endDate.slice(0, 4)}-${endDate.slice(4, 6)}-${endDate.slice(6, 8)}`);
      $('#file-type-selector').val(selectedFileType);

      const dateRangeArray = dateRange(new Date(startDate.slice(0, 4), startDate.slice(4, 6) - 1, startDate.slice(6, 8)), new Date(endDate.slice(0, 4), endDate.slice(4, 6) - 1, endDate.slice(6, 8)));
      
      fetchDataAndCombine(dateRangeArray, selectedFileType)
        .then(({ headers, combinedRows }) => {
          // Rename headers and calculate new columns
          const renamedHeaders = ['Date', 'Symbol', 'Short', 'Sv%', 'SvR', 'Exempt', 'Ev%', 'EvR', 'Total', 'Ov%', 'OvR', 'Market'];
          let priorShortVolume = null;
          let priorExemptVolume = null;

          const processedRows = combinedRows.map((row, index) => {
            const shortVolume = parseFloat(row[2]);
            const shortExemptVolume = parseFloat(row[3]);
            const totalVolume = parseFloat(row[4]);

            const svPercent = ((shortVolume / totalVolume) * 100).toFixed(2);
            const evPercent = ((shortExemptVolume / totalVolume) * 100).toFixed(2);
            const ovPercent = ((shortExemptVolume / shortVolume) * 100).toFixed(2);

            let svRate = 'N/A';
            let evRate = 'N/A';
            let ovRate = 'N/A';

            if (priorShortVolume !== null && priorExemptVolume !== null) {
              svRate = (((shortVolume / priorShortVolume) * 100) - 100).toFixed(2);
              evRate = (((shortExemptVolume / priorExemptVolume) * 100) - 100).toFixed(2);
              ovRate = ((((shortExemptVolume / shortVolume) / (priorExemptVolume / priorShortVolume)) * 100) - 100).toFixed(2);
            }

            priorShortVolume = shortVolume;
            priorExemptVolume = shortExemptVolume;

            return [
              row[0], // Date
              row[1], // Symbol
              shortVolume, // Short
              svPercent, // Sv%
              svRate, // SvR
              shortExemptVolume, // Exempt
              evPercent, // Ev%
              evRate, // EvR
              totalVolume, // Total
              ovPercent, // Ov%
              ovRate, // OvR
              row[5] // Market
            ];
          });

          const table = $('#data-table');

          // Populate headers
          let theadHTML = '';
          renamedHeaders.forEach(header => {
            theadHTML += `<th>${header}</th>`;
          });
          table.find('thead tr').html(theadHTML);

          // Initialize DataTables
          table.DataTable({
            data: processedRows,
            columns: renamedHeaders.map(header => ({ title: header })),
            deferRender: true,
            scrollY: 700,
            scrollCollapse: true,
            scroller: true,
            pageLength: 20,
          });
        })
        .catch(error => {
          console.error('Error fetching the data:', error);
          alert('No data available for the selected date range');
        });
    });

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
  </script>
