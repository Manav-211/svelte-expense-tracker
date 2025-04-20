<script>
  import { onMount } from 'svelte';
  import { Chart } from 'chart.js/auto';

  let items = [];
  let newItem = {
    name: '',
    price: '',
    category: '',
    date: ''
  };
  let total = 0;
  let editingIndex = null;
  let chart = null;
  let chartCanvas = null;

  // Filters
  let filterCategory = '';
  let filterStartDate = '';
  let filterEndDate = '';

  // Theme
  let darkMode = true;

  onMount(() => {
    const stored = localStorage.getItem('expenses');
    if (stored) {
      items = JSON.parse(stored);
      calculateTotal();
      updateChart();
    }
  });

  const updateLocalStorage = () => {
    localStorage.setItem('expenses', JSON.stringify(items));
    calculateTotal();
    updateChart();
  };

  const calculateTotal = () => {
    total = items.reduce((acc, item) => acc + Number(item.price), 0);
  };

  const addItem = () => {
    if (newItem.name && newItem.price && newItem.category && newItem.date) {
      if (editingIndex !== null) {
        items[editingIndex] = { ...newItem };
        editingIndex = null;
      } else {
        items = [...items, { ...newItem }];
      }
      newItem = { name: '', price: '', category: '', date: '' };
      updateLocalStorage();
    }
  };

  const editItem = (index) => {
    newItem = { ...items[index] };
    editingIndex = index;
  };

  const deleteItem = (index) => {
  items.splice(index, 1);
  items = [...items]; // trigger reactivity
  updateLocalStorage();
};

  const filteredItems = () => {
    return items.filter((item) => {
      const categoryMatch = filterCategory ? item.category === filterCategory : true;
      const itemDate = new Date(item.date);
      const startDate = filterStartDate ? new Date(filterStartDate) : null;
      const endDate = filterEndDate ? new Date(filterEndDate) : null;
      const startMatch = startDate ? itemDate >= startDate : true;
      const endMatch = endDate ? itemDate <= endDate : true;
      return categoryMatch && startMatch && endMatch;
    });
  };

  const getCategoryBreakdown = () => {
    const breakdown = {};
    items.forEach((item) => {
      if (!breakdown[item.category]) breakdown[item.category] = 0;
      breakdown[item.category] += Number(item.price);
    });
    return breakdown;
  };

  const updateChart = () => {
    if (!chartCanvas || items.length === 0) return;

    const breakdown = getCategoryBreakdown();
    const categories = Object.keys(breakdown);
    const amounts = Object.values(breakdown);

    if (chart) chart.destroy();

    const backgroundColors = categories.map(() =>
      `hsl(${Math.floor(Math.random() * 360)}, 70%, 60%)`
    );

    chart = new Chart(chartCanvas, {
      type: 'pie',
      data: {
        labels: categories,
        datasets: [{
          data: amounts,
          backgroundColor: backgroundColors,
          borderWidth: 1
        }]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: {
            position: 'right',
            labels: {
              color: '#fff',
              font: {
                size: 14
              }
            }
          }
        }
      }
    });
  };
</script>

<main class={`min-h-screen p-6 sm:p-10 flex flex-col items-center transition-colors duration-300 ${darkMode ? 'bg-slate-900 text-white' : 'bg-white text-black'}`}>
  <div class='w-full max-w-4xl'>
    <div class="flex justify-between items-center mb-6">
      <h1 class='text-4xl font-bold text-center w-full'>💰 Expense Tracker</h1>
      <button on:click={() => darkMode = !darkMode} class="ml-auto p-2 bg-gray-300 rounded text-black hover:bg-gray-400">
        {darkMode ? '☀️ Light Mode' : '🌙 Dark Mode'}
      </button>
    </div>

    <!-- Form -->
    <form on:submit|preventDefault={addItem} class='grid grid-cols-6 gap-2 text-black mb-6'>
      <input bind:value={newItem.name} class='col-span-2 p-3 rounded border' type='text' placeholder='Title' />
      <input bind:value={newItem.price} class='col-span-1 p-3 rounded border' type='number' placeholder='Amount' />
      <select bind:value={newItem.category} class='col-span-1 p-3 rounded border text-black'>
        <option value="" disabled selected>Select Category</option>
        <option value="Food">Food</option>
        <option value="Transport">Transport</option>
        <option value="Housing">Housing</option>
        <option value="Entertainment">Entertainment</option>
        <option value="Utilities">Utilities</option>
        <option value="Other">Other</option>
      </select>
      <input bind:value={newItem.date} class='col-span-1 p-3 rounded border' type='date' />
      <button class='bg-blue-600 hover:bg-blue-500 text-white p-3 rounded col-span-1'>
        {editingIndex !== null ? 'Update' : 'Add'}
      </button>
    </form>

    <!-- Filters -->
    <div class='flex flex-wrap gap-4 mb-6'>
      <select bind:value={filterCategory} class='p-2 rounded text-black'>
        <option value=''>All Categories</option>
        {#each Array.from(new Set(items.map(item => item.category))) as cat}
          <option value={cat}>{cat}</option>
        {/each}
      </select>
      <input bind:value={filterStartDate} class='p-2 rounded text-black' type='date' />
      <input bind:value={filterEndDate} class='p-2 rounded text-black' type='date' />
    </div>

    {#if items.length}
      <!-- Chart Section -->
      <div class={`p-4 rounded mb-6 ${darkMode ? 'bg-slate-800' : 'bg-gray-300'}`}>
        <h2 class='font-semibold mb-4 text-center text-xl'>Expense Distribution</h2>
        <div class='max-h-96'>
          <canvas bind:this={chartCanvas}></canvas>
        </div>
      </div>

      <!-- Expense List -->
      <ul class='space-y-2 mb-6'>
        {#each filteredItems() as item, i}
          <li class={`flex items-center justify-between p-4 rounded ${darkMode ? 'bg-slate-800' : 'bg-gray-100'}`}>
            <div class='grid grid-cols-4 gap-4 w-full'>
              <span>{item.name}</span>
              <span>${item.price}</span>
              <span>{item.category}</span>
              <span>{item.date}</span>
            </div>
            <div class='flex gap-2 ml-4'>
              <button on:click={() => editItem(i)} class='text-yellow-500 hover:text-yellow-400'>✏️</button>
              <button on:click={() => deleteItem(i)} class='text-red-500 hover:text-red-400'>🗑️</button>
            </div>
          </li>
        {/each}
      </ul>

      <!-- Totals & Breakdown -->
      <div class={`p-4 rounded ${darkMode ? 'bg-slate-800' : 'bg-gray-100'}`}>
        <div class='flex justify-between mb-2 font-bold'>
          <span>Total Expenses</span>
          <span>${total}</span>
        </div>
        <div>
          <h2 class='font-semibold mb-2'>Category Breakdown:</h2>
          <ul>
            {#each Object.entries(getCategoryBreakdown()) as [cat, amt]}
              <li class='flex justify-between'>
                <span>{cat}</span>
                <span>${amt}</span>
              </li>
            {/each}
          </ul>
        </div>
      </div>
    {:else}
      <div class='text-center py-10 text-gray-400'>
        No expenses added yet. Start by adding one above!
      </div>
    {/if}
  </div>
</main>
