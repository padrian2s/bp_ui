<script setup>
import { ref, onMounted, computed } from 'vue';
import { Line, Bubble } from 'vue-chartjs';
import { Chart as ChartJS, Title, Tooltip, Legend, LineElement, LinearScale, PointElement, CategoryScale, TimeScale, BubbleController } from 'chart.js';
import 'chart.js/auto';
import annotationPlugin from 'chartjs-plugin-annotation';
import DataLabelsPlugin from 'chartjs-plugin-datalabels';

// Register Chart.js components
ChartJS.register(Title, Tooltip, Legend, LineElement, LinearScale, PointElement, CategoryScale, TimeScale, BubbleController, annotationPlugin, DataLabelsPlugin);

// Data for the graph
const bpData = ref([]);
const chartData = ref(null);
const chartOptions = ref(null);
const loading = ref(true);
const error = ref(null);
const avgSystolic = ref(0);
const avgDiastolic = ref(0);
const intervalDays = ref(3); // Default interval of 3 days for moving averages
const avgSystolicByInterval = ref([]);
const avgDiastolicByInterval = ref([]);
const lineThickness = ref(6); // Default line thickness for average lines
const avgSystolicColor = ref('#888888'); // Default gray color for systolic average line
const avgDiastolicColor = ref('#888888'); // Default gray color for diastolic average line
const measurementStartDate = ref(null); // Start date of the measurement period
const measurementEndDate = ref(null); // End date of the measurement period
const controlsCollapsed = ref(false); // Track if controls are collapsed

// CSV file selection
const availableCsvFiles = ref([
  { name: 'BP_dump.csv', path: '/BP_dump.csv', label: 'BP Dump (Default)' },
  { name: 'Your Requested OMRON Report from 21 Nov 2024 to 21 May 2025.csv', path: '/Your Requested OMRON Report from 21 Nov 2024 to 21 May 2025.csv', label: 'OMRON Report (Nov 2024 - May 2025)' },
  { name: 'Your Requested OMRON Report from 29 Jun 2023 to 31 Dec 2023.csv', path: '/Your Requested OMRON Report from 29 Jun 2023 to 31 Dec 2023.csv', label: 'OMRON Report (Jun 2023 - Dec 2023)' }
]);
const selectedCsvFile = ref(availableCsvFiles.value[0]); // Default to the first file
const uploadedCsvFile = ref(null); // For user-uploaded CSV files

// Data for filtered graphs
const highBPData = ref([]); // Data for systolic >= 140 mmHg
const mediumBPData = ref([]); // Data for systolic between 130-140 mmHg
const nocturnalData = ref([]); // Data for nocturnal measurements
const highBPChartData = ref(null);
const mediumBPChartData = ref(null);
const nocturnalChartData = ref(null);
const highBPChartOptions = ref(null);
const mediumBPChartOptions = ref(null);
const nocturnalChartOptions = ref(null);

// Data for heat map
const heatMapData = ref(null);
const heatMapOptions = ref(null);

// Blood pressure statistics
const bpStats = ref({
  systolic: {
    thresholds: [140, 130, 120, 110, 100, 90],
    counts: {},
    percentages: {}
  },
  diastolic: {
    thresholds: [90, 85, 80, 75, 70, 65],
    counts: {},
    percentages: {}
  }
});

// Blood pressure level thresholds
const bpLevels = {
  systolic: [
    { value: 180, label: 'Hypertensive Crisis', color: '#FF0000' },
    { value: 140, label: 'Hypertension Stage 2', color: '#FF6666' },
    { value: 130, label: 'Hypertension Stage 1', color: '#FFCC00' },
    { value: 120, label: 'Elevated', color: '#FFFF66' },
    { value: 90, label: 'Normal', color: '#66FF66' },
    { value: 0, label: 'Low', color: '#66CCFF' }
  ],
  diastolic: [
    { value: 120, label: 'Hypertensive Crisis', color: '#FF0000' },
    { value: 90, label: 'Hypertension Stage 2', color: '#FF6666' },
    { value: 80, label: 'Hypertension Stage 1', color: '#FFCC00' },
    { value: 80, label: 'Elevated', color: '#FFFF66' },
    { value: 60, label: 'Normal', color: '#66FF66' },
    { value: 0, label: 'Low', color: '#66CCFF' }
  ]
};

// Load data from CSV
const loadData = async () => {
  try {
    loading.value = true;

    let csvText;

    // Check if we're using an uploaded file or a predefined one
    if (uploadedCsvFile.value) {
      // Read the uploaded file
      csvText = await readUploadedFile(uploadedCsvFile.value);
      console.log('Using uploaded CSV file:', uploadedCsvFile.value.name);
    } else {
      // Get the base URL of the application
      const baseUrl = window.location.origin;
      // Encode the file path to handle spaces and special characters
      const encodedPath = selectedCsvFile.value.path.split('/').map(segment =>
        segment ? encodeURIComponent(segment) : ''
      ).join('/');
      const csvUrl = `${baseUrl}${encodedPath}`;
      console.log('Fetching CSV data from:', csvUrl);

      const response = await fetch(csvUrl);
      if (!response.ok) {
        throw new Error(`HTTP error! Status: ${response.status}`);
      }

      csvText = await response.text();
    }

    // Function to read an uploaded file
    async function readUploadedFile(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => resolve(e.target.result);
        reader.onerror = (e) => reject(new Error('Error reading file'));
        reader.readAsText(file);
      });
    }
    if (!csvText || csvText.trim() === '') {
      throw new Error('CSV file is empty');
    }

    // Parse CSV
    const lines = csvText.split('\n');
    if (lines.length < 2) {
      throw new Error('CSV file has insufficient data (less than 2 lines)');
    }

    const headers = lines[0].split(',');

    // Check if required headers exist
    const requiredHeaders = ['Date', 'Time', 'Systolic (mmHg)', 'Diastolic (mmHg)', 'Pulse (bpm)'];
    for (const header of requiredHeaders) {
      if (!headers.some(h => h.trim() === header)) {
        throw new Error(`Required header "${header}" not found in CSV`);
      }
    }

    const parsedData = [];
    for (let i = 1; i < lines.length; i++) {
      if (!lines[i].trim()) continue;

      const values = lines[i].split(',');
      const entry = {};

      headers.forEach((header, index) => {
        entry[header.trim()] = values[index]?.trim() || '';
      });

      // Skip entries with invalid data
      if (entry['Systolic (mmHg)'] === '-2147483648' ||
          entry['Diastolic (mmHg)'] === '-2147483648' ||
          entry['Pulse (bpm)'] === '-2147483648') {
        continue;
      }

      // Convert string values to numbers
      entry['Systolic (mmHg)'] = parseInt(entry['Systolic (mmHg)'], 10);
      entry['Diastolic (mmHg)'] = parseInt(entry['Diastolic (mmHg)'], 10);
      entry['Pulse (bpm)'] = parseInt(entry['Pulse (bpm)'], 10);

      // Create a date object for sorting
      try {
        const dateParts = entry['Date'].split(' ');
        const day = parseInt(dateParts[0], 10);
        const month = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'].indexOf(dateParts[1]);
        const year = parseInt(dateParts[2], 10);

        const timeParts = entry['Time'].split(':');
        const hour = parseInt(timeParts[0], 10);
        const minute = parseInt(timeParts[1], 10);

        // Create a date in the local timezone first
        const localDate = new Date(year, month, day, hour, minute);

        // Convert to Bucharest timezone (UTC+2 or UTC+3 depending on DST)
        // First, create a UTC date by subtracting the local timezone offset
        const utcDate = new Date(localDate.getTime() - localDate.getTimezoneOffset() * 60000);

        // Then add the Bucharest timezone offset (UTC+2 or UTC+3)
        const bucharestOffset = isRomanianDST(localDate) ? 3 : 2; // Hours
        const bucharestDate = new Date(utcDate.getTime() + bucharestOffset * 3600000);

        // Log the timezone conversion for debugging
        console.log(`Date conversion: ${entry['Date']} ${entry['Time']} (local: ${localDate.getHours()}:${localDate.getMinutes()}) -> Bucharest: ${bucharestDate.getHours()}:${bucharestDate.getMinutes()} (offset: ${bucharestOffset}h, DST: ${isRomanianDST(localDate)})`);

        entry.dateObj = bucharestDate;
        entry.formattedDate = `${day}/${month+1} ${bucharestDate.getHours().toString().padStart(2, '0')}:${bucharestDate.getMinutes().toString().padStart(2, '0')}`;

        // Skip entries with invalid dates
        if (isNaN(entry.dateObj.getTime())) {
          console.warn(`Skipping entry with invalid date: ${entry['Date']} ${entry['Time']}`);
          continue;
        }

        parsedData.push(entry);
      } catch (dateErr) {
        console.warn(`Error parsing date for entry at line ${i+1}:`, dateErr);
        continue;
      }
    }

    if (parsedData.length === 0) {
      throw new Error('No valid data points found in CSV');
    }

    // Sort by date (oldest first)
    parsedData.sort((a, b) => a.dateObj - b.dateObj);

    // Store all data points
    bpData.value = parsedData;
    console.log('Total data points to display:', bpData.value.length);

    // Calculate measurement period (start and end dates)
    if (bpData.value.length > 0) {
      measurementStartDate.value = bpData.value[0].dateObj;
      measurementEndDate.value = bpData.value[bpData.value.length - 1].dateObj;
      console.log('Measurement period:',
        measurementStartDate.value.toLocaleDateString(),
        'to',
        measurementEndDate.value.toLocaleDateString());
    }

    // Filter data for separate graphs
    filterDataForGraphs();

    // Calculate blood pressure statistics
    calculateBPStats();

    // Calculate average values
    calculateAverages();

    // Prepare data for Chart.js
    prepareChartData();
    prepareHighBPChartData();
    prepareMediumBPChartData();
    prepareNocturnalChartData();
    prepareHeatMapData();

    loading.value = false;
  } catch (err) {
    console.error('Error loading or processing data:', err);
    error.value = 'Error loading data: ' + err.message;
    loading.value = false;
  }
};

