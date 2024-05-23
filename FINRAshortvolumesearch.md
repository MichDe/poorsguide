
  <div style="padding: 20px;">
    <label for="start-date-selector">Start Date:</label>
    <input type="date" id="start-date-selector" class="form-control" />
  
    <label for="end-date-selector">End Date:</label>
    <input type="date" id="end-date-selector" class="form-control" />
  
    <label for="file-type-selector">File Type:</label>
    <select id="file-type-selector" class="form-control">
      <option value="CNMSshvol" selected>Consolidated</option>
      <option value="FNSQshvol">NASDAQ Carteret</option>
      <option value="FNQCshvol">NASDAQ Chicago</option>
      <option value="FNYXshvol">NYSE</option>
      <option value="FNRAshvol">ADF</option>
      <option value="FORFshvol">ORF</option>
      <!-- Add more options as needed -->
    </select>

    <table id="data-table" class="display" style="width: 100%; margin-top: 20px;">
      <thead>
        <tr></tr>
      </thead>
      <tbody></tbody>
    </table>
  </div>

  <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
  <script src="https://cdn.datatables.net/1.11.3/js/jquery.dataTables.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/bootstrap-datepicker/1.9.0/js/bootstrap-datepicker.min.js"></script>
  <script>
    $(document).ready(function() {
      $('body').toggleClass('dark');

      function getFirstDayOfWeek() {
        const firstDay = new Date().toLocaleDateString(undefined, { weekday: 'long' });
        const weekdays = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
        return weekdays.indexOf(firstDay);
      }

      function isWeekday(date) {
        const firstDayOfWeek = getFirstDayOfWeek();
        const day = (date.getDay() - firstDayOfWeek + 6) % 7;
        return day >= 2 && day <= 7;
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
      $('#start-date-selector').val(defaultDate);
      $('#end-date-selector').val(defaultDate);
      $.fn.dataTable.ext.errMode = 'none';

      function updateUrlAndReload() {
        const startDate = new Date($('#start-date-selector').val());
        const endDate = new Date($('#end-date-selector').val());
        const selectedFileType = $('#file-type-selector').val();

        if (isWeekday(startDate) && isWeekday(endDate)) {
          const formattedStartDate = startDate.toISOString().split('T')[0].replace(/-/g, '');
          const formattedEndDate = endDate.toISOString().split('T')[0].replace(/-/g, '');
          const newUrl = `${window.location.pathname}?startDate=${formattedStartDate}&endDate=${formattedEndDate}&filetype=${selectedFileType}`;
          window.location.href = newUrl;
        } else {
          alert('Please select weekdays.');
          $('#start-date-selector').val(defaultDate);
          $('#end-date-selector').val(defaultDate);
        }
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
          if (isWeekday(currentDate)) {
            dates.push(new Date(currentDate));
          }
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
          const allRows = validData.flatMap((data, index) => parseData(data, delimiter, index === 0));
          const headers = allRows[0];
          const combinedRows = allRows.slice(1).map(row => row.map(cell => {
            if (!isNaN(cell)) return parseFloat(cell);
            return cell;
          }));
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
          const table = $('#data-table');

          // Populate headers
          let theadHTML = '';
          headers.forEach(header => {
            theadHTML += `<th>${header}</th>`;
          });
          table.find('thead tr').html(theadHTML);

          // Initialize DataTables with server-side processing
          table.DataTable({
            data: combinedRows,
            columns: headers.map(header => ({ title: header })),
            deferRender: true,
            scrollY: 720,
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

    function parseData(str, delimiter, isFirstFile) {
      const arr = [];
      let quote = false;
      let skipFirstLine = !isFirstFile; // Skip the header line for all but the first file

      for (let row = 0, col = 0, c = 0; c < str.length; c++) {
        const cc = str[c], nc = str[c + 1];
        if (skipFirstLine && cc === '\n') {
          skipFirstLine = false;
          continue;
        }

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
  
