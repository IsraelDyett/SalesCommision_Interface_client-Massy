<script lang="ts">
    import img from '$lib/components/images/produce.jpg'; // Ensure path is correct
    import logo from '$lib/components/images/Massy_Distribution_Logo_No_BG2.png'; // Ensure path is correct

    let username: string = '';
    let password: string = '';
    let errorMessage: string = '';
    let successMessage: string = '';

    async function handleLogin(event: Event): Promise<void> {
        event.preventDefault(); // Prevent default form submission

        // Prepare the payload
        const payload = {
            username,
            password
        };

        try {
            const response = await fetch('https://localhost:7058/api/Auth/login', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(payload),
            });

            if (response.ok) {
                const result = await response.json();
                const token: string = result.token;

                // Store token in local storage
                localStorage.setItem('authToken', token);

                // Redirect to /smss
                window.location.href = 'http://localhost:5173/smss';
            } else {
                // Extract error details from response
                const errorData = await response.json();
                throw new Error(errorData.message || 'Login failed');
            }
        } catch (error) {
            // Display detailed error message
            errorMessage = `Login failed: ${error.message}`;
            console.error(error);
        }
    }
</script>

<div 
  class="h-screen flex justify-center items-center mx-auto px-4 lg:max-w-none lg:px-0 bg-cover bg-center relative" 
  style="background-image: url({img});"
>
  <!-- Overlay -->
  <div class="absolute inset-0 bg-gray-300 bg-opacity-30"></div>

  <!-- Login Form Content -->
  <div class="relative z-10 bg-white p-6 sm:p-8 rounded-lg shadow-lg w-full max-w-md">
    <img src={logo} alt="Massy Distribution Logo" class="mb-6 w-32 mx-auto" />

    <h2 class="text-2xl font-bold text-center text-gray-800 mb-6">Login to Your Account</h2>

    <!-- Form -->
    <form on:submit={handleLogin} class="space-y-4">
      <!-- Username Input -->
      <div>
        <label for="username" class="block text-sm font-medium text-gray-700">Username</label>
        <input 
          type="text" 
          id="username" 
          bind:value={username} 
          class="mt-1 block w-full p-2 border border-gray-300 rounded-md focus:ring-blue-500 focus:border-blue-500" 
          required 
        />
      </div>

      <!-- Password Input -->
      <div>
        <label for="password" class="block text-sm font-medium text-gray-700">Password</label>
        <input 
          type="password" 
          id="password" 
          bind:value={password} 
          class="mt-1 block w-full p-2 border border-gray-300 rounded-md focus:ring-blue-500 focus:border-blue-500" 
          required 
        />
      </div>

      <!-- Login Button -->
      <div>
        <button 
          type="submit" 
          class="w-full bg-blue-500 text-white font-bold py-2 px-4 rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500"
        >
          Login
        </button>
      </div>

      <!-- Error and Success Messages -->
      {#if errorMessage}
        <p class="text-red-500 text-center">{errorMessage}</p>
      {/if}
      {#if successMessage}
        <p class="text-green-500 text-center">{successMessage}</p>
      {/if}
    </form>
    <div class="flex">
        <!-- Forgot Password Link -->
        <p class=" mt-4  text-sm text-gray-600">
        Forgot your password? <a href="#" class="text-blue-500 hover:underline">Reset here</a>.
        </p>
        <p class="mt-4 ml-16 text-sm text-gray-600">
         <a href="#" class="text-blue-500 hover:underline">Register Here</a>
      </p>
    </div>
  </div>
</div>
