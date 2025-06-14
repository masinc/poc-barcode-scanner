<script lang="ts">
	import { onMount } from 'svelte';
	import Quagga from '@ericblade/quagga2';

	let scannerElement: HTMLDivElement;
	let result = '';
	let isScanning = false;

	onMount(() => {
		return () => {
			if (isScanning) {
				Quagga.stop();
			}
		};
	});

	async function startScanner() {
		if (isScanning) return;

		try {
			await Quagga.init({
				inputStream: {
					name: 'Live',
					type: 'LiveStream',
					target: scannerElement,
					constraints: {
						width: { ideal: 1920 },
						height: { ideal: 240 },
						facingMode: 'environment'
					}
				},
				decoder: {
					readers: [
						'code_128_reader',
						'ean_reader',
						'ean_8_reader',
						'code_39_reader',
						'code_39_vin_reader',
						'codabar_reader',
						'upc_reader',
						'upc_e_reader',
						'i2of5_reader'
					]
				}
			});

			Quagga.start();
			isScanning = true;

			Quagga.onDetected((data) => {
				result = data.codeResult.code || '';
				console.log('Barcode detected:', result);
			});
		} catch (error) {
			console.error('Error starting scanner:', error);
		}
	}

	function stopScanner() {
		if (isScanning) {
			Quagga.stop();
			isScanning = false;
		}
	}

	function clearResult() {
		result = '';
	}
</script>

<div class="container mx-auto p-4">
	<h1 class="text-3xl font-bold mb-6">バーコードスキャナー POC</h1>
	
	<div class="space-y-4">
		<div class="flex gap-2 justify-center">
			<button 
				on:click={startScanner} 
				disabled={isScanning}
				class="px-4 py-2 bg-blue-500 text-white rounded disabled:bg-gray-400"
			>
				{isScanning ? '読み取り中...' : 'スキャン開始'}
			</button>
			
			<button 
				on:click={stopScanner} 
				disabled={!isScanning}
				class="px-4 py-2 bg-red-500 text-white rounded disabled:bg-gray-400"
			>
				スキャン停止
			</button>
			
			<button 
				on:click={clearResult}
				class="px-4 py-2 bg-gray-500 text-white rounded"
			>
				結果をクリア
			</button>
		</div>
		
		<div 
			id="scannerElement"
			bind:this={scannerElement}
			class="w-full max-w-6xl mx-auto border-2 border-gray-300 rounded overflow-hidden relative"
			style="height: 200px;"
		></div>
		
		<style>
			:global(#scannerElement video) {
				width: 100% !important;
				height: 100% !important;
				object-fit: cover !important;
			}
			:global(#scannerElement canvas) {
				width: 100% !important;
				height: 100% !important;
			}
		</style>
		
		{#if result}
			<div class="mt-4 p-4 bg-green-100 border border-green-400 rounded">
				<h2 class="text-lg font-semibold mb-2">読み取り結果:</h2>
				<p class="text-xl font-mono">{result}</p>
			</div>
		{/if}
	</div>
</div>
