<script lang="ts">
	import { onMount } from 'svelte';
	import Quagga from '@ericblade/quagga2';

	let scannerElement: HTMLDivElement;
	let result = '';
	let isScanning = false;
	let bookInfo: any = null;
	let isLoading = false;

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

			Quagga.onDetected(async (data) => {
				const detectedCode = data.codeResult.code || '';
				if (isValidISBN(detectedCode)) {
					result = detectedCode;
					console.log('ISBN detected:', result);
					await fetchBookInfo(detectedCode);
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
		bookInfo = null;
	}

	async function fetchBookInfo(isbn: string) {
		isLoading = true;
		bookInfo = null;
		
		try {
			const response = await fetch(`https://www.googleapis.com/books/v1/volumes?q=isbn:${isbn}`);
			const data = await response.json();
			
			if (data.items && data.items.length > 0) {
				bookInfo = data.items[0].volumeInfo;
			} else {
				bookInfo = { error: '書籍情報が見つかりませんでした' };
			}
		} catch (error) {
			console.error('Google Books API error:', error);
			bookInfo = { error: 'API呼び出しエラー' };
		} finally {
			isLoading = false;
		}
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

		{#if isLoading}
			<div class="mt-4 p-4 bg-blue-100 border border-blue-400 rounded">
				<p class="text-center">書籍情報を取得中...</p>
			</div>
		{/if}

		{#if bookInfo}
			<div class="mt-4 p-4 bg-white border border-gray-300 rounded shadow">
				{#if bookInfo.error}
					<p class="text-red-600">{bookInfo.error}</p>
				{:else}
					<h2 class="text-xl font-bold mb-2">{bookInfo.title || '不明'}</h2>
					{#if bookInfo.authors}
						<p class="text-gray-700 mb-2">著者: {bookInfo.authors.join(', ')}</p>
					{/if}
					{#if bookInfo.publisher}
						<p class="text-gray-600 mb-2">出版社: {bookInfo.publisher}</p>
					{/if}
					{#if bookInfo.publishedDate}
						<p class="text-gray-600 mb-2">出版日: {bookInfo.publishedDate}</p>
					{/if}
					{#if bookInfo.description}
						<p class="text-gray-800 mb-2">概要: {bookInfo.description.substring(0, 200)}...</p>
					{/if}
					{#if bookInfo.imageLinks?.thumbnail}
						<img src={bookInfo.imageLinks.thumbnail} alt="書籍カバー" class="mt-2 max-w-32" />
					{/if}
				{/if}
			</div>
		{/if}
	</div>
</div>