// Calculate blood pressure statistics
const calculateBPStats = () => {
  if (bpData.value.length === 0) return;

  const totalReadings = bpData.value.length;

  // Calculate statistics for systolic readings
  const systolicCounts = {};
  const systolicPercentages = {};
  const systolicThresholds = bpStats.value.systolic.thresholds;

  // Calculate counts for systolic readings between thresholds
  for (let i = 0; i < systolicThresholds.length; i++) {
    const currentThreshold = systolicThresholds[i];
    const nextThreshold = i < systolicThresholds.length - 1 ? systolicThresholds[i + 1] : 0;

    // For the highest threshold, count readings >= that threshold
    if (i === 0) {
      const count = bpData.value.filter(d => d['Systolic (mmHg)'] >= currentThreshold).length;
      systolicCounts[currentThreshold] = count;
    }
    // For other thresholds, count readings between this threshold and the next higher one
    else {
      const previousThreshold = systolicThresholds[i - 1];
      const count = bpData.value.filter(
        d => d['Systolic (mmHg)'] >= currentThreshold && d['Systolic (mmHg)'] < previousThreshold
      ).length;
      systolicCounts[currentThreshold] = count;
    }

    systolicPercentages[currentThreshold] = (systolicCounts[currentThreshold] / totalReadings * 100).toFixed(1);
  }

  bpStats.value.systolic.counts = systolicCounts;
  bpStats.value.systolic.percentages = systolicPercentages;

  // Log systolic statistics for verification
  console.log('Systolic statistics:');
  systolicThresholds.forEach(threshold => {
    console.log(`Threshold ${threshold}: ${systolicCounts[threshold]} measurements`);
  });

  // Calculate statistics for diastolic readings
  const diastolicCounts = {};
  const diastolicPercentages = {};
  const diastolicThresholds = bpStats.value.diastolic.thresholds;

  // Calculate counts for diastolic readings between thresholds
  for (let i = 0; i < diastolicThresholds.length; i++) {
    const currentThreshold = diastolicThresholds[i];
    const nextThreshold = i < diastolicThresholds.length - 1 ? diastolicThresholds[i + 1] : 0;

    // For the highest threshold, count readings >= that threshold
    if (i === 0) {
      const count = bpData.value.filter(d => d['Diastolic (mmHg)'] >= currentThreshold).length;
      diastolicCounts[currentThreshold] = count;
    }
    // For other thresholds, count readings between this threshold and the next higher one
    else {
      const previousThreshold = diastolicThresholds[i - 1];
      const count = bpData.value.filter(
        d => d['Diastolic (mmHg)'] >= currentThreshold && d['Diastolic (mmHg)'] < previousThreshold
      ).length;
      diastolicCounts[currentThreshold] = count;
    }

    diastolicPercentages[currentThreshold] = (diastolicCounts[currentThreshold] / totalReadings * 100).toFixed(1);
  }

  bpStats.value.diastolic.counts = diastolicCounts;
  bpStats.value.diastolic.percentages = diastolicPercentages;

  // Log diastolic statistics for verification
  console.log('Diastolic statistics:');
  diastolicThresholds.forEach(threshold => {
    console.log(`Threshold ${threshold}: ${diastolicCounts[threshold]} measurements`);
  });
};

// Filter data for separate graphs based on systolic pressure
const filterDataForGraphs = () => {
  if (bpData.value.length === 0) return;

  // Filter data for high BP (systolic >= 140 mmHg)
  highBPData.value = bpData.value.filter(entry => entry['Systolic (mmHg)'] >= 140);
  console.log('High BP data points:', highBPData.value.length);

  // Filter data for medium BP (systolic between 130-140 mmHg)
  mediumBPData.value = bpData.value.filter(
    entry => entry['Systolic (mmHg)'] >= 130 && entry['Systolic (mmHg)'] < 140
  );
  console.log('Medium BP data points:', mediumBPData.value.length);

  // Filter data for nocturnal measurements
  nocturnalData.value = bpData.value.filter(entry => entry['Measurement Mode'] === 'Nocturnal');
  console.log('Nocturnal measurements data points:', nocturnalData.value.length);

  // Count night measurements
  const allNightMeasurements = bpData.value.filter(isNightMeasurement);
  console.log('Total night measurements (by time):', allNightMeasurements.length);

  // Count measurements with Nocturnal mode
  const nocturnalMeasurements = bpData.value.filter(entry => entry['Measurement Mode'] === 'Nocturnal');
  console.log('Total measurements with Nocturnal mode:', nocturnalMeasurements.length);

  // Count measurements from NightView device during night hours
  const nightViewNightMeasurements = bpData.value.filter(entry =>
    entry.Device === 'NightView' && (entry.dateObj.getHours() >= 23 || entry.dateObj.getHours() <= 4)
  );
  console.log('Total night measurements from NightView device:', nightViewNightMeasurements.length);

  // Count measurements from EVOLV device during night hours
  const evolvNightMeasurements = bpData.value.filter(entry =>
    entry.Device === 'EVOLV' && (entry.dateObj.getHours() >= 23 || entry.dateObj.getHours() <= 4)
  );
  console.log('Total night measurements from EVOLV device:', evolvNightMeasurements.length);

  // Log counts for all systolic ranges for verification
  console.log('Filtered data counts for systolic ranges:');
  const systolicThresholds = bpStats.value.systolic.thresholds;
  for (let i = 0; i < systolicThresholds.length; i++) {
    const currentThreshold = systolicThresholds[i];
    if (i === 0) {
      // For the highest threshold, count readings >= that threshold
      const count = bpData.value.filter(d => d['Systolic (mmHg)'] >= currentThreshold).length;
      console.log(`>= ${currentThreshold} mmHg: ${count} measurements`);
    } else {
      // For other thresholds, count readings between this threshold and the next higher one
      const previousThreshold = systolicThresholds[i - 1];
      const count = bpData.value.filter(
        d => d['Systolic (mmHg)'] >= currentThreshold && d['Systolic (mmHg)'] < previousThreshold
      ).length;
      console.log(`${currentThreshold}-${previousThreshold} mmHg: ${count} measurements`);
    }
  }
};

// Function to check if a date is in DST for Romania
// Romania uses EEST (UTC+3) from the last Sunday in March to the last Sunday in October
// and EET (UTC+2) for the rest of the year
const isRomanianDST = (date) => {
  const year = date.getFullYear();

  // Last Sunday in March
  const marchLastDay = new Date(year, 2, 31);
  const marchLastSunday = new Date(
    year,
    2,
    31 - ((marchLastDay.getDay() + 6) % 7)
  );

  // Last Sunday in October
  const octoberLastDay = new Date(year, 9, 31);
  const octoberLastSunday = new Date(
    year,
    9,
    31 - ((octoberLastDay.getDay() + 6) % 7)
  );

  // Check if the date is between the last Sunday in March and the last Sunday in October
  return date >= marchLastSunday && date < octoberLastSunday;
};

