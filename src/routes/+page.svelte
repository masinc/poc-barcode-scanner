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
						width: 640,
						height: 480,
						facingMode: 'environment'
					}
				},
				decoder: {
					readers: [
						'ean_reader' // ISBN-13用
					]
				}
			});

			Quagga.start();
			isScanning = true;

			Quagga.onDetected((data) => {
				const detectedCode = data.codeResult.code || '';
				if (isValidISBN(detectedCode)) {
					result = detectedCode;
					console.log('ISBN detected:', result);
				} else {
					console.log('Non-ISBN barcode detected:', detectedCode);
				}
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

	function isValidISBN(code: string): boolean {
		if (!code) return false;
		
		// ISBN-13の場合（978または979で始まる13桁）
		if (code.length === 13 && (code.startsWith('978') || code.startsWith('979'))) {
			return true;
		}
		
		// ISBN-10の場合（10桁）- 現在はほとんど使われていないが念のため
		if (code.length === 10) {
			return true;
		}
		
		return false;
	}
</script>

<div class="container mx-auto p-4">
	<h1 class="text-3xl font-bold mb-6">ISBN バーコードスキャナー</h1>
	
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
		
		
		{#if result}
			<div class="mt-4 p-4 bg-green-100 border border-green-400 rounded">
				<h2 class="text-lg font-semibold mb-2">読み取り結果:</h2>
				<p class="text-xl font-mono">{result}</p>
			</div>
		{/if}
	</div>
</div>
