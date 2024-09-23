<script lang="ts">
    import { onMount, onDestroy } from 'svelte';
    import { writable } from 'svelte/store';
    import type { SMSSItem } from '$lib/components/SMSS'; // Adjust the path as needed
    import img from '$lib/components/images/produce.jpg';
    import logo from '$lib/components/images/Massy_Distribution_Logo_No_BG2.png';
    import Papa from 'papaparse'; 
    import  type { ParseResult } from 'papaparse';  // Import PapaParse and types



    let smssData = writable<SMSSItem[]>([]);
    let openMenuId = writable<number | null>(null); // To track which menu is open
    let openPopupView = writable<string | null>(null); // To track which popup view is open
    let isFilterDropdownOpen = writable(false); // To track filter dropdown state
    let isactionsDropdown = writable(false); // To track filter dropdown state
    let parsedCSVData: SMSSItem[] = [];
    let uploadedFile: File | null = null; // Store the uploaded file
    let csvPreviewData = writable<SMSSItem[]>([]); // Data preview from CSV

    let logoutTimeout: ReturnType<typeof setTimeout>; // For the 59 minutes timeout
    
    let showPopup = writable(false);
    //let currentItem = writable<SMSSItem | null>(null);
        let currentItem = writable<SMSSItem>({
        salesrep: '',
        year: 0,
        fmth: 0,
        cmth: 0,
        gpBudget: 0,
        site: ''
    });
   // Reactive statement to calculate fmth based on cmth
   $: {
        const item = $currentItem;
        $currentItem = {
            ...item,
            fmth: calculateFinancialMonth(item.cmth)
        };
    }

    function calculateFinancialMonth(cmth: number): number {
        const newMonth = ((cmth + 3 - 1) % 12 + 12) % 12 + 1;
        console.log(`Current Month: ${cmth}, +3: ${cmth + 3}, Result: ${newMonth}`);
        return newMonth;
    }

    let searchTerm = writable('');
    let searchYear=writable<number | null>(null);
    let searchFmth=writable<number | null>(null);


    const selectedRows = writable<boolean[]>([]);
    let token = writable<string | null>(null);

    onMount(async () => {
        // Get token from localStorage
        const storedToken = localStorage.getItem('authToken');
        token.set(storedToken);
        // Accessing the current value stored in the writable store
        let currentToken: string | null = null; // Initialize currentToken with a default value
        token.subscribe(value => {
            currentToken = value;
        });
        console.log("token",currentToken);
        if (!currentToken || currentToken ==null) {
            // No token present, redirect to login
            window.location.href = '/';
            return;
        }
        
        // Set timeout to log the user out after 59 minutes
        logoutTimeout = setTimeout(() => {
            logoutUser();
        }, 59 * 60 * 1000); // 59 minutes in milliseconds
        try {
            const response = await fetch('https://localhost:7058/api/SMSS', {
                method: 'GET',
                headers: {
                   // 'Authorization': `Bearer ${currentToken}`, // Include token in Authorization header
                    'Content-Type': 'application/json'
                },
                credentials: 'include' // Include credentials for Windows Authentication
            });
            
            if (!response.ok) {
                //logoutUser();
                throw new Error('Failed to fetch data from the API');
            }

            const data = await response.json();
            smssData.set(data);
        } catch (error) {
            console.error('Error fetching data:', error);
        }

       // Event listener for clicks outside the menu
       const handleClickOutside = (event: MouseEvent) => {
            const popup = document.querySelector('.popup-container');
            const menu = document.querySelector('.menu-container');
            const button = event.target as HTMLElement;

            // Check if the click is inside the menu or button
            if ((menu && !menu.contains(button) && !button.classList.contains('open-menu-button')) ||
            (popup && !popup.contains(button))) {
                openMenuId.set(null);
                //openPopupView.set(null);
            }
        };

        document.addEventListener('click', handleClickOutside);

        // Clean up event listener on component destroy
        onDestroy(() => {
            document.removeEventListener('click', handleClickOutside);
            clearTimeout(logoutTimeout);
        });
    });

    function openPopup(id: number, event: MouseEvent,item: SMSSItem) {
        event.stopPropagation(); // Prevent the event from bubbling up to the document
        console.log(item); 
        openMenuId.set(id); // Set the ID of the item to open its menu
        openPopupView.set(null); // Reset popup view
    }

    function openView(view: string,item: SMSSItem |null) {
        openPopupView.set(view);
        currentItem.set(item || {
            salesrep: '',
            year: 0,
            fmth: 0,
            cmth: 0,
            gpBudget: 0,
            site: ''
        });
    }

    // Logout function
    function logoutUser() {
        // Clear token and redirect to login
        localStorage.removeItem('authToken');
        token.set(null);
        window.location.href = '/'; // Adjust your login URL as necessary
        console.log('User logged out after 59 minutes.');
    }

    async function handleDelete() {
         // Retrieve the token asynchronously
        let currentToken: string | null = null;
        token.subscribe(value => {
            currentToken = value;
        })();
        if (currentToken) {
console.log("create: " + currentToken);
        }
        
        const item = $currentItem;
        if (item) {
            const url = `https://localhost:7058/api/SMSS/${item.salesrep}/${item.year}/${item.cmth}`;
            try {
                const response = await fetch(url, {
                    method: 'DELETE',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${currentToken}`  // Include the JWT token
                    },
                });

                if (response.ok) {
                    // Optionally, refresh data or update the UI
                    console.log('Item deleted successfully');
                    // Remove the item from local state or refetch the data
                    const updatedData = $smssData.filter(i => !(i.salesrep === item.salesrep && i.year === item.year && i.cmth === item.cmth));
                    smssData.set(updatedData);
                } else {
                    console.error('Failed to delete item');
                }
            } catch (error) {
                console.error('Error deleting item:', error);
            }
        }
        closePopup(); // Close the popup after deletion
    }

    async function handleUpdate() {
         // Retrieve the token asynchronously
         let currentToken: string | null = null;
        token.subscribe(value => {
            currentToken = value;
        })();
        if (currentToken) {
console.log("create: " + currentToken);
        }
        

        const item = $currentItem;
        if (item) {
            const url = `https://localhost:7058/api/SMSS/${item.salesrep}/${item.year}/${item.cmth}`;
            try {
                const response = await fetch(url, {
                    method: 'PUT',
                    headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${currentToken}`  // Include the JWT token
                },
                body: JSON.stringify(item),
                });

                if (response.ok) {
                    // Optionally, refresh data or update the UI
                    console.log('Item updated successfully');
                    // Remove the item from local state or refetch the data
                    const updatedData = $smssData.filter(i => !(i.salesrep === item.salesrep && i.year === item.year && i.cmth === item.cmth));
                    smssData.set(updatedData);
                } else {
                    console.error('Failed to update item');
                }
            } catch (error) {
                console.error('Error updating item:', error);
            }
        }
        closePopup(); // Close the popup after deletion
    }

    async function handleCreate() {
        // Retrieve the token asynchronously
        let currentToken: string | null = null;
        token.subscribe(value => {
            currentToken = value;
        })();
        if (currentToken) {
console.log("create: " + currentToken);
        }
        
        const item = $currentItem;
        if (item) {
            try {
                const response = await fetch('https://localhost:7058/api/SMSS', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${currentToken}`  // Include the JWT token Authorization  `Bearer ${currentToken}`
                    },
                    body: JSON.stringify(item),
                });

                if (response.ok) {
                    // Optionally, refresh data or update the UI
                    console.log('Item created successfully');
                    // Fetch and update the data
                    const updatedData = await (await fetch('https://localhost:7058/api/SMSS')).json();
                    smssData.set(updatedData);
                } else {
                    console.error('Failed to create item');
                }
            } catch (error) {
                console.error('Error creating item:', error);
            }
        }
        closePopup(); // Close the popup after creation
    }

    // Function to save changes
    async function handleBulkEdit() {
        // Get the data from the form
        const data = $smssData.map(item => ({
        salesrep: item.salesrep,
        year: item.year,
        fmth: item.fmth,
        cmth: item.cmth,
        gpBudget: item.gpBudget,
        site: item.site
        }));

        // Retrieve the token asynchronously
        let currentToken: string | null = null;
        token.subscribe(value => {
            currentToken = value;
        })();
        if (currentToken) {
console.log("create: " + currentToken);
        }
        
        const item = $currentItem;

        try {
        // Make a POST request to the API
        const response = await fetch('https://localhost:7058/api/SMSS/bulk', {
            method: 'POST',
            headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${currentToken}`  // Include the JWT token
            },
            body: JSON.stringify(data)
        });

        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        // Handle successful response
        const result = await response.text();
        console.log('Success:', result);
        closePopup(); // Close the popup on success
        } catch (error) {
        // Handle error
        console.error('Error:', error);
        }
    }

    // Function to delete selected rows
  async function HandleBulkDelete() {
    // Subscribe to smssData and selectedRows to get their values
    let data: SMSSItem[] = [];
    let selections: boolean[] = [];
    
    smssData.subscribe(value => data = value)();
    selectedRows.subscribe(value => selections = value)();

    // Filter the data based on selectedRows
    const selectedData = data.filter((_, index) => selections[index]);

    if (selectedData.length === 0) {
      alert('No rows selected for deletion');
      return;
    }

     // Retrieve the token asynchronously
     let currentToken: string | null = null;
        token.subscribe(value => {
            currentToken = value;
        })();
        if (currentToken) {
console.log("create: " + currentToken);
        }
    try {
      // Make a DELETE request to the API
      const response = await fetch('https://localhost:7058/api/SMSS/bulk', {
        method: 'DELETE',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${currentToken}`  // Include the JWT token
        },
        body: JSON.stringify(selectedData)
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      // Handle successful response
      const result = await response.text();
      console.log('Success:', result);
      closePopup(); // Close the popup on success
    } catch (error) {
      // Handle error
      console.error('Error:', error);
    }
  }

  function handleFileChange(event: Event) {
    const fileInput = event.target as HTMLInputElement;
    if (fileInput.files && fileInput.files.length > 0) {
        uploadedFile = fileInput.files[0]; // Store the selected file

        // Read the uploaded CSV file
        const reader = new FileReader();
        reader.onload = (e: ProgressEvent<FileReader>) => {
            if (e.target && e.target.result) {
                const csvString = e.target.result as string;

                // Parse CSV string into SMSSItem[]
                parsedCSVData = parseCSV(csvString);
                let parsedCSVData1: SMSSItem[] ;
                // Update preview data in writable store
                csvPreviewData.set(parsedCSVData);
                console.log("csvPreviewData, ",csvPreviewData);
                console.log("parsedCSVData, ",parsedCSVData);
                const unsubscribe = csvPreviewData.subscribe(value => {
                    parsedCSVData1 = value;
                    parsedCSVData = value;
                    // Logging the token value to console
                    console.log("parsedCSVData1", parsedCSVData1); // Logs "token" followed by the current value of the token
                    
                    
                });
                
              
            }
        };
        reader.readAsText(uploadedFile); // Read file as text
    }
}

    // Example function to create a sample SMSSItem object for header validation
    function sampleSMSSItem(): SMSSItem {
        return {
            salesrep: 'Sample Sales Rep',
            year: 2023,
            fmth: 1,
            cmth: 1,
            gpBudget: 10000.0,
            site: 'Sample Site',
        };
    }

    // Function to parse CSV string into SMSSItem[]
    function parseCSV(csvString: string): SMSSItem[] {
        const lines = csvString.split('\n');
        const headers = lines[0].split(',');

        // Ensure headers match expected structure
        if (headers.length !== Object.keys(sampleSMSSItem()).length) {
            throw new Error('CSV format does not match expected structure.');
        }

        const data: SMSSItem[] = [];

        for (let i = 1; i < lines.length; i++) {
            const fields = lines[i].split(',');

            if (fields.length === headers.length) {
                const item: SMSSItem = {
                    salesrep: fields[0].trim(),
                    year: parseInt(fields[1], 10),
                    fmth: parseInt(fields[2], 10),
                    cmth: parseInt(fields[3], 10),
                    gpBudget: parseFloat(fields[4]),
                    site: fields[5].trim(),
                };
                data.push(item);
            }
        }

        return data;
    }

    // Function to handle CSV upload (after preview)
    async function uploadCSVData() {
        console.log("csvPreviewData, ",csvPreviewData);
        console.log("parsedCSVData, ",parsedCSVData);

         // Retrieve the token asynchronously
         let currentToken: string | null = null;
        token.subscribe(value => {
            currentToken = value;
        })();
        if (currentToken) {
console.log("create: " + currentToken);
        }

        
        if (parsedCSVData.length > 0) {
            try {
                const response = await fetch('https://localhost:7058/api/SMSS/bulk', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${localStorage.getItem('authToken')}` // Include token if needed
                    },
                    body: JSON.stringify(parsedCSVData)
                });

                if (!response.ok) {
                    throw new Error('Failed to upload CSV data');
                }

                // Fetch updated data
                const updatedData = await (await fetch('https://localhost:7058/api/SMSS')).json();
                smssData.set(updatedData); // Update SMSS data
                openPopupView.set(null); // Close the popup
            } catch (error) {
                console.error('Error uploading CSV data:', error);
            }
        }
    }

    function closePopup() {
        openMenuId.set(null); // Close the menu
        openPopupView.set(null); // Close the popup view
        console.log('close pop up: ', openPopupView)
    }

    function filterData(data: SMSSItem[], searchTerm: string, searchYear: number | null, searchFmth: number | null): SMSSItem[] {
    console.log(searchTerm, searchYear);
    
    // Filter the data based on searchTerm, searchYear, and searchFmth
    const filteredData = data.filter(item => 
        item.salesrep.toLowerCase().includes(searchTerm.toLowerCase()) &&
        (searchYear === null || item.year === searchYear) &&
        (searchFmth === null || item.fmth === searchFmth)
    );
    
    // Sort the filtered data by year in descending order (most recent first)
    return filteredData.sort((a, b) => b.year - a.year);
}


    // function performSearch() {
    //     $smssData = filterData($smssData, $searchTerm);
    // }

    function toggleFilterDropdown() {
        isFilterDropdownOpen.update(value => !value);
    }

    function toggleFilterActionsDropdown() {
        isactionsDropdown.update(value => !value);
    }

    function applyFilter(filterOption: string) {
        // Logic to apply filter based on the selected filter option
        console.log('Applying filter:', filterOption);
        isFilterDropdownOpen.set(false); // Close the dropdown after applying the filter
    }


  // Initialize selectedRows based on smssData length
  smssData.subscribe(data => {
    selectedRows.set(new Array(data.length).fill(false));
  });

  // Function to handle row selection
  function handleSelectAll(e: Event) {
    const target = e.target as HTMLInputElement;
    if (target) {
      const checked = target.checked;
      selectedRows.update(rows => rows.map(() => checked));
    }
  }

  // Function to handle individual row selection
  function toggleRowSelection(index: number) {
    selectedRows.update(rows => {
      const newRows = [...rows];
      newRows[index] = !newRows[index];
      return newRows;
    });
  }

</script>
<div class="relative h-screen flex flex-col justify-center mx-auto px-4 lg:max-w-none lg:px-0 bg-cover bg-center bd-orange-400">
    <div class=" relative min-h-screen flex flex-col bg-center " style="background-image: url({img});background-color: orange;">
        <div class="absolute inset-0 bg-gray-300 bg-opacity-30">
            <section class="relative z-10 dark:bg-gray-900 p-3 sm:p-5 antialiased " style="background-image: url({img});background-color: orange;">
                <div class="mx-auto max-w-screen-xl px-4 lg:px-12">
                    <!-- Start coding here -->
                    <div class="bg-white dark:bg-gray-800 relative shadow-md sm:rounded-lg overflow-hidden">
                        <div class="flex flex-col md:flex-row items-center justify-between space-y-3 md:space-y-0 md:space-x-4 p-4">
                            <div class="">
                                <img src="{logo}" alt="Logo" class="h-24 top-0 ">
                            </div>
                            <div class="w-full md:w-1/2">
                                <form class="flex items-center">
                                    <label for="simple-search" class="sr-only">Search</label>
                                    <div class="relative w-full">
                                        <div class="absolute inset-y-0 left-0 flex items-center pl-3 pointer-events-none">
                                            <svg aria-hidden="true" class="w-5 h-5 text-gray-500 dark:text-gray-400" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg">
                                                <path fill-rule="evenodd" d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z" clip-rule="evenodd" />
                                            </svg>
                                        </div>
                                        <input type="text" id="simple-search" 
                                        class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                        placeholder="Search" 
                                        bind:value={$searchTerm}
                                        />
                                    </div>
                                </form>
                            </div>
                            <div class="w-full md:w-auto flex flex-col md:flex-row space-y-2 md:space-y-0 items-stretch md:items-center justify-end md:space-x-3 flex-shrink-0">
                                <button type="button" on:click={() => openView('create',null)} class="flex items-center justify-center text-white bg-primary-700 hover:bg-primary-800 focus:ring-4 focus:ring-primary-300 font-medium rounded-lg text-sm px-4 py-2 dark:bg-primary-600 dark:hover:bg-primary-700 focus:outline-none dark:focus:ring-primary-800">
                                    <svg class="h-3.5 w-3.5 mr-2" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
                                        <path clip-rule="evenodd" fill-rule="evenodd" d="M10 3a1 1 0 011 1v5h5a1 1 0 110 2h-5v5a1 1 0 11-2 0v-5H4a1 1 0 110-2h5V4a1 1 0 011-1z" />
                                    </svg>
                                    Add Item
                                </button>
                                <div class="flex items-center space-x-3 w-full md:w-auto">
                                    <button id="actionsDropdownButton" on:click={toggleFilterActionsDropdown} data-dropdown-toggle="actionsDropdown" class="w-full md:w-auto flex items-center justify-center py-2 px-4 text-sm font-medium text-gray-900 focus:outline-none bg-white rounded-lg border border-gray-200 hover:bg-gray-100 hover:text-primary-700 focus:z-10 focus:ring-4 focus:ring-gray-200 dark:focus:ring-gray-700 dark:bg-gray-800 dark:text-gray-400 dark:border-gray-600 dark:hover:text-white dark:hover:bg-gray-700" type="button">
                                        <svg class="-ml-1 mr-1.5 w-5 h-5" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
                                            <path clip-rule="evenodd" fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" />
                                        </svg>
                                        Actions
                                    </button>
                                    {#if $isactionsDropdown}
                                    <div id="actionsDropdown" class=" z-10 w-44 bg-white rounded divide-y divide-gray-100 shadow dark:bg-gray-700 dark:divide-gray-600">
                                        <div id="actionsDropdown" class="z-10  w-56 p-3 bg-white rounded-lg shadow dark:bg-gray-700">
                                            <ul class="space-y-2 text-sm" aria-labelledby="filterDropdownButton">
                                                <li class="flex items-center">
                                                    <button type="button" on:click={() => openView('Bulk Edit',null)} class="flex w-full items-center py-2 px-4 hover:bg-gray-100 dark:hover:bg-gray-600 text-red-500 dark:hover:text-red-400">
                                                        Bulk Edit
                                                    </button>
                                                </li>
                                                <li class="flex items-center">
                                                    <button type="button" on:click={() => openView('Bulk Delete',null)} class="flex w-full items-center py-2 px-4 hover:bg-gray-100 dark:hover:bg-gray-600 text-red-500 dark:hover:text-red-400">
                                                        Bulk Delete
                                                    </button>
                                                </li>
                                                <li class="flex items-center">
                                                    <button type="button" on:click={() => openView('FileUpload',null)} class="flex w-full items-center py-2 px-4 hover:bg-gray-100 dark:hover:bg-gray-600 text-red-500 dark:hover:text-red-400">
                                                        Upload File
                                                    </button>
                                                </li>
                                            </ul>
                                        </div>
                                    </div>
                                    {/if}
                                    <button id="filterDropdownButton" on:click={toggleFilterDropdown} data-dropdown-toggle="filterDropdown" class="w-full md:w-auto flex items-center justify-center py-2 px-4 text-sm font-medium text-gray-900 focus:outline-none bg-white rounded-lg border border-gray-200 hover:bg-gray-100 hover:text-primary-700 focus:z-10 focus:ring-4 focus:ring-gray-200 dark:focus:ring-gray-700 dark:bg-gray-800 dark:text-gray-400 dark:border-gray-600 dark:hover:text-white dark:hover:bg-gray-700" type="button">
                                        <svg xmlns="http://www.w3.org/2000/svg" aria-hidden="true" class="h-4 w-4 mr-2 text-gray-400" viewbox="0 0 20 20" fill="currentColor">
                                            <path fill-rule="evenodd" d="M3 3a1 1 0 011-1h12a1 1 0 011 1v3a1 1 0 01-.293.707L12 11.414V15a1 1 0 01-.293.707l-2 2A1 1 0 018 17v-5.586L3.293 6.707A1 1 0 013 6V3z" clip-rule="evenodd" />
                                        </svg>
                                        Filter
                                        <svg class="-mr-1 ml-1.5 w-5 h-5" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
                                            <path clip-rule="evenodd" fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" />
                                        </svg>
                                    </button>
                                    {#if $isFilterDropdownOpen}
                                        <div id="filterDropdown" class="z-10  w-56 p-3 bg-white rounded-lg shadow dark:bg-gray-700">
                                            <ul class="space-y-2 text-sm" aria-labelledby="filterDropdownButton">
                                                <li class="flex items-center">
                                                    <form class="flex items-center">
                                                        <div class="relative w-full">
                                                            <input type="number" id="simple-search" 
                                                            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                                            placeholder="Year" 
                                                            bind:value={$searchYear}
                                                            />
                                                        </div>
                                                    </form>
                                                </li>
                                                <li class="flex items-center">
                                                    <form class="flex items-center">
                                                        <div class="relative w-full">
                                                            <input type="number" id="simple-search" 
                                                            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                                            placeholder="Fmth" 
                                                            bind:value={$searchFmth}
                                                            />
                                                        </div>
                                                    </form>
                                                </li>
                                            </ul>
                                        </div>
                                    {/if}
                                    <button type="button" on:click={() => logoutUser()} class="flex items-center justify-center text-white bg-gray-600 hover:bg-gray-800 focus:ring-4 focus:ring-primary-300 font-medium rounded-lg text-sm px-4 py-2 dark:bg-primary-600 dark:hover:bg-primary-700 focus:outline-none dark:focus:ring-primary-800">      
                                        Log out
                                    </button>
                                </div>
                            </div>
                        </div>
                <div class="mx-auto max-w-screen-xl px-4 lg:px-12">
                    <!-- Start coding here -->
                    <div class="overflow-x-auto">
                        <table class="text-center w-full text-sm text-left text-gray-500 dark:text-gray-400">
                            <thead class="text-xs text-gray-700 uppercase bg-gray-50 dark:bg-gray-700 dark:text-gray-400">
                                <tr>
                                    <th scope="col" class="px-4 py-4">Sales Rep</th>
                                    <th scope="col" class="px-4 py-3">Year</th>
                                    <th scope="col" class="px-4 py-3">Fiscal Month</th>
                                    <th scope="col" class="px-4 py-3">Current Month</th>
                                    <th scope="col" class="px-4 py-3">GP Budget</th>
                                    <th scope="col" class="px-4 py-3">Site</th>
                                    <th scope="col" class="px-4 py-3">
                                        <span class="sr-only">Actions</span>
                                    </th>
                                </tr>
                            </thead>
                            <tbody>
                                {#each filterData($smssData, $searchTerm, $searchYear, $searchFmth) as item, index}
                                <tr class="border-b dark:border-gray-700">
                                    <th scope="row" class="px-4 py-3 font-medium text-gray-900 whitespace-nowrap dark:text-white">{item.salesrep}</th>
                                    <td class=" px-4 py-3">20{item.year}</td>
                                    <td class="px-4 py-3">{item.fmth}</td>
                                    <td class="px-4 py-3">{item.cmth}</td>
                                    <td class="px-4 py-3">{item.gpBudget}</td>
                                    <td class="px-4 py-3 max-w-[12rem] truncate">{item.site}</td>
                                    <td class="px-4 py-3 flex items-center justify-end">
                                        <button 
                                            class="text-blue-500 hover:underline open-menu-button" 
                                            on:click={(event) => openPopup(index, event, item)}
                                        >
                                        <svg class="w-5 h-5" aria-hidden="true" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg">
                                            <path d="M6 10a2 2 0 11-4 0 2 2 0 014 0zM12 10a2 2 0 11-4 0 2 2 0 014 0zM16 12a2 2 0 100-4 2 2 0 000 4z" />
                                        </svg>
                                        </button>
                                        {#if $openMenuId === index}
                                            <div class="menu-container absolute right-0 z-10 w-44 bg-white rounded divide-y divide-gray-100 shadow dark:bg-gray-700 dark:divide-gray-600">
                                                <ul class="py-1 text-sm">
                                                    <li>
                                                        <button type="button" on:click={() => openView('edit',item)} class="flex w-full items-center py-2 px-4 hover:bg-gray-100 dark:hover:bg-gray-600 dark:hover:text-white text-gray-700 dark:text-gray-200">
                                                            Edit
                                                        </button>
                                                    </li>
                                                    <li>
                                                        <button type="button" on:click={() => openView('preview',item)} class="flex w-full items-center py-2 px-4 hover:bg-gray-100 dark:hover:bg-gray-600 dark:hover:text-white text-gray-700 dark:text-gray-200">
                                                            Preview
                                                        </button>
                                                    </li>
                                                    <li>
                                                        <button type="button" on:click={() => openView('delete',item)} class="flex w-full items-center py-2 px-4 hover:bg-gray-100 dark:hover:bg-gray-600 text-red-500 dark:hover:text-red-400">
                                                            Delete
                                                        </button>
                                                    </li>
                                                </ul>
                                            </div>
                                        {/if}
                                    </td>
                                </tr>
                                {/each}
                            </tbody>
                        </table>
                    </div>
                </div>
                <div class="mx-auto max-w-screen-xl px-4 lg:px-12">
                    <div class="text-center bg-white dark:bg-gray-800 relative shadow-md sm:rounded-lg overflow-hidden">
                        <h3 class="m-6 underline">Sales Commisions <span class="">&bull;</span> SMMS </h3>
                    </div>
                </div>

                {#if $openPopupView === 'create'}
                    <div class="popup-container fixed inset-0 z-20 flex items-center justify-center bg-gray-900 bg-opacity-30">
                        <div class="bg-white py-6 px-20 rounded shadow-lg dark:bg-gray-800">
                            <h2 class="text-lg font-bold m-2 mb-4">Create New Item</h2>
                            <form on:submit|preventDefault={handleCreate}>
                                <div class="mb-4">
                                    <label for="salesrep" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Sales Rep</label>
                                    <input id="salesrep" type="text" bind:value={$currentItem.salesrep} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                </div>
                                <div class="mb-4">
                                    <label for="year" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Year</label>
                                    <input id="year" type="number" bind:value={$currentItem.year} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                </div>
                                <!-- <div class="mb-4">
                                    <label for="fmth" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Financial Month</label>
                                    <input id="fmth" type="text" bind:value={$currentItem.fmth} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                </div> -->
                                <div class="mb-4">
                                    <label for="cmth" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Current Month</label>
                                    <input id="cmth" type="text" bind:value={$currentItem.cmth} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                </div>
                                <div class="mb-4">
                                    <label for="gpBudget" class="block text-sm font-medium text-gray-700 dark:text-gray-200">GP Budget</label>
                                    <input id="gpBudget" type="number" bind:value={$currentItem.gpBudget} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                </div>
                                <div class="mb-4">
                                    <label for="site" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Site</label>
                                    <input id="site" type="text" bind:value={$currentItem.site} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                </div>
                                <button type="submit" class="mt-4 px-4 py-2 bg-blue-500 text-white rounded">Create</button>
                                <button type="button" on:click={closePopup} class="mt-4 px-4 py-2 bg-gray-500 text-white rounded">Close</button>
                            </form>
                        </div>
                    </div>
                {/if}

                {#if $openPopupView === 'edit'}
                        <div class="popup-container fixed inset-0 z-20 flex items-center justify-center bg-gray-900 bg-opacity-30">
                            <div class="bg-white py-6 px-20 rounded shadow-lg dark:bg-gray-800">
                                <h2 class="text-lg font-bold m-2 mb-4">Edit Item</h2>
                                {#if $currentItem}
                                    <form on:submit|preventDefault={handleUpdate}>
                                        <div class="mb-4">
                                            <label for="salesrep" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Sales Rep</label>
                                            <input id="salesrep" type="text" bind:value={$currentItem.salesrep} class="bg-gray-200 mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" readonly /> 
                                        </div>
                                        <div class="mb-4">
                                            <label for="year" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Year</label>
                                            <input id="year" type="number" bind:value={$currentItem.year} class="bg-gray-200 mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" readonly /> 
                                        </div>
                                        <!-- <div class="mb-4">
                                            <label for="fmth" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Financial Month</label>
                                            <input id="fmth" type="text" value={$currentItem.fmth} class="bg-gray-200 mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" readonly />
                                        </div> -->
                                        <div class="mb-4">
                                            <label for="cmth" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Current Month</label>
                                            <input id="cmth" type="text" bind:value={$currentItem.cmth} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                        </div>
                                        <div class="mb-4">
                                            <label for="gpBudget" class="block text-sm font-medium text-gray-700 dark:text-gray-200">GP Budget</label>
                                            <input id="gpBudget" type="number" bind:value={$currentItem.gpBudget} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                        </div>
                                        <div class="mb-4">
                                            <label for="site" class="block text-sm font-medium text-gray-700 dark:text-gray-200">Site</label>
                                            <input id="site" type="text" bind:value={$currentItem.site} class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm dark:bg-gray-900 dark:border-gray-600 dark:text-gray-300" />
                                        </div>
                                        <button type="submit" class="mt-4 px-4 py-2 bg-blue-500 text-white rounded">Update</button>
                                        <button type="button" on:click={closePopup} class="mt-4 px-4 py-2 bg-gray-500 text-white rounded">Close</button>
                                    </form>
                                {/if}
                            </div>
                        </div>
                    {/if}

                    {#if $openPopupView === 'preview'}
                    <div class="popup-container fixed inset-0 z-20 flex items-center justify-center bg-gray-900 bg-opacity-30">
                        <div class="text-center items-center justify-center bg-white p-6 px-20 rounded shadow-lg dark:bg-gray-800">
                            <h2 class="text-lg font-bold mb-4">{$currentItem.salesrep}</h2>
                            <!-- Content to preview -->
                            {#if $currentItem}
                                <table class="text-left text-gray-800 m-4 min-w-full bg-white dark:bg-gray-800 border-collapse border border-gray-300">
                                    <tbody>
                                        <!-- <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Sales Rep</td>
                                            <td>{$currentItem.salesrep}</td>
                                        </tr> -->
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Year</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.year}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Financial Month</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.fmth}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Current Year</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.cmth}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">GP Budget</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.gpBudget}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Site</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.site}</td>
                                        </tr>
                                    </tbody>
                                </table>
                            {/if}
                            <button on:click={closePopup} class="mt-4 px-4 py-2 bg-gray-500 text-white rounded">Close</button>
                        </div>
                    </div>
                {/if}
                

                {#if $openPopupView === 'delete'}
                    <div class="popup-container fixed inset-0 z-20 flex items-center justify-center bg-gray-900 bg-opacity-30">
                        <div class="text-center items-center justify-center bg-white p-6 px-20 rounded shadow-lg dark:bg-gray-800">
                            <h2 class="text-lg font-bold mb-4">{$currentItem.salesrep}</h2>
                            <!-- Content to preview -->
                            {#if $currentItem}
                                <table class="text-left text-gray-800 m-4 min-w-full bg-white dark:bg-gray-800 border-collapse border border-gray-300">
                                    <tbody>
                                        <!-- <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Sales Rep</td>
                                            <td>{$currentItem.salesrep}</td>
                                        </tr> -->
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Year</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.year}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Financial Month</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.fmth}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Current Year</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.cmth}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">GP Budget</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.gpBudget}</td>
                                        </tr>
                                        <tr class="border border-gray-300">
                                            <td  class="font-bold border border-gray-300 p-2">Site</td>
                                            <td class="text-center border border-gray-300 p-2">{$currentItem.site}</td>
                                        </tr>
                                    </tbody>
                                </table>
                            {/if}
                            <button on:click={handleDelete} class="mt-4 px-4 py-2 bg-red-500 text-white rounded">Delete</button>
                            <button on:click={closePopup} class="mt-4 px-4 py-2 bg-gray-500 text-white rounded">Cancel</button>
                        </div>
                    </div>
                {/if}
                

            <!-- Bulk Edit Popup -->
            {#if $openPopupView === 'Bulk Edit'}
            <div class="popup-container fixed inset-0 z-20 flex items-center justify-center bg-gray-900 bg-opacity-30">
                <div class="bg-white p-6 rounded shadow-lg dark:bg-gray-800 max-h-screen overflow-auto">
                    <div class="mx-auto max-w-screen-xl px-4 lg:px-12">
                        <!-- Start coding here -->
                        <div class="bg-orange-600 dark:bg-gray-800 relative shadow-md sm:rounded-lg overflow-hidden">
                            <div class="flex flex-col md:flex-row items-center justify-between space-y-3 md:space-y-0 md:space-x-4 p-4">
                               
                                <div class="w-full md:w-1/2">
                                    <form class="flex items-center">
                                        <label for="simple-search" class="sr-only">Search</label>
                                        <div class="relative w-full">
                                            <div class="absolute inset-y-0 left-0 flex items-center pl-3 pointer-events-none">
                                                <svg aria-hidden="true" class="w-5 h-5 text-gray-500 dark:text-gray-400" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg">
                                                    <path fill-rule="evenodd" d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z" clip-rule="evenodd" />
                                                </svg>
                                            </div>
                                            <input type="text" id="simple-search" 
                                            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                            placeholder="Search" 
                                            bind:value={$searchTerm}
                                            />
                                        </div>
                                    </form>
                                </div>
                                <div class="w-full md:w-auto flex flex-col md:flex-row space-y-2 md:space-y-0 items-stretch md:items-center justify-end md:space-x-3 flex-shrink-0">
                                   
                                    <div class="flex items-center space-x-3 w-full md:w-auto">
                                        
                                        <button id="filterDropdownButton" on:click={toggleFilterDropdown} data-dropdown-toggle="filterDropdown" class="w-full md:w-auto flex items-center justify-center py-2 px-4 text-sm font-medium text-gray-900 focus:outline-none bg-white rounded-lg border border-gray-200 hover:bg-gray-100 hover:text-primary-700 focus:z-10 focus:ring-4 focus:ring-gray-200 dark:focus:ring-gray-700 dark:bg-gray-800 dark:text-gray-400 dark:border-gray-600 dark:hover:text-white dark:hover:bg-gray-700" type="button">
                                            <svg xmlns="http://www.w3.org/2000/svg" aria-hidden="true" class="h-4 w-4 mr-2 text-gray-400" viewbox="0 0 20 20" fill="currentColor">
                                                <path fill-rule="evenodd" d="M3 3a1 1 0 011-1h12a1 1 0 011 1v3a1 1 0 01-.293.707L12 11.414V15a1 1 0 01-.293.707l-2 2A1 1 0 018 17v-5.586L3.293 6.707A1 1 0 013 6V3z" clip-rule="evenodd" />
                                            </svg>
                                            Filter
                                            <svg class="-mr-1 ml-1.5 w-5 h-5" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
                                                <path clip-rule="evenodd" fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" />
                                            </svg>
                                        </button>
                                        {#if $isFilterDropdownOpen}
                                            <div id="filterDropdown" class="z-10  w-56 p-3 bg-white rounded-lg shadow dark:bg-gray-700">
                                                <ul class="space-y-2 text-sm" aria-labelledby="filterDropdownButton">
                                                    <li class="flex items-center">
                                                        <form class="flex items-center">
                                                            <div class="relative w-full">
                                                                <input type="number" id="simple-search" 
                                                                class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                                                placeholder="Year" 
                                                                bind:value={$searchYear}
                                                                />
                                                            </div>
                                                        </form>
                                                    </li>
                                                    <li class="flex items-center">
                                                        <form class="flex items-center">
                                                            <div class="relative w-full">
                                                                <input type="number" id="simple-search" 
                                                                class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                                                placeholder="Fmth" 
                                                                bind:value={$searchFmth}
                                                                />
                                                            </div>
                                                        </form>
                                                    </li>
                                                </ul>
                                            </div>
                                        {/if}
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                <table class="w-full text-sm text-left text-gray-500 dark:text-gray-400">
                    <thead class="text-xs text-gray-700 uppercase bg-gray-50 dark:bg-gray-700 dark:text-gray-400">
                    <tr>
                        <th scope="col" class="px-4 py-4">Sales Rep</th>
                        <th scope="col" class="px-4 py-3">Year</th>
                        <!-- <th scope="col" class="px-4 py-3">Financial Month</th> -->
                        <th scope="col" class="px-4 py-3">Current Year</th>
                        <th scope="col" class="px-4 py-3">GP Budget</th>
                        <th scope="col" class="px-4 py-3">Site</th>
                        <th scope="col" class="px-4 py-3">
                        <span class="sr-only">Actions</span>
                        </th>
                    </tr>
                    </thead>
                    <tbody>
                    {#each filterData($smssData, $searchTerm, $searchYear, $searchFmth) as item, index}
                        <tr class="border-b dark:border-gray-700">
                        <td class="px-4 py-3">
                            <input type="text" bind:value={item.salesrep} class="w-full p-2 border border-gray-300 rounded" />
                        </td>
                        <td class="px-4 py-3">
                            <input type="text" bind:value={item.year} class="w-full p-2 border border-gray-300 rounded" />
                        </td>
                        <!-- <td class="px-4 py-3">
                            <input type="text" bind:value={item.fmth} class="w-full p-2 border border-gray-300 rounded" />
                        </td> -->
                        <td class="px-4 py-3">
                            <input type="text" bind:value={item.cmth} class="w-full p-2 border border-gray-300 rounded" />
                        </td>
                        <td class="px-4 py-3">
                            <input type="number" bind:value={item.gpBudget} class="w-full p-2 border border-gray-300 rounded" />
                        </td>
                        <td class="px-4 py-3 max-w-[12rem] truncate">
                            <input type="text" bind:value={item.site} class="w-full p-2 border border-gray-300 rounded" />
                        </td>
                        
                        </tr>
                    {/each}
                    </tbody>
                    <button on:click={handleBulkEdit} class="px-4 py-2 m-2 bg-red-500 text-white rounded">Save Changes</button>
                    <button on:click={closePopup} class="mt-4 px-4 py-2 bg-gray-500 text-white rounded">Cancel</button>
                </table>

                
                </div>
            </div>
            {/if}

            {#if $openPopupView === 'Bulk Delete'}
            <div class="popup-container fixed inset-0 z-20 flex items-center justify-center bg-gray-900 bg-opacity-30">
                <div class="bg-white p-6 rounded shadow-lg dark:bg-gray-800 max-h-screen overflow-auto">
                    <div class="mx-auto max-w-screen-xl px-4 lg:px-12">
                        <!-- Start coding here -->
                        <div class="bg-orange-600 dark:bg-gray-800 relative shadow-md sm:rounded-lg overflow-hidden">
                            <div class="flex flex-col md:flex-row items-center justify-between space-y-3 md:space-y-0 md:space-x-4 p-4">
                               
                                <div class="w-full md:w-1/2">
                                    <form class="flex items-center">
                                        <label for="simple-search" class="sr-only">Search</label>
                                        <div class="relative w-full">
                                            <div class="absolute inset-y-0 left-0 flex items-center pl-3 pointer-events-none">
                                                <svg aria-hidden="true" class="w-5 h-5 text-gray-500 dark:text-gray-400" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg">
                                                    <path fill-rule="evenodd" d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z" clip-rule="evenodd" />
                                                </svg>
                                            </div>
                                            <input type="text" id="simple-search" 
                                            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                            placeholder="Search" 
                                            bind:value={$searchTerm}
                                            />
                                        </div>
                                    </form>
                                </div>
                                <div class="w-full md:w-auto flex flex-col md:flex-row space-y-2 md:space-y-0 items-stretch md:items-center justify-end md:space-x-3 flex-shrink-0">
                                   
                                    <div class="flex items-center space-x-3 w-full md:w-auto">
                                        
                                        <button id="filterDropdownButton" on:click={toggleFilterDropdown} data-dropdown-toggle="filterDropdown" class="w-full md:w-auto flex items-center justify-center py-2 px-4 text-sm font-medium text-gray-900 focus:outline-none bg-white rounded-lg border border-gray-200 hover:bg-gray-100 hover:text-primary-700 focus:z-10 focus:ring-4 focus:ring-gray-200 dark:focus:ring-gray-700 dark:bg-gray-800 dark:text-gray-400 dark:border-gray-600 dark:hover:text-white dark:hover:bg-gray-700" type="button">
                                            <svg xmlns="http://www.w3.org/2000/svg" aria-hidden="true" class="h-4 w-4 mr-2 text-gray-400" viewbox="0 0 20 20" fill="currentColor">
                                                <path fill-rule="evenodd" d="M3 3a1 1 0 011-1h12a1 1 0 011 1v3a1 1 0 01-.293.707L12 11.414V15a1 1 0 01-.293.707l-2 2A1 1 0 018 17v-5.586L3.293 6.707A1 1 0 013 6V3z" clip-rule="evenodd" />
                                            </svg>
                                            Filter
                                            <svg class="-mr-1 ml-1.5 w-5 h-5" fill="currentColor" viewbox="0 0 20 20" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
                                                <path clip-rule="evenodd" fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" />
                                            </svg>
                                        </button>
                                        {#if $isFilterDropdownOpen}
                                            <div id="filterDropdown" class="z-10  w-56 p-3 bg-white rounded-lg shadow dark:bg-gray-700">
                                                <ul class="space-y-2 text-sm" aria-labelledby="filterDropdownButton">
                                                    <li class="flex items-center">
                                                        <form class="flex items-center">
                                                            <div class="relative w-full">
                                                                <input type="number" id="simple-search" 
                                                                class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                                                placeholder="Year" 
                                                                bind:value={$searchYear}
                                                                />
                                                            </div>
                                                        </form>
                                                    </li>
                                                    <li class="flex items-center">
                                                        <form class="flex items-center">
                                                            <div class="relative w-full">
                                                                <input type="number" id="simple-search" 
                                                                class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-primary-500 focus:border-primary-500 block w-full pl-10 p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-primary-500 dark:focus:border-primary-500" 
                                                                placeholder="Fmth" 
                                                                bind:value={$searchFmth}
                                                                />
                                                            </div>
                                                        </form>
                                                    </li>
                                                </ul>
                                            </div>
                                        {/if}
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                <table class="w-full text-sm text-left text-gray-500 dark:text-gray-400">
                    <thead class="text-xs text-gray-700 uppercase bg-gray-50 dark:bg-gray-700 dark:text-gray-400">
                    <tr>
                        <th scope="col" class="px-4 py-4">
                        <input type="checkbox" on:change="{handleSelectAll}" />
                        </th>
                        <th scope="col" class="px-4 py-4">Sales Rep</th>
                        <th scope="col" class="px-4 py-3">Year</th>
                        <th scope="col" class="px-4 py-3">Financial Month</th>
                        <th scope="col" class="px-4 py-3">Current Year</th>
                        <th scope="col" class="px-4 py-3">GP Budget</th>
                        <th scope="col" class="px-4 py-3">Site</th>
                        <th scope="col" class="px-4 py-3">
                        <span class="sr-only">Actions</span>
                        </th>
                    </tr>
                    </thead>
                    <tbody>
                        {#each filterData($smssData, $searchTerm, $searchYear, $searchFmth) as item, index}
                        <tr class="border-b dark:border-gray-700">
                        <td class="px-4 py-3">
                            <input 
                            type="checkbox" 
                            checked={$selectedRows[index]} 
                            on:change={() => toggleRowSelection(index)} 
                            />
                        </td>
                        <td class="px-4 py-3">{item.salesrep}</td>
                        <td class="px-4 py-3">{item.year}</td>
                        <td class="px-4 py-3">{item.fmth}</td>
                        <td class="px-4 py-3">{item.cmth}</td>
                        <td class="px-4 py-3">{item.gpBudget}</td>
                        <td class="px-4 py-3">{item.site}</td>
                        </tr>
                    {/each}
                    </tbody>
                </table>
                <button on:click={HandleBulkDelete} class="px-4 py-2 bg-red-500 text-white rounded">Delete Selected</button>
                <button on:click={closePopup} class="mt-4 px-4 py-2 bg-gray-500 text-white rounded">Cancel</button>
                </div>
            </div>
            {/if}

            {#if $openPopupView === 'FileUpload'}
            <div class="fixed inset-0 z-20 flex items-center justify-center bg-gray-900 bg-opacity-50">
            <div class="bg-white py-6 px-8 rounded shadow-lg dark:bg-gray-800 max-h-screen overflow-auto">
                <h2 class="text-lg font-bold mb-4">Upload CSV File</h2>
        
                <!-- File Input -->
                <input type="file" accept=".csv" on:change={handleFileChange} class="block mb-4" />
        
                <!-- Preview Table -->
                {#if $csvPreviewData.length > 0}
                <h3 class="text-md font-bold mb-2">CSV Preview:</h3>
                <table class="w-full text-sm text-left border border-gray-200">
                    <thead>
                    <tr>
                        <th class="border-b py-2 px-4">Sales Rep</th>
                        <th class="border-b py-2 px-4">Year</th>
                        <th class="border-b py-2 px-4">Financial Month</th>
                        <th class="border-b py-2 px-4">Current Year</th>
                        <th class="border-b py-2 px-4">GP Budget</th>
                        <th class="border-b py-2 px-4">Site</th>
                    </tr>
                    </thead>
                    <tbody>
                    {#each $csvPreviewData as item}
                        <tr>
                        <td class="py-2 px-4">{item.salesrep}</td>
                        <td class="py-2 px-4">{item.year}</td>
                        <td class="py-2 px-4">{item.fmth}</td>
                        <td class="py-2 px-4">{item.cmth}</td>
                        <td class="py-2 px-4">{item.gpBudget}</td>
                        <td class="py-2 px-4">{item.site}</td>
                        </tr>
                    {/each}
                    </tbody>
                </table>
        
                <!-- Upload Button -->
                <button on:click={uploadCSVData} class="mt-4 text-white bg-green-500 p-2 rounded">
                    Upload CSV Data
                </button>
                {/if}
        
                <!-- Cancel Button -->
                <button on:click={closePopup} class="mt-4 text-white bg-red-500 p-2 rounded">
                Cancel
                </button>
            </div>
            </div>
        {/if}
            </section>
    </div>
    </div>
</div>