// Check if a measurement was taken during night hours (23:00-04:00)
// Fixed algorithm to address the issue: "nu are cum sa fie atatea masuratori de noapte, verifica algoritmul"
// The previous algorithm was counting all measurements between 23:00 and 04:00 as night measurements,
// which was resulting in too many night measurements. The new algorithm only counts measurements as
// night measurements if they have the "Nocturnal" mode explicitly set, or if they're taken during
// night hours (23:00-04:00) AND from the NightView device.
const isNightMeasurement = (entry) => {
  // Get the hour in Bucharest timezone
  const hour = entry.dateObj.getHours();

  // Consider a measurement as a night measurement if:
  // 1. It has the "Nocturnal" mode explicitly set, OR
  // 2. It's taken during night hours (23:00-04:00) AND from the NightView device
  const isNocturnal = entry['Measurement Mode'] === 'Nocturnal';
  const isNightHour = hour >= 23 || hour <= 4;
  const isNightViewDevice = entry.Device === 'NightView';

  const result = isNocturnal || (isNightHour && isNightViewDevice);

  // Log for debugging with more detailed information
  console.log(`Checking night measurement: ${entry.Date} ${entry.Time}, Hour: ${hour}, Device: ${entry.Device}, Mode: ${entry['Measurement Mode']}, isNocturnal: ${isNocturnal}, isNightHour: ${isNightHour}, isNightViewDevice: ${isNightViewDevice}, Result: ${result}`);

  return result;
};

// Calculate average systolic and diastolic values
const calculateAverages = () => {
  if (bpData.value.length === 0) return;

  // Calculate sum of systolic and diastolic values
  const systolicSum = bpData.value.reduce((sum, entry) => sum + entry['Systolic (mmHg)'], 0);
  const diastolicSum = bpData.value.reduce((sum, entry) => sum + entry['Diastolic (mmHg)'], 0);

  // Calculate averages
  avgSystolic.value = Math.round(systolicSum / bpData.value.length);
  avgDiastolic.value = Math.round(diastolicSum / bpData.value.length);

  console.log('Average Systolic:', avgSystolic.value, 'mmHg');
  console.log('Average Diastolic:', avgDiastolic.value, 'mmHg');

  // Calculate moving averages
  calculateMovingAverages();
};

// Calculate moving averages over specified interval days
const calculateMovingAverages = () => {
  if (bpData.value.length === 0) return;

  // Create a map of dates to data points
  const dataByDate = new Map();

  // Group data points by date (ignoring time)
  bpData.value.forEach(entry => {
    const date = entry.dateObj.toISOString().split('T')[0]; // Get YYYY-MM-DD format
    if (!dataByDate.has(date)) {
      dataByDate.set(date, []);
    }
    dataByDate.get(date).push(entry);
  });

  // Sort dates
  const sortedDates = Array.from(dataByDate.keys()).sort();

  // Initialize arrays for moving averages
  const systolicAverages = [];
  const diastolicAverages = [];

  // Calculate moving averages for each data point
  bpData.value.forEach((entry, index) => {
    const currentDate = entry.dateObj.toISOString().split('T')[0];
    const currentDateIndex = sortedDates.indexOf(currentDate);

    // Get data points within the interval
    const intervalData = [];

    // Look back intervalDays.value days (including current day)
    for (let i = 0; i < intervalDays.value; i++) {
      const dateIndex = currentDateIndex - i;
      if (dateIndex >= 0) {
        const date = sortedDates[dateIndex];
        intervalData.push(...dataByDate.get(date));
      }
    }

    // Calculate averages for this interval
    if (intervalData.length > 0) {
      const systolicSum = intervalData.reduce((sum, dp) => sum + dp['Systolic (mmHg)'], 0);
      const diastolicSum = intervalData.reduce((sum, dp) => sum + dp['Diastolic (mmHg)'], 0);

      systolicAverages[index] = Math.round(systolicSum / intervalData.length);
      diastolicAverages[index] = Math.round(diastolicSum / intervalData.length);
    } else {
      // Fallback to overall average if no data in interval
      systolicAverages[index] = avgSystolic.value;
      diastolicAverages[index] = avgDiastolic.value;
    }
  });

  // Store the calculated averages
  avgSystolicByInterval.value = systolicAverages;
  avgDiastolicByInterval.value = diastolicAverages;

  console.log(`Calculated moving averages over ${intervalDays.value} day intervals`);
};

// Prepare data for Chart.js
const prepareChartData = () => {
  if (bpData.value.length === 0) return;

  // Extract data for chart
  const labels = bpData.value.map(d => d.formattedDate);
  const systolicData = bpData.value.map(d => d['Systolic (mmHg)']);
  const diastolicData = bpData.value.map(d => d['Diastolic (mmHg)']);
  const pulseData = bpData.value.map(d => d['Pulse (bpm)']);

  // Determine point styles based on time of day
  const pointStyles = bpData.value.map(d => isNightMeasurement(d) ? 'rect' : 'circle');
  const pointRadiuses = bpData.value.map(d => isNightMeasurement(d) ? 4 : 1);

  // Group data by month for month segments
  const monthIndices = [];
  let currentMonth = null;

  bpData.value.forEach((d, index) => {
    const month = d.dateObj.getMonth();
    const year = d.dateObj.getFullYear();
    const monthYear = `${month}-${year}`;

    if (monthYear !== currentMonth) {
      currentMonth = monthYear;
      monthIndices.push({
        index,
        month: ['Ian', 'Feb', 'Mar', 'Apr', 'Mai', 'Iun', 'Iul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'][month],
        year
      });
    }
  });

  // Create chart data
  chartData.value = {
    labels: labels,
    datasets: [
      {
        label: 'Systolic (mmHg)',
        backgroundColor: 'rgba(255, 0, 0, 0.2)',
        borderColor: '#FF0000',
        borderWidth: 2,
        pointBackgroundColor: bpData.value.map(d => isNightMeasurement(d) ? '#FF9900' : '#FF0000'),
        pointStyle: pointStyles,
        pointRadius: pointRadiuses,
        data: systolicData,
        tension: 0.9
      },
      {
        label: 'Diastolic (mmHg)',
        backgroundColor: 'rgba(0, 0, 255, 0.2)',
        borderColor: '#0000FF',
        borderWidth: 2,
        pointBackgroundColor: bpData.value.map(d => isNightMeasurement(d) ? '#9900FF' : '#0000FF'),
        pointStyle: pointStyles,
        pointRadius: pointRadiuses,
        data: diastolicData,
        tension: 0.9
      },
      {
        label: 'Pulse (bpm)',
        backgroundColor: 'rgba(0, 204, 0, 0.2)',
        borderColor: '#00CC00',
        borderWidth: 2,
        pointBackgroundColor: bpData.value.map(d => isNightMeasurement(d) ? '#99CC00' : '#00CC00'),
        pointStyle: pointStyles,
        pointRadius: pointRadiuses,
        data: pulseData,
        tension: 0.9
      },
      {
        label: `Avg Systolic (${intervalDays.value} days)`,
        backgroundColor: `${avgSystolicColor.value}4D`, // 30% opacity (4D in hex)
        borderColor: `${avgSystolicColor.value}B3`, // 70% opacity (B3 in hex)
        borderWidth: lineThickness.value,
        pointRadius: 0,
        data: avgSystolicByInterval.value,
        tension: 0.2,
        fill: false
      },
      {
        label: `Avg Diastolic (${intervalDays.value} days)`,
        backgroundColor: `${avgDiastolicColor.value}4D`, // 30% opacity (4D in hex)
        borderColor: `${avgDiastolicColor.value}B3`, // 70% opacity (B3 in hex)
        borderWidth: lineThickness.value,
        pointRadius: 0,
        data: avgDiastolicByInterval.value,
        tension: 0.2,
        fill: false
      }
    ]
  };

  // Create chart options
  chartOptions.value = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      tooltip: {
        mode: 'index',
        intersect: false,
        callbacks: {
          label: function(context) {
            let label = context.dataset.label || '';
            if (label) {
              label += ': ';
            }
            if (context.parsed.y !== null) {
              label += context.parsed.y;
            }
            return label;
          }
        }
      },
      legend: {
        position: 'top',
        labels: {
          padding: 20,
          boxWidth: 15,
          font: {
            size: 14
          }
        }
      },
      annotation: {
        annotations: createAnnotations(monthIndices)
      },
      datalabels: {
        align: 'top',
        anchor: 'end',
        offset: 15,
        rotation: function(context) {
          // More aggressive rotation to prevent overlap
          return context.dataIndex % 3 === 0 ? -20 : (context.dataIndex % 3 === 1 ? 0 : 20);
        },
        color: function(context) {
          // Use different colors for night measurements
          if (context.datasetIndex === 0) {
            const entry = bpData.value[context.dataIndex];
            return isNightMeasurement(entry) ? '#FF9900' : context.dataset.borderColor;
          }
          return context.dataset.borderColor;
        },
        font: {
          weight: 'bold',
          size: 11
        },
        formatter: function(value, context) {
          // Only show labels for systolic values above 140 (including 140)
          if (context.datasetIndex === 0 && value >= 140) {
            // Get the corresponding diastolic value
            const diastolicValue = bpData.value[context.dataIndex]['Diastolic (mmHg)'];
            // Format as "sys/dia"
            return `${value}/${diastolicValue}`;
          }
          // Hide labels for other values
          return null;
        },
        padding: 6,
        borderRadius: 4,
        backgroundColor: function(context) {
          // Use different background for night measurements
          if (context.datasetIndex === 0) {
            const entry = bpData.value[context.dataIndex];
            return isNightMeasurement(entry) ? 'rgba(255, 153, 0, 0.2)' : 'rgba(255, 255, 255, 0.9)';
          }
          return 'rgba(255, 255, 255, 0.9)';
        },
        display: function(context) {
          // More aggressive filtering to prevent overlap
          const index = context.dataIndex;
          const value = context.dataset.data[index];
          // Only show every third label if there are many data points
          return value >= 140 && context.datasetIndex === 0 &&
                 (bpData.value.length < 20 || index % 3 === 0);
        }
      }
    },
    scales: {
      x: {
        ticks: {
          maxRotation: 45,
          minRotation: 45,
          font: {
            size: 12
          },
          padding: 10
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        }
      },
      y: {
        min: 40,
        max: 200,
        ticks: {
          font: {
            size: 12
          },
          padding: 10,
          callback: function(value) {
            return value;
          }
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        },
        title: {
          display: true,
          text: 'mmHg / bpm',
          font: {
            size: 14,
            weight: 'bold'
          },
          padding: {
            top: 0,
            bottom: 10
          }
        }
      }
    }
  };
};

// Prepare data for high BP chart (systolic >= 140 mmHg)
const prepareHighBPChartData = () => {
  if (highBPData.value.length === 0) return;

  // Extract data for chart
  const labels = highBPData.value.map(d => {
    const date = d.dateObj;
    return `${date.toLocaleDateString()} ${date.getHours()}:${date.getMinutes().toString().padStart(2, '0')}`;
  });
  const systolicData = highBPData.value.map(d => d['Systolic (mmHg)']);
  const diastolicData = highBPData.value.map(d => d['Diastolic (mmHg)']);

  // Determine point styles based on time of day
  const pointStyles = highBPData.value.map(d => isNightMeasurement(d) ? 'rect' : 'circle');
  const pointRadiuses = highBPData.value.map(d => isNightMeasurement(d) ? 4 : 2);

  // Create chart data
  highBPChartData.value = {
    labels: labels,
    datasets: [
      {
        label: 'Systolic (mmHg)',
        backgroundColor: 'rgba(255, 0, 0, 0.2)',
        borderColor: '#FF0000',
        borderWidth: 2,
        pointBackgroundColor: highBPData.value.map(d => isNightMeasurement(d) ? '#FF9900' : '#FF0000'),
        pointStyle: pointStyles,
        pointRadius: pointRadiuses,
        data: systolicData,
        tension: 0.4
      },
      {
        label: 'Diastolic (mmHg)',
        backgroundColor: 'rgba(0, 0, 255, 0.2)',
        borderColor: '#0000FF',
        borderWidth: 2,
        pointBackgroundColor: highBPData.value.map(d => isNightMeasurement(d) ? '#9900FF' : '#0000FF'),
        pointStyle: pointStyles,
        pointRadius: pointRadiuses,
        data: diastolicData,
        tension: 0.4
      }
    ]
  };

  // Create chart options
  highBPChartOptions.value = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      tooltip: {
        mode: 'index',
        intersect: false,
        callbacks: {
          label: function(context) {
            let label = context.dataset.label || '';
            if (label) {
              label += ': ';
            }
            if (context.parsed.y !== null) {
              label += context.parsed.y;
            }
            return label;
          },
          title: function(context) {
            return `Date: ${context[0].label}`;
          }
        }
      },
      legend: {
        position: 'top',
        labels: {
          padding: 10,
          boxWidth: 15,
          font: {
            size: 12
          }
        }
      },
      datalabels: {
        align: 'top',
        anchor: 'end',
        offset: 15,
        rotation: function(context) {
          // More aggressive rotation to prevent overlap
          return context.dataIndex % 3 === 0 ? -20 : (context.dataIndex % 3 === 1 ? 0 : 20);
        },
        color: function(context) {
          // Use different colors for night measurements
          if (context.datasetIndex === 0) {
            const entry = highBPData.value[context.dataIndex];
            return isNightMeasurement(entry) ? '#FF9900' : context.dataset.borderColor;
          }
          return context.dataset.borderColor;
        },
        font: {
          weight: 'bold',
          size: 11
        },
        formatter: function(value, context) {
          // Always show labels for systolic values
          if (context.datasetIndex === 0) { // Systolic
            // Get the corresponding diastolic value
            const diastolicValue = highBPData.value[context.dataIndex]['Diastolic (mmHg)'];
            // Format as "sys/dia"
            return `${value}/${diastolicValue}`;
          }
          // Hide labels for diastolic dataset to avoid duplication
          return null;
        },
        padding: 6,
        borderRadius: 4,
        backgroundColor: function(context) {
          // Use different background for night measurements
          if (context.datasetIndex === 0) {
            const entry = highBPData.value[context.dataIndex];
            return isNightMeasurement(entry) ? 'rgba(255, 153, 0, 0.2)' : 'rgba(255, 255, 255, 0.9)';
          }
          return 'rgba(255, 255, 255, 0.9)';
        },
        // Add more space between labels
        display: function(context) {
          // More aggressive filtering to prevent overlap
          if (highBPData.value.length > 15) {
            return context.dataIndex % 3 === 0 && context.datasetIndex === 0;
          } else if (highBPData.value.length > 8) {
            return context.dataIndex % 2 === 0 && context.datasetIndex === 0;
          }
          return context.datasetIndex === 0;
        }
      }
    },
    scales: {
      x: {
        ticks: {
          maxRotation: 45,
          minRotation: 45,
          font: {
            size: 10
          },
          padding: 5
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        }
      },
      y: {
        min: 40,
        max: 200,
        ticks: {
          font: {
            size: 12
          },
          padding: 5
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        },
        title: {
          display: true,
          text: 'mmHg',
          font: {
            size: 12,
            weight: 'bold'
          }
        }
      }
    }
  };
};

// Prepare data for nocturnal measurements chart
const prepareNocturnalChartData = () => {
  if (nocturnalData.value.length === 0) return;

  // Extract data for chart
  const labels = nocturnalData.value.map(d => {
    const date = d.dateObj;
    return `${date.toLocaleDateString()} ${date.getHours()}:${date.getMinutes().toString().padStart(2, '0')}`;
  });
  const systolicData = nocturnalData.value.map(d => d['Systolic (mmHg)']);
  const diastolicData = nocturnalData.value.map(d => d['Diastolic (mmHg)']);
  const pulseData = nocturnalData.value.map(d => d['Pulse (bpm)']);

  // Create chart data
  nocturnalChartData.value = {
    labels: labels,
    datasets: [
      {
        label: 'Systolic (mmHg)',
        backgroundColor: 'rgba(255, 153, 0, 0.2)',
        borderColor: '#FF9900',
        borderWidth: 2,
        pointBackgroundColor: '#FF9900',
        pointStyle: 'rect',
        pointRadius: 4,
        data: systolicData,
        tension: 0.4
      },
      {
        label: 'Diastolic (mmHg)',
        backgroundColor: 'rgba(153, 0, 255, 0.2)',
        borderColor: '#9900FF',
        borderWidth: 2,
        pointBackgroundColor: '#9900FF',
        pointStyle: 'rect',
        pointRadius: 4,
        data: diastolicData,
        tension: 0.4
      },
      {
        label: 'Pulse (bpm)',
        backgroundColor: 'rgba(153, 204, 0, 0.2)',
        borderColor: '#99CC00',
        borderWidth: 2,
        pointBackgroundColor: '#99CC00',
        pointStyle: 'rect',
        pointRadius: 4,
        data: pulseData,
        tension: 0.4
      }
    ]
  };

  // Create chart options
  nocturnalChartOptions.value = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      tooltip: {
        mode: 'index',
        intersect: false,
        callbacks: {
          label: function(context) {
            let label = context.dataset.label || '';
            if (label) {
              label += ': ';
            }
            if (context.parsed.y !== null) {
              label += context.parsed.y;
            }
            return label;
          },
          title: function(context) {
            return `Date: ${context[0].label}`;
          }
        }
      },
      legend: {
        position: 'top',
        labels: {
          padding: 10,
          boxWidth: 15,
          font: {
            size: 12
          }
        }
      },
      datalabels: {
        align: 'top',
        anchor: 'end',
        offset: 15,
        rotation: function(context) {
          // More aggressive rotation to prevent overlap
          return context.dataIndex % 3 === 0 ? -20 : (context.dataIndex % 3 === 1 ? 0 : 20);
        },
        color: function(context) {
          return context.dataset.borderColor;
        },
        font: {
          weight: 'bold',
          size: 11
        },
        formatter: function(value, context) {
          if (context.datasetIndex === 0) { // Systolic
            // Get the corresponding diastolic value
            const diastolicValue = nocturnalData.value[context.dataIndex]['Diastolic (mmHg)'];
            // Format as "sys/dia"
            return `${value}/${diastolicValue}`;
          }
          // Hide labels for other datasets to avoid duplication
          return null;
        },
        padding: 6,
        borderRadius: 4,
        backgroundColor: 'rgba(255, 255, 255, 0.9)',
        // Add more space between labels
        display: function(context) {
          // More aggressive filtering to prevent overlap
          if (nocturnalData.value.length > 15) {
            return context.dataIndex % 3 === 0 && context.datasetIndex === 0;
          } else if (nocturnalData.value.length > 8) {
            return context.dataIndex % 2 === 0 && context.datasetIndex === 0;
          }
          return context.datasetIndex === 0;
        }
      }
    },
    scales: {
      x: {
        ticks: {
          maxRotation: 45,
          minRotation: 45,
          font: {
            size: 10
          },
          padding: 5
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        }
      },
      y: {
        min: 40,
        max: 200,
        ticks: {
          font: {
            size: 12
          },
          padding: 5
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        },
        title: {
          display: true,
          text: 'mmHg / bpm',
          font: {
            size: 12,
            weight: 'bold'
          }
        }
      }
    }
  };
};

// Prepare data for medium BP chart (systolic between 130-140 mmHg)
const prepareMediumBPChartData = () => {
  if (mediumBPData.value.length === 0) return;

  // Extract data for chart
  const labels = mediumBPData.value.map(d => {
    const date = d.dateObj;
    return `${date.toLocaleDateString()} ${date.getHours()}:${date.getMinutes().toString().padStart(2, '0')}`;
  });
  const systolicData = mediumBPData.value.map(d => d['Systolic (mmHg)']);
  const diastolicData = mediumBPData.value.map(d => d['Diastolic (mmHg)']);

  // Determine point styles based on time of day
  const pointStyles = mediumBPData.value.map(d => isNightMeasurement(d) ? 'rect' : 'circle');
  const pointRadiuses = mediumBPData.value.map(d => isNightMeasurement(d) ? 4 : 2);

  // Create chart data
  mediumBPChartData.value = {
    labels: labels,
    datasets: [
      {
        label: 'Systolic (mmHg)',
        backgroundColor: 'rgba(255, 204, 0, 0.2)',
        borderColor: '#FFCC00',
        borderWidth: 2,
        pointBackgroundColor: mediumBPData.value.map(d => isNightMeasurement(d) ? '#FF9900' : '#FFCC00'),
        pointStyle: pointStyles,
        pointRadius: pointRadiuses,
        data: systolicData,
        tension: 0.4
      },
      {
        label: 'Diastolic (mmHg)',
        backgroundColor: 'rgba(0, 0, 255, 0.2)',
        borderColor: '#0000FF',
        borderWidth: 2,
        pointBackgroundColor: mediumBPData.value.map(d => isNightMeasurement(d) ? '#9900FF' : '#0000FF'),
        pointStyle: pointStyles,
        pointRadius: pointRadiuses,
        data: diastolicData,
        tension: 0.4
      }
    ]
  };

  // Create chart options
  mediumBPChartOptions.value = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      tooltip: {
        mode: 'index',
        intersect: false,
        callbacks: {
          label: function(context) {
            let label = context.dataset.label || '';
            if (label) {
              label += ': ';
            }
            if (context.parsed.y !== null) {
              label += context.parsed.y;
            }
            return label;
          },
          title: function(context) {
            return `Date: ${context[0].label}`;
          }
        }
      },
      legend: {
        position: 'top',
        labels: {
          padding: 10,
          boxWidth: 15,
          font: {
            size: 12
          }
        }
      },
      datalabels: {
        align: 'top',
        anchor: 'end',
        offset: 15,
        rotation: function(context) {
          // More aggressive rotation to prevent overlap
          return context.dataIndex % 3 === 0 ? -20 : (context.dataIndex % 3 === 1 ? 0 : 20);
        },
        color: function(context) {
          // Use different colors for night measurements
          if (context.datasetIndex === 0) {
            const entry = mediumBPData.value[context.dataIndex];
            return isNightMeasurement(entry) ? '#FF9900' : context.dataset.borderColor;
          }
          return context.dataset.borderColor;
        },
        font: {
          weight: 'bold',
          size: 11
        },
        formatter: function(value, context) {
          // Always show labels for systolic values
          if (context.datasetIndex === 0) { // Systolic
            // Get the corresponding diastolic value
            const diastolicValue = mediumBPData.value[context.dataIndex]['Diastolic (mmHg)'];
            // Format as "sys/dia"
            return `${value}/${diastolicValue}`;
          }
          // Hide labels for diastolic dataset to avoid duplication
          return null;
        },
        padding: 6,
        borderRadius: 4,
        backgroundColor: function(context) {
          // Use different background for night measurements
          if (context.datasetIndex === 0) {
            const entry = mediumBPData.value[context.dataIndex];
            return isNightMeasurement(entry) ? 'rgba(255, 153, 0, 0.2)' : 'rgba(255, 255, 255, 0.9)';
          }
          return 'rgba(255, 255, 255, 0.9)';
        },
        // Add more space between labels
        display: function(context) {
          // More aggressive filtering to prevent overlap
          if (mediumBPData.value.length > 15) {
            return context.dataIndex % 3 === 0 && context.datasetIndex === 0;
          } else if (mediumBPData.value.length > 8) {
            return context.dataIndex % 2 === 0 && context.datasetIndex === 0;
          }
          return context.datasetIndex === 0;
        }
      }
    },
    scales: {
      x: {
        ticks: {
          maxRotation: 45,
          minRotation: 45,
          font: {
            size: 10
          },
          padding: 5
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        }
      },
      y: {
        min: 40,
        max: 200,
        ticks: {
          font: {
            size: 12
          },
          padding: 5
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        },
        title: {
          display: true,
          text: 'mmHg',
          font: {
            size: 12,
            weight: 'bold'
          }
        }
      }
    }
  };
};

// Prepare data for heat map of high BP measurements (systolic >= 140 mmHg)
const prepareHeatMapData = () => {
  if (highBPData.value.length === 0) return;

  // Group data by day of week and hour of day
  const heatMapCounts = {};

  // Initialize the counts object with zeros for all day-hour combinations
  for (let day = 0; day < 7; day++) {
    for (let hour = 0; hour < 24; hour++) {
      const key = `${day}-${hour}`;
      heatMapCounts[key] = 0;
    }
  }

  // Group data by day of week, hour of day, and actual date
  const dateMap = {}; // Map to store actual dates for each day-hour combination

  highBPData.value.forEach(entry => {
    const day = entry.dateObj.getDay(); // 0 = Sunday, 1 = Monday, etc.
    const hour = entry.dateObj.getHours();
    const key = `${day}-${hour}`;

    // Store the date information for this day-hour combination
    if (!dateMap[key]) {
      dateMap[key] = [];
    }

    // Add the date (day and month) to the dateMap if it's not already there
    const dateDay = entry.dateObj.getDate();
    const dateMonth = entry.dateObj.getMonth() + 1; // getMonth() returns 0-11
    const dateStr = `${dateDay}/${dateMonth}`;

    if (!dateMap[key].includes(dateStr)) {
      dateMap[key].push(dateStr);
    }

    heatMapCounts[key]++;
  });

  // Convert the counts to a format suitable for a scatter chart
  const data = [];
  for (let day = 0; day < 7; day++) {
    for (let hour = 0; hour < 24; hour++) {
      const key = `${day}-${hour}`;
      const count = heatMapCounts[key];
      if (count > 0) {
        data.push({
          x: day,
          y: hour,
          r: Math.min(20, Math.max(5, count * 3)), // Size based on count (min 5, max 20)
          count: count, // Store the count for tooltips
          dates: dateMap[key] // Store the dates for this day-hour combination
        });
      }
    }
  }

  // Create chart data
  heatMapData.value = {
    datasets: [{
      label: 'Măsurători cu tensiune sistolică ≥ 140 mmHg',
      data: data,
      backgroundColor: function(context) {
        const count = context.raw.count;
        // Color intensity based on count
        const intensity = Math.min(1, count / 5); // Max intensity at 5 or more measurements
        return `rgba(255, 0, 0, ${intensity})`;
      },
      borderColor: 'rgba(255, 0, 0, 0.5)',
      borderWidth: 1,
      pointStyle: 'circle',
      radius: function(context) {
        return context.raw.r;
      }
    }]
  };

  // Create chart options
  heatMapOptions.value = {
    responsive: true,
    maintainAspectRatio: false,
    scales: {
      x: {
        type: 'linear',
        position: 'bottom',
        min: -0.5,
        max: 6.5,
        ticks: {
          stepSize: 1,
          callback: function(value) {
            const days = ['Duminică', 'Luni', 'Marți', 'Miercuri', 'Joi', 'Vineri', 'Sâmbătă'];
            // Ensure value is a valid index for the days array
            const dayIndex = Math.round(value);
            if (dayIndex >= 0 && dayIndex < days.length) {
              return `Zi: ${days[dayIndex]}`;
            } else {
              return `Zi: ${dayIndex}`; // Return the index if it's not valid
            }
          }
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        },
        title: {
          display: true,
          text: 'Ziua săptămânii (vezi tooltip pentru data exactă)',
          font: {
            size: 14,
            weight: 'bold'
          }
        }
      },
      y: {
        type: 'linear',
        min: -0.5,
        max: 23.5,
        ticks: {
          stepSize: 1,
          callback: function(value) {
            // Ensure value is a valid hour
            const hour = Math.round(value);
            if (hour >= 0 && hour < 24) {
              return `${hour.toString().padStart(2, '0')}:00`;
            } else {
              return `${hour}:00`;
            }
          }
        },
        grid: {
          display: true,
          color: 'rgba(0, 0, 0, 0.1)'
        },
        title: {
          display: true,
          text: 'Ora zilei',
          font: {
            size: 14,
            weight: 'bold'
          }
        }
      }
    },
    plugins: {
      tooltip: {
        callbacks: {
          title: function(context) {
            const item = context[0];
            const dayIndex = Math.round(item.raw.x);
            const days = ['Duminică', 'Luni', 'Marți', 'Miercuri', 'Joi', 'Vineri', 'Sâmbătă'];
            const day = dayIndex >= 0 && dayIndex < days.length ? days[dayIndex] : `Ziua ${dayIndex}`;
            const hour = Math.round(item.raw.y);
            const hourStr = hour >= 0 && hour < 24 ? `${hour.toString().padStart(2, '0')}:00` : `${hour}:00`;
            return `${day}, ora ${hourStr}`;
          },
          label: function(context) {
            const count = context.raw.count;
            return `${count} ${count === 1 ? 'măsurătoare' : 'măsurători'} cu tensiune sistolică ≥ 140 mmHg`;
          },
          afterLabel: function(context) {
            let result = 'Cercurile mai mari indică mai multe măsurători cu valori ridicate în acest interval orar.';

            // Add date information if available
            if (context.raw.dates && context.raw.dates.length > 0) {
              const dateCount = context.raw.dates.length;
              const datesStr = context.raw.dates.join(', ');

              if (dateCount === 1) {
                result += `\nData exactă: ${datesStr}`;
              } else {
                result += `\nDatele exacte: ${datesStr}`;
              }
            }

            return result;
          }
        },
        backgroundColor: 'rgba(0, 0, 0, 0.8)',
        titleFont: {
          weight: 'bold'
        },
        padding: 10,
        displayColors: false
      },
      legend: {
        display: true,
        position: 'bottom',
        labels: {
          generateLabels: function(chart) {
            return [
              {
                text: 'Cercurile reprezintă măsurători cu tensiune sistolică ≥ 140 mmHg',
                fillStyle: 'rgba(255, 0, 0, 0.7)',
                strokeStyle: 'rgba(255, 0, 0, 0.5)',
                lineWidth: 1,
                hidden: false,
                index: 0
              },
              {
                text: 'Cercurile mai mari indică mai multe măsurători în acel moment',
                fillStyle: 'rgba(255, 0, 0, 0.7)',
                strokeStyle: 'rgba(255, 0, 0, 0.5)',
                lineWidth: 1,
                hidden: false,
                index: 1
              },
              {
                text: 'Plasați cursorul peste cerc pentru a vedea datele exacte (zi/lună)',
                fillStyle: 'rgba(255, 0, 0, 0.7)',
                strokeStyle: 'rgba(255, 0, 0, 0.5)',
                lineWidth: 1,
                hidden: false,
                index: 2
              },
            ];
          },
          boxWidth: 15,
          boxHeight: 15,
          padding: 20,
          font: {
            size: 12
          }
        }
      },
      title: {
        display: true,
        text: 'Heat Map al măsurătorilor cu tensiune sistolică ≥ 140 mmHg (distribuția pe zile și ore)',
        font: {
          size: 16,
          weight: 'bold'
        }
      },
      subtitle: {
        display: true,
        text: 'Dimensiunea cercurilor indică numărul de măsurători cu valori ridicate în ziua și ora respectivă. Plasați cursorul peste cerc pentru a vedea datele exacte (zi/lună).',
        font: {
          size: 14
        },
        padding: {
          bottom: 10
        }
      }
    }
  };
};

// Create annotations for blood pressure levels and month segments
const createAnnotations = (monthIndices = []) => {
  const annotations = [];

  // Add horizontal lines for systolic thresholds
  bpLevels.systolic.forEach((level, index) => {
    if (index === bpLevels.systolic.length - 1) return; // Skip the last level (0)

    annotations.push({
      type: 'line',
      mode: 'horizontal',
      scaleID: 'y',
      value: level.value,
      borderColor: level.color,
      borderWidth: 1,
      borderDash: [5, 5],
      label: {
        content: `${level.label} (${level.value} mmHg)`,
        enabled: true,
        position: 'left',
        backgroundColor: 'rgba(255, 255, 255, 0.8)',
        color: '#333',
        font: {
          size: 10
        },
        padding: 5
      }
    });
  });

  // Add horizontal lines for diastolic thresholds
  bpLevels.diastolic.forEach((level, index) => {
    if (index === bpLevels.diastolic.length - 1) return; // Skip the last level (0)

    annotations.push({
      type: 'line',
      mode: 'horizontal',
      scaleID: 'y',
      value: level.value,
      borderColor: level.color,
      borderWidth: 1,
      borderDash: [2, 2],
      label: {
        content: `${level.label} (${level.value} mmHg)`,
        enabled: true,
        position: 'right',
        backgroundColor: 'rgba(255, 255, 255, 0.8)',
        color: '#333',
        font: {
          size: 10
        },
        padding: 5
      }
    });
  });

  // Add vertical lines and labels for month segments
  monthIndices.forEach(({ index, month, year }) => {
    annotations.push({
      type: 'line',
      mode: 'vertical',
      scaleID: 'x',
      value: index,
      borderColor: 'rgba(0, 0, 0, 0.5)',
      borderWidth: 1,
      borderDash: [5, 5],
      label: {
        content: `${month} ${year}`,
        enabled: true,
        position: 'bottom',
        backgroundColor: 'rgba(255, 255, 255, 0.8)',
        color: '#333',
        font: {
          size: 12,
          weight: 'bold'
        },
        padding: 5,
        yAdjust: 20 // Move label below the x-axis
      }
    });
  });

  return annotations;
};

// Handle changes to the interval days
const updateIntervalDays = () => {
  // Ensure intervalDays is a positive number
  if (intervalDays.value <= 0) {
    intervalDays.value = 1;
  } else if (intervalDays.value > 30) {
    intervalDays.value = 30;
  }

  console.log(`Interval days changed to ${intervalDays.value}`);

  // Recalculate moving averages
  calculateMovingAverages();

  // Update chart data
  prepareChartData();
};

// Handle changes to the line thickness
const updateLineThickness = () => {
  // Ensure lineThickness is a positive number
  if (lineThickness.value <= 0) {
    lineThickness.value = 1;
  } else if (lineThickness.value > 20) {
    lineThickness.value = 20;
  }

  console.log(`Line thickness changed to ${lineThickness.value}`);

  // Update chart data
  prepareChartData();
};

// Handle changes to the line colors
const updateLineColors = () => {
  console.log(`Line colors changed to: Systolic=${avgSystolicColor.value}, Diastolic=${avgDiastolicColor.value}`);

  // Update chart data
  prepareChartData();
};

// Handle changes to the selected CSV file
const handleCsvFileChange = async () => {
  console.log(`Selected CSV file changed to: ${selectedCsvFile.value.label}`);
  uploadedCsvFile.value = null; // Clear any uploaded file

  // Reset data
  bpData.value = [];
  chartData.value = null;
  chartOptions.value = null;

  // Reload data with the new file
  try {
    await loadData();
    console.log('Data loaded and chart prepared');
  } catch (err) {
    console.error('Error loading data:', err);
    error.value = 'Error loading data: ' + err.message;
  }
};

// Handle file uploads
const handleFileUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  // Check if it's a CSV file
  if (!file.name.endsWith('.csv')) {
    error.value = 'Please upload a CSV file';
    return;
  }

  console.log(`File uploaded: ${file.name}`);
  uploadedCsvFile.value = file;
  selectedCsvFile.value = null; // Clear the selected file

  // Reset data
  bpData.value = [];
  chartData.value = null;
  chartOptions.value = null;

  // Reload data with the uploaded file
  try {
    await loadData();
    console.log('Data loaded and chart prepared');
  } catch (err) {
    console.error('Error loading data:', err);
    error.value = 'Error loading data: ' + err.message;
  }
};

// Format date for display in Romanian
const formatDateRo = (date) => {
  if (!date) return '';

  const day = date.getDate();
  const month = date.getMonth() + 1;
  const year = date.getFullYear();

  // Romanian month names
  const monthNames = ['Ianuarie', 'Februarie', 'Martie', 'Aprilie', 'Mai', 'Iunie',
                      'Iulie', 'August', 'Septembrie', 'Octombrie', 'Noiembrie', 'Decembrie'];

  return `${day} ${monthNames[month-1]} ${year}`;
};

// Computed property for the measurement period text
const measurementPeriodText = computed(() => {
  if (!measurementStartDate.value || !measurementEndDate.value) {
    return '';
  }

  const startFormatted = formatDateRo(measurementStartDate.value);
  const endFormatted = formatDateRo(measurementEndDate.value);

  return `Perioada de măsurare: ${startFormatted} - ${endFormatted}`;
});

// Load data when component is mounted
onMounted(async () => {
  console.log('Component mounted');
  try {
    await loadData();
    console.log('Data loaded and chart prepared');
  } catch (err) {
    console.error('Error in onMounted:', err);
    error.value = 'Error initializing graph: ' + err.message;
  }
});
</script>

<template>
  <div class="bp-graph-container">
    <h2>Blood Pressure and Pulse Graph</h2>
    <div v-if="measurementPeriodText" class="measurement-period">
      {{ measurementPeriodText }}
    </div>

    <div class="controls-header">
      <button class="toggle-controls-btn" @click="controlsCollapsed = !controlsCollapsed">
        {{ controlsCollapsed ? 'Arată controalele' : 'Ascunde controalele' }}
      </button>
    </div>

    <div class="controls" :class="{ 'collapsed': controlsCollapsed }">
      <!-- CSV File Selection -->
      <div class="file-selection-control">
        <div class="file-dropdown">
          <label for="csv-file-select">Selectează fișierul CSV:</label>
          <select
            id="csv-file-select"
            v-model="selectedCsvFile"
            @change="handleCsvFileChange"
            :disabled="uploadedCsvFile !== null"
          >
            <option v-for="file in availableCsvFiles" :key="file.path" :value="file">
              {{ file.label }}
            </option>
          </select>
        </div>

        <div class="file-upload">
          <label for="csv-file-upload">Sau încarcă un fișier CSV:</label>
          <input
            id="csv-file-upload"
            type="file"
            accept=".csv"
            @change="handleFileUpload"
            :disabled="selectedCsvFile !== null && uploadedCsvFile === null"
          />
          <div v-if="uploadedCsvFile" class="uploaded-file-info">
            Fișier încărcat: {{ uploadedCsvFile.name }}
            <button @click="uploadedCsvFile = null; selectedCsvFile = availableCsvFiles[0]; handleCsvFileChange()">Șterge</button>
          </div>
        </div>
      </div>

      <div class="interval-control">
        <label for="interval-days">Interval zile pentru medie:</label>
        <input
          id="interval-days"
          type="number"
          v-model.number="intervalDays"
          min="1"
          max="30"
          @change="updateIntervalDays"
        />
      </div>

      <div class="line-control">
        <label for="line-thickness">Grosime linie:</label>
        <input
          id="line-thickness"
          type="number"
          v-model.number="lineThickness"
          min="1"
          max="20"
          @change="updateLineThickness"
        />
      </div>

      <div class="color-control">
        <div class="color-picker">
          <label for="systolic-color">Culoare Sistolică:</label>
          <input
            id="systolic-color"
            type="color"
            v-model="avgSystolicColor"
            @change="updateLineColors"
          />
        </div>

        <div class="color-picker">
          <label for="diastolic-color">Culoare Diastolică:</label>
          <input
            id="diastolic-color"
            type="color"
            v-model="avgDiastolicColor"
            @change="updateLineColors"
          />
        </div>
      </div>
    </div>

    <div class="night-measurement-legend">
      <div class="legend-item">
        <span class="legend-symbol night">■</span>
        <span class="legend-text">Măsurători de noapte (mod Nocturnal sau NightView între 23:00 - 04:00)</span>
      </div>
      <div class="legend-item">
        <span class="legend-symbol day">●</span>
        <span class="legend-text">Măsurători de zi</span>
      </div>
    </div>

    <div v-if="loading" class="loading">
      Loading data...
    </div>

    <div v-if="error" class="error">
      {{ error }}
    </div>

    <div v-if="!loading && !error" class="graph-wrapper">
      <Line
        v-if="chartData && chartOptions"
        :data="chartData"
        :options="chartOptions"
      />
    </div>

    <!-- Separate graphs for specific BP ranges -->
    <div v-if="!loading && !error" class="specific-graphs">
      <!-- High BP Graph (Systolic >= 140 mmHg) -->
      <div v-if="highBPData.length > 0" class="specific-graph-container">
        <h3>Tensiune Sistolică ≥ 140 mmHg</h3>
        <div class="specific-graph-wrapper">
          <Line
            v-if="highBPChartData && highBPChartOptions"
            :data="highBPChartData"
            :options="highBPChartOptions"
          />
        </div>
        <div class="specific-graph-info">
          <p>{{ highBPData.length }} măsurători cu tensiune sistolică ≥ 140 mmHg</p>
          <p>{{ highBPData.filter(isNightMeasurement).length }} măsurători de noapte</p>
        </div>
      </div>

      <!-- Medium BP Graph (Systolic between 130-140 mmHg) -->
      <div v-if="mediumBPData.length > 0" class="specific-graph-container">
        <h3>Tensiune Sistolică între 130-140 mmHg</h3>
        <div class="specific-graph-wrapper">
          <Line
            v-if="mediumBPChartData && mediumBPChartOptions"
            :data="mediumBPChartData"
            :options="mediumBPChartOptions"
          />
        </div>
        <div class="specific-graph-info">
          <p>{{ mediumBPData.length }} măsurători cu tensiune sistolică între 130-140 mmHg</p>
          <p>{{ mediumBPData.filter(isNightMeasurement).length }} măsurători de noapte</p>
        </div>
      </div>

      <!-- Nocturnal Measurements Graph -->
      <div v-if="nocturnalData.length > 0" class="specific-graph-container nocturnal-graph">
        <h3>Măsurători în Mod Nocturnal</h3>
        <div class="specific-graph-wrapper">
          <Line
            v-if="nocturnalChartData && nocturnalChartOptions"
            :data="nocturnalChartData"
            :options="nocturnalChartOptions"
          />
        </div>
        <div class="specific-graph-info">
          <p>{{ nocturnalData.length }} măsurători în mod Nocturnal</p>
        </div>
      </div>

      <!-- Heat Map for High BP Measurements -->
      <div v-if="highBPData.length > 0" class="specific-graph-container heat-map-graph">
        <h3>Heat Map al Măsurătorilor cu Tensiune Sistolică ≥ 140 mmHg</h3>
        <div class="specific-graph-wrapper">
          <Bubble
            v-if="heatMapData && heatMapOptions"
            :data="heatMapData"
            :options="heatMapOptions"
          />
        </div>
        <div class="specific-graph-info">
          <p>{{ highBPData.length }} măsurători cu tensiune sistolică ≥ 140 mmHg</p>
          <p>Distribuția măsurătorilor pe zile și ore</p>
        </div>
      </div>
    </div>

    <!-- Blood Pressure Statistics Summary -->
    <div v-if="!loading && !error" class="bp-stats-summary">
      <!-- Systolic Statistics -->
      <div class="stats-section">
        <h3>Tensiune Sistolică</h3>
        <div class="stats-container">
          <div v-for="threshold in bpStats.systolic.thresholds" :key="'sys-'+threshold" class="stat-item">
            <div class="stat-value">
              <span class="count">{{ bpStats.systolic.counts[threshold] }}</span>
              <span class="percentage">({{ bpStats.systolic.percentages[threshold] }}%)</span>
            </div>
            <div class="stat-label">
              {{ bpStats.systolic.counts[threshold] === 1 ? 'Măsurătoare' : '' }}
              <template v-if="threshold === bpStats.systolic.thresholds[0]">
                peste {{ threshold }} mmHg
              </template>
              <template v-else>
                între {{ threshold }} și {{ bpStats.systolic.thresholds[bpStats.systolic.thresholds.indexOf(threshold) - 1] }} mmHg
              </template>
            </div>
          </div>
        </div>
      </div>

      <!-- Diastolic Statistics -->
      <div class="stats-section">
        <h3>Tensiune Diastolică</h3>
        <div class="stats-container">
          <div v-for="threshold in bpStats.diastolic.thresholds" :key="'dia-'+threshold" class="stat-item">
            <div class="stat-value">
              <span class="count">{{ bpStats.diastolic.counts[threshold] }}</span>
              <span class="percentage">({{ bpStats.diastolic.percentages[threshold] }}%)</span>
            </div>
            <div class="stat-label">
              {{ bpStats.diastolic.counts[threshold] === 1 ? 'Măsurătoare' : '' }}
              <template v-if="threshold === bpStats.diastolic.thresholds[0]">
                peste {{ threshold }} mmHg
              </template>
              <template v-else>
                între {{ threshold }} și {{ bpStats.diastolic.thresholds[bpStats.diastolic.thresholds.indexOf(threshold) - 1] }} mmHg
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.bp-graph-container {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

h2 {
  text-align: center;
  margin-bottom: 10px;
  color: #333;
  font-size: 1.8rem;
}

.measurement-period {
  text-align: center;
  margin-bottom: 20px;
  color: #666;
  font-size: 1.1rem;
  font-style: italic;
}

.loading, .error {
  text-align: center;
  padding: 20px;
  font-size: 16px;
  margin: 20px 0;
  background-color: #f8f9fa;
  border-radius: 8px;
}

.error {
  color: #dc3545;
  background-color: #f8d7da;
  border: 1px solid #f5c6cb;
}

.graph-wrapper {
  width: 100%;
  height: 600px;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  position: relative;
  background-color: #ffffff;
  margin-bottom: 20px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  padding: 20px;
}

.bp-stats-summary {
  margin-top: 30px;
  padding: 20px;
  background-color: #f8f9fa;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.bp-stats-summary h3 {
  text-align: center;
  margin-bottom: 15px;
  color: #333;
  font-size: 1.5rem;
}

.stats-section {
  margin-bottom: 30px;
}

.stats-section:last-child {
  margin-bottom: 0;
}

.stats-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-around;
  gap: 15px;
}

.stat-item {
  flex: 1 1 150px;
  max-width: 200px;
  padding: 15px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  text-align: center;
  transition: transform 0.2s;
}

.stat-item:hover {
  transform: translateY(-3px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.stat-value {
  margin-bottom: 10px;
}

.count {
  font-size: 1.8rem;
  font-weight: bold;
  color: #d9534f;
  margin-right: 5px;
}

.percentage {
  font-size: 1.2rem;
  color: #666;
}

.stat-label {
  font-size: 0.9rem;
  color: #333;
}

.controls {
  margin-bottom: 20px;
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 20px;
}

.interval-control, .line-control {
  display: flex;
  align-items: center;
  background-color: #f8f9fa;
  padding: 10px 15px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.interval-control label, .line-control label {
  margin-right: 10px;
  font-weight: bold;
  color: #333;
}

.interval-control input, .line-control input {
  width: 60px;
  padding: 5px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  text-align: center;
}

.interval-control input:focus, .line-control input:focus {
  outline: none;
  border-color: #4a90e2;
  box-shadow: 0 0 0 2px rgba(74, 144, 226, 0.2);
}

.color-control {
  display: flex;
  flex-direction: column;
  gap: 10px;
  background-color: #f8f9fa;
  padding: 10px 15px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.color-picker {
  display: flex;
  align-items: center;
}

.color-picker label {
  margin-right: 10px;
  font-weight: bold;
  color: #333;
  min-width: 140px;
}

.color-picker input[type="color"] {
  width: 40px;
  height: 30px;
  padding: 0;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
}

.color-picker input[type="color"]:focus {
  outline: none;
  border-color: #4a90e2;
  box-shadow: 0 0 0 2px rgba(74, 144, 226, 0.2);
}

/* Controls header and toggle button */
.controls-header {
  display: flex;
  justify-content: center;
  margin-bottom: 10px;
}

.toggle-controls-btn {
  background-color: #4a90e2;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px 16px;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.3s;
}

.toggle-controls-btn:hover {
  background-color: #3a7bc8;
}

/* Collapsible controls */
.controls {
  max-height: 500px;
  overflow: hidden;
  transition: max-height 0.5s ease-in-out, opacity 0.5s ease-in-out, margin-bottom 0.5s ease-in-out;
  opacity: 1;
}

.controls.collapsed {
  max-height: 0;
  opacity: 0;
  margin-bottom: 0;
}

/* File selection controls */
.file-selection-control {
  display: flex;
  flex-direction: column;
  gap: 15px;
  background-color: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 600px;
  margin-bottom: 15px;
}

.file-dropdown, .file-upload {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.file-dropdown label, .file-upload label {
  font-weight: bold;
  color: #333;
}

.file-dropdown select {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  background-color: white;
  cursor: pointer;
}

.file-dropdown select:focus {
  outline: none;
  border-color: #4a90e2;
  box-shadow: 0 0 0 2px rgba(74, 144, 226, 0.2);
}

.file-dropdown select:disabled {
  background-color: #e9ecef;
  cursor: not-allowed;
}

.file-upload input[type="file"] {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: white;
  cursor: pointer;
}

.file-upload input[type="file"]:disabled {
  background-color: #e9ecef;
  cursor: not-allowed;
}

.uploaded-file-info {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 8px;
  padding: 8px 12px;
  background-color: #e9f7fe;
  border-radius: 4px;
  font-size: 0.9rem;
}

.uploaded-file-info button {
  background-color: #dc3545;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 4px 8px;
  font-size: 0.8rem;
  cursor: pointer;
  transition: background-color 0.3s;
}

.uploaded-file-info button:hover {
  background-color: #c82333;
}

/* Night measurement legend */
.night-measurement-legend {
  display: flex;
  justify-content: center;
  gap: 20px;
  margin: 10px 0 20px;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 5px;
}

.legend-symbol {
  font-size: 18px;
  line-height: 1;
}

.legend-symbol.night {
  color: #FF9900;
}

.legend-symbol.day {
  color: #FF0000;
}

.legend-text {
  font-size: 14px;
  color: #333;
}

/* Specific graphs */
.specific-graphs {
  margin-top: 40px;
  display: flex;
  flex-direction: column;
  gap: 40px;
}

.specific-graph-container {
  background-color: #f8f9fa;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.specific-graph-container h3 {
  text-align: center;
  margin-bottom: 15px;
  color: #333;
  font-size: 1.3rem;
}

.specific-graph-wrapper {
  height: 400px;
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  background-color: #ffffff;
  margin-bottom: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  padding: 10px;
}

.specific-graph-info {
  display: flex;
  justify-content: space-around;
  margin-top: 10px;
}

.specific-graph-info p {
  margin: 0;
  font-size: 14px;
  color: #666;
  font-weight: bold;
}

/* Nocturnal graph specific styles */
.nocturnal-graph {
  background-color: #f8f0ff; /* Light purple background */
  border-left: 4px solid #9900FF; /* Purple border */
}

.nocturnal-graph h3 {
  color: #9900FF; /* Purple text */
}

/* Heat map graph specific styles */
.heat-map-graph {
  background-color: #fff0f0; /* Light red background */
  border-left: 4px solid #FF0000; /* Red border */
}

.heat-map-graph h3 {
  color: #FF0000; /* Red text */
}
</style>
